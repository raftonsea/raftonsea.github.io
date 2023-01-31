---
title:  " Future.get() 原理分析"
date:  2020-05-27 14:16:01 -0600
categories: juc
layout: post
---
 > 一个面试题 ： 在java中创建线程的方式有几种？很多答案是：两种：Runnable 接口和Callable 接口。两者有什么区别呢？ Runnable 无返回值，Callable 有返回值。
 
 上面的面试题，严格意义上说，我认为答案是不正确的，从java中创建线程其实只有一种方式： new Thread(Runnable) 。直接使用Callable是无法直接创建线程的，他需要与 线程池配合使用才可以。
 
 在线程池的实现中，当传入的参数为Callable时其实是将该Callable封装了一层，封装为一个 Runnable 然后交给 Thread 来执行的。
 在使用线程池时，一般通过他两个方法来提交任务： 
 
 ```
 
   java.util.concurrent.AbstractExecutorService#submit(java.util.concurrent.Callable<T>) 
   
   public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
    
    
    java.util.concurrent.ThreadPoolExecutor#execute:
    
     public void execute(Runnable command) {
            if (command == null)
                throw new NullPointerException();
            
            int c = ctl.get();
            if (workerCountOf(c) < corePoolSize) {
                if (addWorker(command, true))
                    return;
                c = ctl.get();
            }
            if (isRunning(c) && workQueue.offer(command)) {
                int recheck = ctl.get();
                if (! isRunning(recheck) && remove(command))
                    reject(command);
                else if (workerCountOf(recheck) == 0)
                    addWorker(null, false);
            }
            else if (!addWorker(command, false))
                reject(command);
        }
    
```
 

对于第一个方法，提交了一个 Callable 接口，返回了一个 Future 接口。而 Callable 可以带返回值，因此线程池执行完成以后，会将该值返回，
通过Future.get() 即可取到该值。一般这样用：

```

 Future future = threadPool.submit( ()->{
    ..... 
    return "ok";
 });
 
 future.get();

```


如果这个 Callable 没有执行完呢 ？ 那 get() 将会一直等待。 那线程池是如何做到这点的呢 ？

 ---
 在执行 submit() 的时候，其实是将我们提交的Callable 又包装了一层：
 
 ``` 
 
  public <T> Future<T> submit(Callable<T> task) {
         if (task == null) throw new NullPointerException();
         RunnableFuture<T> ftask = newTaskFor(task);    #1
         execute(ftask);
         return ftask;
     }
     
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable); #2
    }

    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable; #3
        this.state = NEW;   #4     // ensure visibility of callable
    }
    
```

在 #1 可见将提交的 callable 提交给了 #2 方法，#2 调用了 FutureTask 的 FutureTask<T>(callable) 构造函数生成了一个 RunnableFuture 对象

```
RunnableFuture 继承了 Runnable, Future 
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}


FutureTask 是 RunnableFuture 的实现类
public class FutureTask<V> implements RunnableFuture<V> 


```


可见 ***RunnableFuture 是一个接口，该接口继承自 Runnable, Future  因此他既是 Runnable 又是 Future 。
而 FutureTask 则是  RunnableFuture 的实现类***。
因此当向线程池提交一个 Callable 的时候，其实是提交了一个 Runnable+Future 对象。而该对象内部持有 Callable 对象。

当我们执行完成 submit之后，返回了一个 FutureTask ( ->RunnableFuture ->Runnable+Future)， 因此我们拿到的是一个 Future，这样我们就可以调用Future 的相关方法了。

于是我们调用 Future.get() ，其实是调用的 FutureTask 的 get() 方法：

