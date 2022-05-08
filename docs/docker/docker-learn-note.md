## 学习背景 ##
近些年，如果你是一名开发，那么`Docker`这个热门的技术名词你一定听说过，我也不例外，但是因为平时没接触过，所以一直都没动力去了解使用。最近在折腾自己维护在github上的项目的时候，发现项目所依赖的外部环境比较多，比如zookeeper、redis、elasticsearch等等，如果都安装一遍的话比较麻烦，而且因为是自己整理平时积累用的项目，对数据也没什么要求，只要有这样一个环境能保证项目运行起来就行了。第一时间想到的就是利用docker容器，这样自己不但可以顺便学习一下这个热门技术，还可以把搭建成果作为这个项目的一部分让有兴趣研究的项目关注者直接在接触到这个项目的时候可以快速把环境搭建起来，可谓是一举两得。

有兴趣的朋友可以访问项目地址:[https://github.com/MartinDai/SpringBoot-Project][1]

----------

# 安装 Docker Desktop

参考 [https://www.docker.com/get-started][2]，安装并启动后就可以使用下面的这些命令了

# docker 命令

## 查看docker版本

`docker --version`

查看当前docker版本，可以顺便验证docker是否安装成功且启动好了

![1](https://raw.githubusercontent.com/MartinDai/Blog/master/resources/images/docker/docker-learn-note/1.png)

## 查看帮助

`docker --help`

查看docker命令帮助，包含所有支持的操作命令使用规则及简介

![2](https://raw.githubusercontent.com/MartinDai/Blog/master/resources/images/docker/docker-learn-note/2.png)

还可以看某个指令的详细帮助,如：`docker images --help`，docker所有命令都可以在最后加上`--help`来查看该命令的使用帮助

![3](https://raw.githubusercontent.com/MartinDai/Blog/master/resources/images/docker/docker-learn-note/3.png)

## 拉取镜像

`docker pull [OPTIONS] NAME[:TAG|@DIGEST]`

下载镜像，如果没有指定镜像地址，默认从[官方的hub][3]下载指定的镜像，官方的hub提供了绝大多数热门的组件镜像，可以根据自己的需要进行搜索，这个网站有点类似github的模式，各个官方组件一般都会有比较详细的使用说明，比如Redis

![4](https://raw.githubusercontent.com/MartinDai/Blog/master/resources/images/docker/docker-learn-note/4.png)

可以使用`docker pull redis`下载最新版本的redis镜像

![5](https://raw.githubusercontent.com/MartinDai/Blog/master/resources/images/docker/docker-learn-note/5.png)

也可以指定下载的版本，如`docker pull redis:5.0.5`就可以下载5.0.5这个版本的镜像

当然有一些组件没有发布在官方的hub上，比如elasticsearch和kibana,这两个镜像需要从docker.elastic.co这个地址下载,可以使用`docker pull docker.elastic.co/elasticsearch/elasticsearch:6.2.4`下载

国内访问docker官方镜像有时候会超时，可以配置deamon.json使用国内的镜像
```
{ "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn/","https://hub-mirror.c.163.com","https://registry.docker-cn.com"], "insecure-registries": ["10.0.0.12:5000"] }
```

## 查看镜像

`docker images`

查看当前已下载的镜像列表

![6](https://raw.githubusercontent.com/MartinDai/Blog/master/resources/images/docker/docker-learn-note/6.png)

## 删除镜像

`docker rmi [OPTIONS] IMAGE [IMAGE...]`

举例：`docker rmi my-image:1.0`，表示删除名为`my-image`，版本号为`1.0`的镜像

## 使用镜像创建容器

`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`

使用指定镜像创建一个新的容器并运行，例如我们想创建运行redis容器，则可以使用命令`docker run --name my-redis -p 6379:6379 -d redis`，其中`--name`是`docker run`提供的参数，后面紧跟着的`my-redis`是对应的值，表示启动以后容器的名称，如果不指定则会使用随机生成的一个字符串。`-p 6379:6379`表示把本机端口6379映射到容器的6379端口，`-d`表示后台运行，如果不指定则启动后会自动进入容器控制台，并且退出控制台的同时会关闭容器。

## 容器查看

`docker container ls [OPTIONS]`

查看容器，可以通过`docker container ls`查看当前运行的容器，或者通过`docker container ls -a`查看所有创建的容器

## 删除容器

`docker container rm [OPTIONS] CONTAINER [CONTAINER...]`

举例：`docker container rm my-container1 my-container2`
表示同时删除name为`my-container1`和`my-container2`的两个容器

## 启动容器

`docker start [OPTIONS] CONTAINER [CONTAINER...]`

举例：`docker start my-container1`表示启动name为`my-container1`的容器

## 容器执行命令

`docker exec [OPTIONS] CONTAINER COMMAND [ARG...]`

对指定的容器执行命令，我们可以通过执行`docker exec -it my-redis /bin/bash`进入我们刚刚启动的容器

## 复制文件到容器

`docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH`

举例：`docker cp /Users/martin/Downloads/test.txt e4cf118af140:/var/lib/dev/`

其中：
`/Users/martin/Downloads/test.txt`为本地文件路径
`e4cf118af140`为容器ID
`/var/lib/dev/`为容器目录

## 复制容器文件到本地

`docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-`

举例：`docker cp e4cf118af140:/var/lib/dev/test.txt /Users/martin/Downloads/`

其中：
`/Users/martin/Downloads/`为本地路径
`e4cf118af140`为容器ID
`/var/lib/dev/test.txt`为容器文件路径

## 停止容器

`docker stop [OPTIONS] CONTAINER [CONTAINER...]`

停止容器，如果要停止上面启动的redis容器，则可以使用命令`docker stop my-redis`，指定了名字的好处立马就可以体现出来了，我们可以很精准的控制容器，而不需要去查询容器名称

## 修改容器作为新镜像

`docker commit [-m] [-a] CONTAINERID REPOSITORY[:TAG]`

`-m` 类似代码提交时的comment信息
`-a` 指定修改者信息
`CONTAINERID` 用来创建镜像的容器ID
`REPOSITORY[:TAG]` 目标镜像的仓库名和tag信息

创建成功后会新镜像的ID

举例：`docker commit -m "add something" -a "Martin Dai" e4cf118af140 my-image:latest`

## 推送镜像到远程

`docker push REPOSITORY[:TAG]`

举例：`docker push my-image:latest`

## 基于容器导出镜像

`docker export [OPTIONS] CONTAINER`

举例：`docker export -o my-image.tar my-container`，表示name为`my-container`的容器导出到`my-image.tar`文件

## 导入镜像

`docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]`

举例：`docker import my-image.tar my-image:latest`，表示将`my-image.tar`导入为镜像，名为`my-image`，版本号为`latest`


# docker-compose 命令

有时候项目依赖的外部环境比较多，但是又不想一个一个启动各个容器怎么办呢，`docker-compose`就是用来解决这个问题的，该命令可以通过使用指定的yml同时启动多个容器。

假如我们现在有个yml(具体yml规则可参考[官方文档][4])，且文件名为docker-compose.yml

```yml
version: '3.7' #标识docker-compose的版本，不同版本所支持的配置项有些不一样
services: #服务（也就是各个容器）配置
  redis: #服务名称，用于配置文件内关联使用
    image: redis:5.0 #镜像版本
    container_name: redis #容器名称
    command: redis-server /etc/redis/redis.conf #启动后执行的命令
    restart: always #启动失败是否重启
    volumes: #路径扩展映射配置
          - ./redis/:/etc/redis/ #把当前目录下的redis文件夹映射到容器中的/etc/redis文件夹，这样就可以在容器之外维护配置文件了
    ports: #端口映射配置
      - 6379:6379 #把本地的6379端口映射到容器的6379端口
    networks: #网络配置
      - net-cache
  memcached:
    image: memcached:1.5
    container_name: memcached
    restart: always
    ports:
      - 11211:11211
    networks:
      - net-cache
networks:
  net-cache:
    driver: bridge #配置桥接网络
```
进入该文件所在的目录，然后执行`docker-compose up`就可以启动redis和memcached这两个容器，如果要使用其他文件名，则可以使用`-f`参数来指定文件名，如`docker-compose -f docker-compose-cache.yml up`，如果需要后台运行，则可以在最后加上`-d`。

有启动就有停止，如果想要停止`docker-compose up`启动的容器，可以执行`docker-compose down`命令停止所有组合的容器。



[1]: https://github.com/MartinDai/SpringBoot-Project
[2]: https://www.docker.com/get-started
[3]: https://hub.docker.com/
[4]: https://docs.docker.com/compose/gettingstarted/