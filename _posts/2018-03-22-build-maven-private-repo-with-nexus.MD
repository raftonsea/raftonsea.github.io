---
title:  "使用 nexus oss 搭建maven私服"
date:  2018-03-22 13:16:01 -0600
categories: maven private resp
---


这几天由于项目的关系，需要公司内部搭建一个maven私服，将jar都托管上去。同时，也需要将版本控制工具换成Git，搭一套Git服务。
顺道复习了一下maven私服的搭建，记录一下，也是答应了超哥写一篇博客给他看。(*^__^*)  


maven私服搭建的工具有很多，可以是nexus，artifactory 等，这里选择的是nexus。
nexus 也有两个版本 oss  和pro 版本， oss 是开源的， pro 是收费的，选择 nexus oss版本 。
nexus 也支持docker， 从docker hub 上直接拉取 oss 版本来安装也可以，这里选择从官网下载oss版本。
官网下载地址： https://www.sonatype.com/oss-thank-you-zip
我下载的版本是 nexus-2.14.5-02 ，下载后解压，解压后分两个文件夹： 

{% highlight java %}
    nexus-2.14.5-02
    sonatype-work 
{% endhighlight %}

nexus-2.14.5-02是主文件夹， sonatype-work 包含了nexus的数据， 比如日志、备份等信息。
启动nexus之前， 需要确保对应的jdk版本匹配。如下图

![]({{ "/assets/img/20180322/nexus-java.png" | absolute_url }})

2.14版本建议jdk8，jdk7无法保证。

进入 nexus-2.14.5-02\bin 执行 nexus.bat console : start  命令启动即可。

nexus 默认自带了jetty作为web服务器，因此启动无错误后，直接访问即可，默认端口为8081 
当然可以修改端口，在conf 目录下 nexus.properties 

{% highlight java %}  
    # Jetty section
    application-port=8082
    application-host=0.0.0.0
    nexus-webapp=${bundleBasedir}/nexus
    nexus-webapp-context-path=/nexus 
    # Nexus section
    nexus-work=${bundleBasedir}/../sonatype-work/nexus
    runtime=${bundleBasedir}/nexus/WEB-INF 
    # orientdb buffer size in megabytes
    storage.diskCache.bufferSize=4096 
{% endhighlight %}

我本地为8082 因此访问地址为： http://localhost:8082/nexus/ 登录页面如图：

![]({{ "/assets/img/20180322/nexus-login.png" | absolute_url }})

点击右上角登录，弹出登录窗口，默认用户名/密码为： admin/admin123

登录后，在右侧导航栏中点击：views/repositories 选择 Repositories 

![]({{ "/assets/img/20180322/nexus-repo.png" | absolute_url }})

显示系统内所有的仓库信息，仓库种类默认分为四类：


{% highlight java %}  
     group : 仓库组 
     hosted : 本地仓库
     proxy :  代理仓库
     virtual :  虚拟仓库
{% endhighlight %}

hosted ： 可以理解为本地仓库，即我们自己搭建的私服仓库，比如公司内部自己开发了一个工程
打成jar包后，即可提交到本地仓库中，这样本公司内部可以访问，而外界无法访问。
proxy ： 代理仓库， 例如，我们使用maven私服，私服的hosted仓库上传了我们公司内部的jar
但一部分共用的jar，比如spring的jar，没必要从私服来，而是继续从中央仓库来， 这样就使用proxy
类型的仓库，他的作用是代理另外一个库，当我们maven客户端连接到私服，无法获取到对应jar的时候，
会通过配置的代理库进行jar的查询。一般指向的是maven中央库，国内可以指向阿里云的库。
virtual ： 虚拟仓库，很少用。
group ： 组的概念是： 以当前开发为例，我们一部分jar需要内部使用，因此配了一个hosted库，
但一部分需要从别的地方而来，比如spring的jar， 就需要从proxy库来加载，那如何让客户端自动
选择呢？做法是，将这两个库，hosted 库和proxy库加入到一个组中，maven客户端访问这个group
对应的库，就可以实现自动从这两个库中查找对应jar了。


**1、**新建一个本地repo，命名为 demorepository****  

可参照官网：

https://help.sonatype.com/repomanager2/maven-and-other-build-tools/adding-a-new-repository

![]({{ "/assets/img/20180322/nexus-add-repo.png" | absolute_url }})

指定部署策略为：allow redeploy 默认为disable 

![]({{ "/assets/img/20180322/nexus-demo-repo.png" | absolute_url }})


**2、修改中央仓库指向阿里云**

![]({{ "/assets/img/20180322/nexus-central-aliyun.png" | absolute_url }})

阿里云地址：http://maven.aliyun.com/nexus/content/groups/public/ 

**同时需要修改中央仓库的 Download Remote Indexes 属性为true**

这一项的作用是，当私服的中央库没有对应jar的时候，自动请求配置的中央库（阿里云）加载jar 。 

**3、配置 Public Repositories** 

配置group 仓库，将我们创建的私有库加入到组中 

![]({{ "/assets/img/20180322/nexus-addtogroup.png" | absolute_url }})

将DemoRepository从右侧的 available repo 拖动到左侧即可 。

