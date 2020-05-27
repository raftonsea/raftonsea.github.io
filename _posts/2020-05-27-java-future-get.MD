---
title:  " Future.get() 原理分析"
date:  2020-05-27 14:16:01 -0600
categories: juc
layout: post
---
 > 一个面试题 ： 在java中创建线程的方式有几种？很多答案是：两种：Runnable 接口和Callable 接口。两者有什么区别呢？ Runnable 无返回值，Callable 有返回值。
 
 上面的面试题，严格意义上说，答案是不正确的，从java中创建线程其实只有一种方式： new Thread(Runnable) 。直接使用Callable是无法直接创建线程的，他需要与 线程池配合使用才可以。
 
 在线程池的实现中，当传入的参数为Callable时其实是将该Callable封装了一层，封装为一个 Runnable 然后交给 Thread 来执行的。
 在使用线程池的时候一般有两种方式：一种是直接提交一个 runnable 接口，此时该线程无返回值；一种是提交一个 Callable 接口，此时该实现有返回值。
   
   
   
   
   
   
   
   
   
   
   
   
  
 
