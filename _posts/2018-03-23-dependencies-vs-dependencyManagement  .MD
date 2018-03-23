---
title:  " maven dependencies 和 dependencyManagement 导致的工程依赖问题 "
date:  2018-03-22 11:16:01 -0600
categories: maven 
---


今天上午在调试工程的时候遇到一个问题：我的工程分了5个子模块，这些子模块都继承自一个父模块。父模块配置了相关的依赖包，我的想法是，父模块配置好所有的依赖，子模块只添加自己需要的依赖即可，但是
在实际发布的时候遇到了问题，子模块明明指定了自己的需要的依赖，但是发布后的war中，仍然是父模块中所有的依赖都被添加了进来。

 父 pom：

{% highlight java %}
        <dependencies>
            <dependency>
                <groupId>com.demo.qdp</groupId>
                <artifactId>activation</artifactId>
                <version>1.1</version>
            </dependency>
            ........
        <dependencies>
{% endhighlight %}



 子模块 pom
 
{% highlight java %}
        <dependencies>
            <dependency>
                <groupId>com.demo.qdp</groupId>
                <artifactId>activation</artifactId>
                <version>1.1</version>
            </dependency> 
        <dependencies>
{% endhighlight %}


我的期望是，子工程模块只有一个依赖即： com.demo.qdp.activation 但实际情况确是：父模块的所有依赖都加载了进来。

上网搜了一下，原来是  dependencies 标签和 dependencyManagement 的问题。最后从  <a href="https://stackoverflow.com/questions/38882221/how-to-inherit-dependency-from-a-parent-pom-to-a-child-pom">stackoverflow</a>  上找到了相关答案 


{% highlight java %}
 
You should declare dependencies you want to inherit under a <dependencies> section to achieve this. <dependencyManagement> is used for definitions that must be referenced later, whenever needed, within the <dependencies> of a particular child to become effective.

UPDATE: Be careful when declaring dependencies that every child pom will inherit. Very quickly you can end up having dependencies you don't really need just because they are declared in the parent. As mentioned by other commenters, <dependencyManagement> may be a better choice, although it isn't what you wanted originally.

{% endhighlight %}



{% highlight java %}
If you goal is to inherit particular dependencies to all child poms, without a need to decleare them in each of the children - then yes, you should declare them within <dependencies> and not <dependencyManagement>. On the other hand, a dependency declared within <dependencyManagement> can be used only when you need it in a child pom - you just write its group and artifact inside that child's <dependencies>
{% endhighlight %}

如果希望的效果是：子模块不需要任何声明就可以使用这些依赖包，那么就选择dependencies标签，否则就选择dependencyManagement
后者只会在子模块中手动添加<dependency></dependency>引用的时候，这个jar包才会被引用进来。



<a href="https://stackoverflow.com/questions/2619598/differences-between-dependencymanagement-and-dependencies-in-maven">这篇帖子</a>说的就更明显了：

{% highlight java %}
     In the parent POM, the main difference between the <dependencies> and <dependencyManagement> is this:
     Artifacts specified in the <dependencies> section will ALWAYS be included as a dependency of the child module(s).
     Artifacts specified in the <dependencyManagement> section, will only be included in the child module if they were also specified in the <dependencies> section of the child module itself. Why is it good you ask? because you specify the version and/or scope in the parent, and you can leave them out when specifying the dependencies in the child POM. This can help you use unified versions for dependencies for child modules, without specifying the version in each child module
{% endhighlight %}

父pom的 <dependencies> 标签中的依赖，始终会作为依赖传递到子模块中。而<dependencyManagement> 中的依赖，只有子模块手动声明了<dependencies>才会被添加进来。

因此，我在  <dependencies> 外面添加了一层<dependencyManagement>，再次刷新，子工程的依赖变了。