``` 
  public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)   // state 是 FutureTask 内部维护的状态，用一个整型标识当前对象的状态。
            s = awaitDone(false, 0L);  // 若当前 state 小于 完成状态，则进入 awaitDone 进行等待
        return report(s);
    }
    
    
     private int awaitDone(boolean timed, long nanos)
            throws InterruptedException {
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            WaitNode q = null;   #1
            boolean queued = false;
            for (;;) {  // while true 一直执行
                if (Thread.interrupted()) {
                    removeWaiter(q);
                    throw new InterruptedException();
                }
    
                int s = state;
                if (s > COMPLETING) { // 再次判断当前状态是否完成，若当前完成了，则直接返回，这样可省去一次无用的等待。
                    if (q != null)
                        q.thread = null;
                    return s;
                }
                else if (s == COMPLETING) // cannot time out yet
                    Thread.yield();  // 若当前状态为 COMPLETING ，表示正在有其他线程执行完成操作，则此时 让出 cpu 
                else if (q == null)
                    q = new WaitNode(); // #2 第一轮循环会进入这个方法，因为 #1 
                else if (!queued)
                    queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                         q.next = waiters, q); // 当 #2 执行完以后，下一轮的循环将进入这个方法
                                                         //  q.next = waiters . 将当前新建的节点
                                                         // q 的下一个节点设置为 waiters （此时为 null） ,  q.next = waiters 的结果为 waiters
                                                         //  即这个 CAS 的做法是：将 waiters（此时为null） 设置为新建的 p ，而 p.next 为 null 
                                                         // 若此时多个线程执行到该方法，则保证了 多个线程构成了一个 链表，每个节点的 next 指向
                                                         // 前一个节点，最后达到的线程，排到了 队列的第一个；
                else if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) {
                        removeWaiter(q);
                        return state;
                    }
                    LockSupport.parkNanos(this, nanos);
                }
                else
                    LockSupport.park(this); // 最终会进入这个方法，执行 LockSupport.park(this) 将当前线程 挂起。
            }
        }
    
```

可见，在内部会先进行状态的判断 ( state ) ，若 state <= COMPLETING (正在完成)， 则会进入 awaitDone() 方法： 在该方法内将会调用LockSupport.park(this);  将线程挂起。
线程挂起后，什么时候会释放呢？
由于 FutureTask 同时是一个 Runnable 对象，因此他提交到线程池以后，最终会被线程池调用他的 run 方法。 因此， 他的run方法最终会被执行。


```

 public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();   // 执行 callable 的 call方法，获取其返回值。
                    ran = true; // 将 ran 设置为 true 
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result); // ran 为 true ，则会进入 set(result) 方法
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
    
    
    
       protected void set(V v) {
            if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) { // 通过 CAS  将 stateOffset 从 NEW 状态变更为 COMPLETING  
                outcome = v;
                UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // 若 CAS 设值成功，则将当前对象状态设置为 NORMAL ：正常完成
                finishCompletion(); 
            }
        }
        
        
         private void finishCompletion() {
                // assert state > COMPLETING;
                for (WaitNode q; (q = waiters) != null;) {
                    if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) { // 当该 callable 执行完成时，
                    // 进入该方法，此时若多个线程在等待 future 结束，则他们构成了一个 链表，waiters 指向了头节点。
                    //  CAS 将 waiters 设置为 null ，然后 while true 通知 排队的线程链表
                        for (;;) {
                            Thread t = q.thread;
                            if (t != null) {
                                q.thread = null;
                                LockSupport.unpark(t); // 每获取到一个 节点，则将该节点对应的 线程 unpark 。
                            }
                            WaitNode next = q.next; // 链表继续向下执行，确保所有等待的节点都会被唤醒。
                            if (next == null)
                                break;
                            q.next = null; // unlink to help gc
                            q = next;
                        }
                        break;
                    }
                }
        
                done();
        
                callable = null;        // to reduce footprint
            }
        


```


---

> 总结起来就是： 当尝试调用 Future.get() 时，若有多个线程在调用，则将这些线程构成一个链表。 同时 future 内部 使用 CAS 确保 waiters 对象始终指向头节点。
当 future 执行完成以后，先尝试将 waiters 头结点置空，并且 for 循环 唤醒排队的线程。

其他知识：
> 其中挂起线程用到了 LockSupport.park() , 唤醒线程用到了 LockSupport.unpark(); LockSupport 是 jdk5 之后加入的一个操作线程的
工具类。

 


   
   
   
   
   
   
   
   
   
   
   
  
 
