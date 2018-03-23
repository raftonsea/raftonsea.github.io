---
title:  "使用docker + nexus:oss 搭建私有仓库"
date:  2018-03-23 13:16:01 -0600
categories: docker
---

使用nexus的oss版本搭建maven私服的确是很方便，但是nexus也是有环境要求的，比如不同的oss版本，对应不同的jdk版本。
好在nexus是支持docker的，使用docker来搭建nexus oss环境，就可以不用考虑jdk的版本了，因为官方已经为我们封装好了。

docker也是有两个版本：社区版和企业版，企业版是收费的，社区版是免费的，选择社区版，下载地址：<a href="https://www.docker.com/community-edition#/download"> 社区版 </a>。

下载好以后安装，安装成功后，会在任务栏有个docker的图标。右单击，选择setting，弹出设置页面

![]({{ "/assets/img/20180323/docer-install.png" | absolute_url }})

![]({{ "/assets/img/20180323/docker-setting.png" | absolute_url }})


一切顺利的话，安装成功后，docker会自动注册环境变量。

打开cmd 

输入命令查看版本：

{% highlight java %}
    docker -v 
{% endhighlight %}


{% highlight java %}
    Docker version 17.12.0-ce, build c97c6d6
{% endhighlight %}

 docker有两个重要概念：image 和 container 
 
 image 镜像文件可以理解为一个包含软件运行环境的安装包，制作好以后放到DockerHub，大家从DockerHub拉取这个镜像到本地。
 
 container 容器：可以理解为一个独立的运行环境，一个container从一个image启动而来，一个image可以启动多个container 。
 
 举例说明：我们用spring boot 做了一个网站，运行这个网站，需要一个Tomcat，Tomcat又需要安装jre环境 ，不同的Tomcat又需要不同的jre环境，如果我们自己动手来部署，那么我们需要准备Tomcat，jdk，将项目部署上去，
 我们测试环境在windows 32 位系统上，而生产环境换成了centos6.5 我们又需要重新下载jdk，重新安装部署。这中间很多时间都是重复枯燥的。docker的出现解决了这个痛点，
 docker直接打包了一个image(jdk+tomcat) ，我们拿到以后，直接从这个image启动一个container(可以理解为在Windows环境部署一套) ，也可以再启动一个container(可以理解为从centos部署一套)，有了docker，我们只需要执行一条命令就行了
 而不是像以前一样，一个环境折腾一圈。
 
 下面开始从docker部署nexus
 
 **1、下载nexus的image**
 
  查看当前的image文件
   {% highlight java %}
       docker images
   {% endhighlight %}
 
 
 执行命令，拉取nexus的image：

 {% highlight java %}
     docker pull sonatype/nexus:oss 
 {% endhighlight %}
 
 docker pull  是执行拉取，即从dockerhub上拉取对应名称的image文件
 
 sonatype/nexus:oss    image 名称， oss是其对应的tag 
 
 国内直接访问docker源速度略慢，因此可以使用代理或者国内的源。
 
 下载完成后，查看image
   {% highlight java %}
       docker images
   {% endhighlight %} 
  
 
 ![]({{ "/assets/img/20180323/docker-pull.png" | absolute_url }})
 
 ![]({{ "/assets/img/20180323/docker-images.png" | absolute_url }})
 
  image文件下载成功。
  
  
  **2、启动 container ： nexus** 
 
 执行命令：
   {% highlight java %}
       docker run -d -p 8082:8081 --name nexus sonatype/nexus:oss
   {% endhighlight %} 
 
 docker run  启动docker容器 
 
 -d 以后台进程运行
 
 -p 端口映射， 将本地的8082 映射到容器的8081端口，因此启动后使用 localhost:8082/nexus 访问
 
 --name 容器的名称 
 
 sonatype/nexus:oss ： image镜像的名称。
 
 ![]({{ "/assets/img/20180323/docker-run.png" | absolute_url }})
 
 至此，容器启动成功。可以在本地以 http://localhost:8082/nexus/#welcome 进行访问，默认用户名/密码为： admin/admin123
 
 具体部署maven私服，可以参照<a href="http://blog.cfyh.tech/maven/private/resp/2018/03/22/build-maven-private-repo-with-nexus.html">这里</a>
 
 **3、查看容器**

  运行命令
   {% highlight java %}
       docker container ls 
   {% endhighlight %} 
   会列出当前所有正在运行的container 
   
   {% highlight java %}
          docker container ls  -a
   {% endhighlight %}
   列出所有容器：包含运行的和未运行的。
 
  ![]({{ "/assets/img/20180323/docker-list.png" | absolute_url }})
  
  **4、停止容器**
  
  执行命令：
  
   {% highlight java %}
       docker stop nexus 
   {% endhighlight %} 
   
   nexus 对应容器的名称
   
   **5、再次启动容器**
   
   注意，若一个启动一个已经启动过的容器，不能使用上面的  docker run 命令，否则会出现问题，docker run 用来启动一个全新的容器，若--name参数相同，则会报错，无法启动。
   
   应使用如下命令：
      {% highlight java %}
          docker start nexus 
      {% endhighlight %} 
   nexus 为容器名称
   
   **6、查看启动日志**
   
   {% highlight java %}
             docker logs -f nexus
   {% endhighlight %} 
      
   nexus 为容器名称
   
   **7、设置启动参数**  
   
   nexus 提供了几个运行时参数
   
   {% highlight java %}
     CONTEXT_PATH,   设置访问路径，默认为 '/nexus'
     MAX_HEAP, jvm最大内存，默认 768m.
     MIN_HEAP, jvm最大内存，默认 256m.
     JAVA_OPTS. 设置jvm参数 ，默认 -server -XX:MaxPermSize=192m -Djava.net.preferIPv4Stack=true.
     LAUNCHER_CONF.  nexus启动配置文件，默认: ./conf/jetty.xml ./conf/jetty-requestlog.xml
   {% endhighlight %} 
      
   设置jvm内存：
  
   {% highlight java %}
       docker run -d -p 8082:8081 --name nexusjvm -e MAX_HEAP=768m sonatype/nexus:oss
   {% endhighlight %} 
   
   这样，相当于重新启动了一个容器，因此，之前操作的nexus私服的数据将会丢失，这显然不合适。因此，需要将数据持久化。
   
   **8、持久化数据**
   
   nexus 支持两种数据持久化的方式：
   
   {% highlight java %}
    1、使用一个"data volume"类型的container，即启动一个单独的容器，用来存储数据，然后把我们需要持久化的容器，通过volumes-from 指向该容器，达到数据持久共享的目的。这也是推荐做法。
    2、挂载一个系统的文件夹作为数据存储路径，这种方式，不灵活，而且依赖于系统，有可能出现权限问题。
   {% endhighlight %} 
   
   使用第一种：
   {% highlight java %}
         docker run -d --name nexus-data sonatype/nexus:oss echo "data-only container for Nexus"
         docker run -d -p 8082:8081 --name nexus-persist --volumes-from nexus-data sonatype/nexus:oss
   {% endhighlight %} 
   
   先启动一个名为 nexus-data 的容器，然后再启动了一个名为nexus-persist的容器，通过volumes-from 关联到该容器！
   
   
   全文完。
   
   
   
   参考链接：
   
   <a href="https://hub.docker.com/r/sonatype/nexus/">nexus DockerHub</a>
   
   <a href="https://docs.docker.com/engine/reference/commandline/docker/">Dokcer official commandline</a>
   
   
   
   
   
   
   
   
   
   
   
   
   
   
   
  
 
