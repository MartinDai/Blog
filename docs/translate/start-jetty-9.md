每个单独发行的Jetty版本都有`bin/jetty.sh`这个脚本，可以在各种Unix（包括OS X）系统中用来管理jetty的启动。

这个脚本适用于在Unix中把Jetty设置为服务。

##**快速启动Jetty服务**

以下是运行Jetty服务的最短步骤：

```shell
[/opt/jetty]# tar -zxf /home/user/downloads/jetty-distribution-9.3.1-SNAPSHOT.tar.gz 
[/opt/jetty]# cd jetty-distribution-9.3.1-SNAPSHOT/
[/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT]# ls
bin        lib                         modules      resources  start.jar
demo-base  license-eplv10-aslv20.html  notice.html  start.d    VERSION.txt
etc        logs                        README.TXT   start.ini  webapps

[/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT]# cp bin/jetty.sh /etc/init.d/jetty
[/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT]# echo JETTY_HOME=`pwd` > /etc/default/jetty
[/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT]# cat /etc/default/jetty
JETTY_HOME=/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT

[/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT]# service jetty start
Starting Jetty: OK Wed Nov 20 10:26:53 MST 2013
```

从这个简单的例子中，我们可以看到在`/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT`文件夹下的Jetty作为Unix服务成功运行了。

这看起来都很好，但是你使用的是root角色运行的默认配置的Jetty服务。

##**Jetty服务的实用性设置**

有多种方式可以实现这一点，这主要取决于你的Unix环境(或者是公司政策)。

这里假设我们使用的是Linux系统（示例版本为Ubuntu 12.04.3 LTS）。

###**准备系统**

```shell
# mkdir -p /opt/jetty
# mkdir -p /opt/web/mybase
# mkdir -p /opt/jetty/temp
```

以下是这几个文件夹的作用：

- `/opt/jetty` 用于存放解压后的Jetty发布文件。
- `/opt/web/mybase` 自定义用于存放web应用，包括所有能让它们运行在服务器上的必要配置。
- `/opt/jetty/temp` 这是分配给java服务的临时文件夹（可以把它看作是`java.io.tmpdir`这个系统属性）。

这里是故意保持和标准的临时文件夹命名`/tmp`不一样的，因为它还兼作servlet的规范工作目录。（这是我们的经验，在长时间运行的Jetty服务器上，标准临时目录通常由各种清理脚本管理）

###**确认你安装了Java 7**

Jetty`${project.version}`运行需要Java 7（或者以上），确保你安装了合适的Java版本。

```shell
# apt-get install openjdk-7-jdk
```

