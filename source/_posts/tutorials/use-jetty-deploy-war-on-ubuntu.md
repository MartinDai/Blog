---
title: 在Ubuntu上使用Jetty部署War包
date: 2019-10-13 20:48:00
categories: 
- 教程
---

## 前提

有一台装有Ubuntu系统的服务器和一个可以部署的War包

## 安装Java

**创建文件夹**
```shell
sudo mkdir /usr/java
cd /usr/java
```
进入[https://www.oracle.com/technetwork/java/javase/downloads/index.html](https://www.oracle.com/technetwork/java/javase/downloads/index.html)找到需要安装的JDK版本下载地址

**下载JDK**
```shell
sudo wget --no-check-certificate -c --header "Cookie: oraclelicense=accept-securebackup-cookie" https://download.oracle.com/otn-pub/java/jdk/13+33/5b8a42f3905b406298b72d750b6919f6/jdk-13_linux-x64_bin.tar.gz
```
有些版本不支持这种方式下载，所以只能手动下载后再上传到服务器

**解压JDK**
```shell
sudo tar -xvzf jdk-13_linux-x64_bin.tar.gz
```

**安装Java软链**
```shell
sudo update-alternatives --install "/usr/bin/java" "java" "/usr/java/jdk-13/bin/java" 0
```
```shell
sudo update-alternatives --install "/usr/bin/javac" "javac" "/usr/java/jdk-13/bin/javac" 0
```
```shell
sudo update-alternatives --set java /usr/java/jdk-13/bin/java
```
```shell
sudo update-alternatives --set javac /usr/java/jdk-13/bin/javac
```
其中`jdk-13`是上一步解压后的文件夹名，根据实际版本做替换

<!--more-->

**验证Java软链**
```shell
update-alternatives --list java
```
```shell
update-alternatives --list javac
```
应该可以输出配置的路径

**修改环境变量**
```shell
sudo nano /etc/environment
```

在PATH变量后追加
```text
:/usr/java/jdk-13/bin
```

新增变量
```text
JAVA_HOME="/usr/java/jdk-13"
```

Control+X保存退出，编辑后文件类似于如下：
```text
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/java/jdk-13/bin"
JAVA_HOME="/usr/java/jdk-13"
```

**验证Java版本**

重新登录终端，执行
```shell
java -version
```

## 安装Jetty

进入[https://www.eclipse.org/jetty/download.html](https://www.eclipse.org/jetty/download.html) 复制下载地址

**创建文件夹**
```shell
sudo mkdir /usr/jetty
cd /usr/jetty
```

**下载Jetty**
```shell
sudo wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-distribution/9.4.21.v20190926/jetty-distribution-9.4.21.v20190926.tar.gz
```

**解压Jetty**
```shell
sudo tar -xvzf jetty-distribution-9.4.21.v20190926.tar.gz
```

## 上传War包

如果有权限可以直接使用rz命令上传
**进入wabapps文件夹**
```shell
cd /usr/jetty/jetty-distribution-9.4.21.v20190926/webapps/
rz
```

当然也可以使用如下指令将本地文件复制到服务器临时目录
```shell
scp ~/project.war username@hostname:/tmp  
```
其中`username`为用户名，`hostname`为服务器外网地址

然后复制到webapps目录
```shell
sudo mv /tmp/project.war /usr/jetty/jetty-distribution-9.4.21.v20190926/webapps/
```

## 启动Jetty

**编辑start.ini**
```shell
sudo vi /usr/jetty/jetty-distribution-9.4.21.v20190926/start.ini
```

找到jetty.http.host和jetty.http.port，去掉前面的#号,如有需要可修改绑定端口
```text
## Connector host/address to bind to
# jetty.http.host=0.0.0.0

## Connector port to listen on
# jetty.http.port=8080
```

**启动Jetty**
```shell
sudo /usr/jetty/jetty-distribution-9.4.21.v20190926/bin/jetty.sh start nohup
```

**停止Jetty**
```shell
sudo /usr/jetty/jetty-distribution-9.4.21.v20190926/bin/jetty.sh stop
```

如果只有一个应用想要把根路径绑定到该应用，则可在webapps目录下添加应用同名的xml，如当前有project.war，则可新增project.xml文件,内容为
```xml
<?xml version="1.0"  encoding="ISO-8859-1"?>
<!DOCTYPE Configure PUBLIC "-//Jetty//Configure//EN" "http://www.eclipse.org/jetty/configure.dtd">
<Configure class="org.eclipse.jetty.webapp.WebAppContext">
    <Set name="contextPath">/</Set>
    <Set name="war"><SystemProperty name="jetty.home" default="."/>/webapps/project.war</Set>
</Configure>
```

完成

参考链接
[https://www.javahelps.com/2019/04/install-latest-oracle-jdk-on-linux.html](https://www.javahelps.com/2019/04/install-latest-oracle-jdk-on-linux.html)
[https://www.cnblogs.com/freeweb/p/5942972.html](https://www.cnblogs.com/freeweb/p/5942972.html)
