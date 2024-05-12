---
title: Docker构建多平台镜像
date: 2024-04-05 14:27:00
categories: 
- Docker
---

## 多平台镜像使用场景

我们知道Docker镜像是支持多平台（不同的操作系统/架构）的，比如linux/amd64，linux/arm64，linux/riscv64等，当我们需要在不同平台使用容器运行我们的镜像的时候，通常可能会考虑分别编译各个平台的镜像文件，然后打上不同的tag用来区分平台，使用的时候也同样需要根据实际运行的平台在配置文件中选择不同的tag，这样就导致配置文件无法良好的共用，使用起来相当的不方便。

多平台镜像就是解决此问题的一个方案，那么什么是多平台镜像呢？我们以MySQL的官方镜像为例（Docker Hub上的大多数Docker官方镜像都提供了多平台镜像）

![1.png](https://imgs.doodl6.com/docker/build-multi-platform-image/1.png)

可以看到，该镜像支持在linux/amd64和linux/arm64平台下运行，通过执行相同的命令`docker pull mysql:8.0`获取镜像，docker会自动根据manifest的描述找到并下载适合当前系统架构的镜像文件。

如此，便可以在多个平台之间共用同一份配置文件，而无需多余的处理，是不是方便很多呢？

---

## 如何构建多平台镜像

接下来介绍两种构建多平台镜像的方法

### 方法一（推荐）

此方法需要开启docker的containerd-snapshotter特性

#### 开启containerd-snapshotter

修改/etc/docker/daemon.json文件，添加
```text
"features": {
  "containerd-snapshotter": true
}
```
![2.png](https://imgs.doodl6.com/docker/build-multi-platform-image/2.png)

重启docker服务

**注意：开启该配置以后，会导致原来下载的镜像文件被隐藏，开启和关闭该配置情况下的镜像文件是隔离存储的（OrbStack下验证）**

**OrbStack在设置里面的Docker功能页添加**

**Docker Desktop也可以参考[https://docs.docker.com/desktop/containerd/](https://docs.docker.com/desktop/containerd/)**

<!--more-->

#### 查看当前构建工具支持的平台

```shell
docker buildx inspect --bootstrap
```
![3.png](https://imgs.doodl6.com/docker/build-multi-platform-image/3.png)

其中的Platforms就是当前支持编译构建的平台

#### 编译多平台镜像

编译指定的多平台镜像加载到本地，平台之间需使用逗号隔开

```shell
docker buildx build --platform linux/amd64,linux/arm64 -t martindai/wechat-robot:1.0 --load .
```
**如果不想把镜像加载到本地，想要直接推送到仓库，修改`--load`为`--push`即可**

查看镜像列表
```shell
docker images
```
![4.png](https://imgs.doodl6.com/docker/build-multi-platform-image/4.png)

这里你会发现，有两个镜像文件，其中第一个就是我们编译的多平台镜像，文件大小比较大，而第二个则是适用于当前平台的镜像文件。

#### 推送到仓库（确保已经登录）
```shell
docker push martindai/wechat-robot:1.0
```
![5.png](https://imgs.doodl6.com/docker/build-multi-platform-image/5.png)

可以看到，我们的多平台镜像就完成了。

### 方法二

我们可以使用命令查看一下方法一编译出来的多平台镜像的manifest信息

```shell
docker manifest inspect martindai/wechat-robot:1.0
```
![6.png](https://imgs.doodl6.com/docker/build-multi-platform-image/6.png)

可以看到，本质上这个多平台镜像只是一个索引，并不包含实际的文件，实际背后就是适用于两个平台的独立的镜像文件，docker在使用该镜像的时候会解析该索引文件，然后选择拉取合适的实际的镜像文件，相当于对于使用者屏蔽了平台这一层的信息。

方法一其实就是编译完多个平台的镜像以后，自动创建了一个索引，然后把各个平台的镜像做了一个关联。

那么方法二就是要手动创建这个索引，镜像的关联完全由我们自己控制。

下面开始操作

#### **注意：此方法不能开启containerd-snapshotter特性**

#### 交叉编译多平台镜像

通过tag区分平台

```shell
docker buildx build --platform linux/amd64 -t martindai/wechat-robot:1.1-amd64 .
docker buildx build --platform linux/arm64 -t martindai/wechat-robot:1.1-arm64 .
```

或通过镜像名区分平台

```shell
docker buildx build --platform linux/amd64 -t martindai/wechat-robot-amd64:1.1 .
docker buildx build --platform linux/arm64 -t martindai/wechat-robot-arm64:1.1 .
```

**PS：下面操作基于tag区分平台**

#### 推送到仓库

手动关联的镜像必须要在仓库里面有才行，所以需要先把编译好的单平台镜像推送到仓库

```shell
docker push martindai/wechat-robot:1.1-amd64
docker push martindai/wechat-robot:1.1-arm64
```
![7.png](https://imgs.doodl6.com/docker/build-multi-platform-image/7.png)

#### 创建manifest关联镜像

```shell
docker manifest create martindai/wechat-robot:1.1 martindai/wechat-robot:1.1-amd64 martindai/wechat-robot:1.1-arm64 --amend
```

#### 确认manifest的信息

```shell
docker manifest inspect martindai/wechat-robot:1.1
```
![8.png](https://imgs.doodl6.com/docker/build-multi-platform-image/8.png)

可以看到该manifest跟方法一的manifest是类似的（不是完全一样，类型其实是不一样的，只是效果类似），也是关联了两个平台镜像。

如果需要修改manifest，可以使用如下命令

```shell
docker manifest annotate --arch arm64 martindai/wechat-robot:1.1 martindai/wechat-robot-arm64:1.1
```
该命令表示，修改`martindai/wechat-robot:1.1`的manifest的`arm64`架构关联的镜像为`martindai/wechat-robot-arm64:1.1`

当然也可以删除manifest，重新创建

```shell
docker manifest rm martindai/wechat-robot:1.1
```

#### 推送manifest到仓库

信息确认没问题以后，把创建的menifest推送到远程仓库

```shell
docker manifest push martindai/wechat-robot:1.1
```
![9.png](https://imgs.doodl6.com/docker/build-multi-platform-image/9.png)

可以看到仓库多了一个多平台镜像，并且关联的就是我们之前上传的两个单平台镜像。

---

## 总结

方法一使用起来比较方便，也是个人比较推荐的，可以配置在稳定的测试/生产环境使用。

方法二使用起来稍微会麻烦一点，但是会比较灵活，比较适合一些定制化/开发场景。

两种方法都可以完成创建多平台镜像的工作，具体使用就看个人根据实际情况选择。