或者[下载Java7](http://www.oracle.com/technetwork/java/javase/downloads/index.html)

```shell
# java -version
java version "1.6.0_27"
OpenJDK Runtime Environment (IcedTea6 1.12.6) (6b27-1.12.6-1ubuntu0.12.04.2)
OpenJDK 64-Bit Server VM (build 20.0-b12, mixed mode)

# update-alternatives --list java
/usr/lib/jvm/java-6-openjdk-amd64/jre/bin/java
/usr/lib/jvm/java-7-openjdk-amd64/jre/bin/java

# update-alternatives --config java
There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-6-openjdk-amd64/jre/bin/java   1061      auto mode
  1            /usr/lib/jvm/java-6-openjdk-amd64/jre/bin/java   1061      manual mode
  2            /usr/lib/jvm/java-7-openjdk-amd64/jre/bin/java   1051      manual mode

Press enter to keep the current choice[*], or type selection number: 2
update-alternatives: using /usr/lib/jvm/java-7-openjdk-amd64/jre/bin/java to provide /usr/bin/java (java) in manual mode.

# java -version
java version "1.7.0_25"
OpenJDK Runtime Environment (IcedTea 2.3.10) (7u25-2.3.10-1ubuntu0.12.04.2)
OpenJDK 64-Bit Server VM (build 23.7-b01, mixed mode)
```

###**创建用于运行Jetty的用户**

推荐创建一个指定的用户用于运行Jetty，该用户应该拥有运行Jetty的最小设置权限。

```shell
# useradd --user-group --shell /bin/false --home-dir /opt/jetty/temp jetty
```

这里创建了一个名为`jetty`的用户，并属于名为`jetty`的组，不能访问shell（`/bin/false`），主目录在`/opt/jetty/temp`。

###**下载并解压你的发布版本**

可以从[Official Eclipse Download Site](http://www.eclipse.org/jetty/documentation/current/quick-start-getting-started.html#jetty-downloading)获取一个发布副本。

解压到指定文件夹。

```shell
[/opt/jetty]# tar -zxf /home/user/Downloads/jetty-distribution-9.3.1-SNAPSHOT.tar.gz 
[/opt/jetty]# ls -F
jetty-distribution-9.3.1-SNAPSHOT/
[/opt/jetty]# mkdir /opt/jetty/temp
```

可能解压出来的Jetty发布文件夹名称看起来看奇怪或不合理的，但是从Jetty9.1开始，`${jetty.home}`和`${jetty.base}`分开可以隔离你的web引用特殊的设置，从而更简单地升级。

已经创建的`/opt/jetty/temp`作为持久的目录当作Jetty的缓存和工作目录。许多Unix系统会定期清理`/tmp`文件夹，这种行为在不是Servlet容器所期望的，而且会导致问题的发生，`/opt/jetty/temp`这个持久的目录就是应对这种行为的解决方案。

###**配置web应用**

`/opt/web/mybase`这个文件夹就是`${jetty.base}`，所以让我们配置它来保存你的应用及其配置。

***小贴士**：在过去的Jetty版本中，你得在Jetty发布目录下操作修改或添加，虽然这依然支持，但是我们鼓励你设置一个合适的`${jetty.base}`目录，因为这将有利于你在将来更容易升级Jetty的版本。

```shell
# cd /opt/web/mybase/
[/opt/web/mybase]# ls
[/opt/web/mybase]# java -jar /opt/jetty/jetty-distribution-9.3.1-SNAPSHOT/start.jar \
   --add-to-start=deploy,http,logging
WARNING: deploy          initialised in ${jetty.base}/start.ini (appended)
WARNING: deploy          enabled in     ${jetty.base}/start.ini
WARNING: server          initialised in ${jetty.base}/start.ini (appended)
WARNING: server          enabled in     ${jetty.base}/start.ini
WARNING: http            initialised in ${jetty.base}/start.ini (appended)
WARNING: http            enabled in     ${jetty.base}/start.ini
WARNING: server          enabled in     ${jetty.base}/start.ini
WARNING: logging         initialised in ${jetty.base}/start.ini (appended)
WARNING: logging         enabled in     ${jetty.base}/start.ini
[/opt/web/mybase]# ls -F
start.ini  webapps/
```

此时，你已经为你的`/opt/web/mybase`启用以下模块：

**deploy**	：这个模块将执行`/opt/web/mybase/webapps`目录下的web应用程序部署（war文件、解压的目录或可部署上下文的Jetty IoC XML文件）。

**http**：这个设置一个单独的连接器,监听基本的HTTP请求。可以通过已经创建的`start.ini`文件配置连接器。

**logging**：当Jetty作为服务运行的时候，开启日志是非常重要的。这个模块将开启捕获基本的标准输出和标准错误日志记录功能，并保存到到`/opt/web/mybase/logs/`文件夹。

参考[Using start.jar ](http://www.eclipse.org/jetty/documentation/current/start-jar.html)获取更多关于如何配置`${jetty.base}`文件夹的信息。

复制一个war文件到目录中。

```shell
# cp /home/user/projects/mywebsite.war /opt/web/mybase/webapps/
```

大多数服务设备希望Jetty是运行在80端口的，现在你有机会可以把默认的8080改为80。
编辑`/opt/web/mybase/start.ini`并修改`jetty.http.port`的值。

```shell
# grep jetty.http.port /opt/web/mybase/start.ini
jetty.port=80
```

###**修改权限**

修改Jetty发布的权限，设置你创建的用户可以访问你的web应用文件夹。

```shell
# chown --recursive jetty /opt/jetty
# chown --recursive jetty /opt/web/mybase
```

###**配置服务层**

接下来我们需要新建一个Jetty服务让它在Unix系统中可以和标准的服务一样管理调用。

```shell
# cp /opt/jetty/jetty-distribution-9.3.1-SNAPSHOT/bin/jetty.sh /etc/init.d/jetty
# echo "JETTY_HOME=/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT" > /etc/default/jetty
# echo "JETTY_BASE=/opt/web/mybase" >> /etc/default/jetty
# echo "TMPDIR=/opt/jetty/temp" >> /etc/default/jetty
```

测试配置

```shell
# service jetty status
Checking arguments to Jetty: 
START_INI      =  /opt/web/mybase/start.ini
JETTY_HOME     =  /opt/jetty/jetty-distribution-9.3.1-SNAPSHOT
JETTY_BASE     =  /opt/web/mybase
JETTY_CONF     =  /opt/jetty/jetty-distribution-9.3.1-SNAPSHOT/etc/jetty.conf
JETTY_PID      =  /var/run/jetty.pid
JETTY_START    =  /opt/jetty/jetty-distribution-9.3.1-SNAPSHOT/start.jar
JETTY_LOGS     =  /opt/web/mybase/logs
CLASSPATH      =  
JAVA           =  /usr/bin/java
JAVA_OPTIONS   =  -Djetty.state=/opt/web/mybase/jetty.state 
       -Djetty.logs=/opt/web/mybase/logs
       -Djetty.home=/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT 
       -Djetty.base=/opt/web/mybase 
       -Djava.io.tmpdir=/opt/jetty/temp
JETTY_ARGS     =  jetty-logging.xml jetty-started.xml
RUN_CMD        =  /usr/bin/java 
       -Djetty.state=/opt/web/mybase/jetty.state 
       -Djetty.logs=/opt/web/mybase/logs 
       -Djetty.home=/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT 
       -Djetty.base=/opt/web/mybase 
       -Djava.io.tmpdir=/opt/jetty/temp
       -jar /opt/jetty/jetty-distribution-9.3.1-SNAPSHOT/start.jar 
       jetty-logging.xml 
       jetty-started.xml
```

###**启动你的服务**

现在你有一个在`/opt/web/mybase`的`${jetty.base}`和一个在`/opt/jetty/jetty-distribution-9.3.1-SNAPSHOT`的Jetty发布版本，需要启动服务才能让它长时间的成为服务级文件。

开始，启动吧。

```shell
# service jetty start
Starting Jetty: OK Wed Nov 20 12:35:28 MST 2013

# service jetty check
..(snip)..
Jetty running pid=2958

[/opt/web/mybase]# ps u 2958
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
jetty     2958  5.3  0.1 11179176 53984 ?      Sl   12:46   0:00 /usr/bin/java -Djetty...
```

你现在应该让它在服务器上运行，试一试。

> 原文链接：[Startup a Unix Service using jetty.sh](http://www.eclipse.org/jetty/documentation/current/startup-unix-service.html)