至此，maven 私服搭建完成。 

下面配置maven客户端访问私服。

1、配置maven的配置文件

 以idea为例，修改maven默认的setting文件位置： 
 
![]({{ "/assets/img/20180322/nexus-setting.png" | absolute_url }})


setting文件的内容为：

{% highlight xml %}  
     
    <?xml version="1.0" encoding="UTF-8"?>
     <settings> 
      <localRepository>E:/DevelopEnvironment/MavenLocalResp</localRepository> 
      <mirrors>
        <mirror>
          <!--This sends everything else to /public -->
          <!-- 指定一个新的maven中央库的信息，id为nexus，mirrorOf 指匹配所有请求， -->
          <!-- url是私服地址 -->
          <id>nexus</id>
          <mirrorOf>*</mirrorOf>
          <url>http://localhost:8082/nexus/content/groups/public/</url>
        </mirror>
      </mirrors>
      <profiles>
        <profile>
          <id>nexus</id>
          <!--Enable snapshots for the built in central repo to direct -->
          <!--all requests to nexus via the mirror -->
          <repositories>
            <repository>
              <id>central</id>
              <url>http://central</url>
              <releases><enabled>true</enabled></releases>
              <snapshots><enabled>true</enabled></snapshots>
            </repository>
          </repositories>
         <pluginRepositories>
            <pluginRepository>
              <id>central</id>
              <url>http://central</url>
              <releases><enabled>true</enabled></releases>
              <snapshots><enabled>true</enabled></snapshots>
            </pluginRepository>
          </pluginRepositories>
        </profile>
      </profiles>
      <activeProfiles>
        <!--make the profile active all the time -->
        <activeProfile>nexus</activeProfile>
      </activeProfiles>
       
      
      <!-- servers  节点配置的是访问maven私服的用户，用户部署jar包或者文件到私服的时候--> 
       
       <!-- 用户名密码为maven私服的用户名密码-->
       
      <servers>
          <server>
          <id>releases</id>
          <username>admin</username>
          <password>admin123</password>
        </server>
    	  <server>
    		  <id>snapshots</id>
    		  <username>admin</username>
    		  <password>admin123</password>
        </server>
      </servers>
        
    </settings>
    
{% endhighlight %}

上面配置文件的说明见官网：
https://help.sonatype.com/repomanager2/maven-and-other-build-tools/apache-maven

上面的文件直接拷贝使用即可，注意文件编码。

**2、新建maven项目，并修改对应的pom.xml文件**


{% highlight xml %}  
   
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.myrepo</groupId>
    <artifactId>demorepo</artifactId>
    <version>1.0</version>

    <repositories>
        <repository>
            <id>demorepo</id>
            <name>demorepo</name>
            <url>http://localhost:8082/nexus/content/groups/public/</url>
        </repository>
    </repositories>
 
    <distributionManagement>
        <repository>
            <id>releases</id>
            <name>releases</name>
            <url>http://localhost:8082/nexus/content/repositories/demorepository/</url>
            <uniqueVersion>true</uniqueVersion>
        </repository>
        <snapshotRepository>
            <id>snapshots</id>
            <name>snapshots</name>
            <url>http://localhost:8082/nexus/content/repositories/demorepository/</url>
        </snapshotRepository>
    </distributionManagement>

</project>   
{% endhighlight %}



repositories : 中央仓库地址，配置成私服地址即可，注意是group的仓库的地址
**distributionManagement ： 主要是用于将本地的程序打包部署到maven私服上用，这里的id name 需要跟setting.xml 
文件中的 servers 节点中的id对应起来。 对于普通的项目，不需要部署到maven私服上， 不配置这段也可以。**


至此，客户端配置完成，直接运行 mvn deploy 部署即可， 这样就可以将这个工程部署到私服上去。

一般的开发并不需要，只需要 mvn install 到本地仓库即可。 

 **3、运行 mvn deploy 测试** 

手动运行 mvn deploy 或者 借助工具运行，下图为idea

![]({{ "/assets/img/20180322/nexus-deploy.png" | absolute_url }})

双击运行，观察控制台输出，正常情况下为 success

成功后，到maven私服中，找到对应的 demorepository 仓库，点击后面的url进入查看
 
页，可以看到部署的这个jar包。

**4、上传第三方的jar**

有一种可能是，开发中加入了第三方的jar包，需要上传到maven私服。 

对于nexus oss 2 来说，提供了一个web页面进行上传 ，以demorepository为例，

![]({{ "/assets/img/20180322/nexus-uploadjar.png" | absolute_url }})

![]({{ "/assets/img/20180322/nexus-submitjar.png" | absolute_url }})


上传后，就像配置正常的依赖一样，配置对应的依赖引入即可。

当然，maven提供了一个命令行的方式上传jar包，而且在nexus3的版本中，也没有了

web页面上传

命令格式为：

mvn deploy:deploy-file -DgroupId=com.demo.qdp -DartifactId=bcprov  -Dversion=1.1  -Dpackaging=jar -Dfile=C:\Users\Administrator\Desktop\bcprov-jdk14-1.48.jar  -Durl=http://localhost:8082/nexus/content/repositories/demorepository/ -DrepositoryId=releases

