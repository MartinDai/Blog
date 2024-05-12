---
title: Serverless部署应用并使用Cloudflare加速和支持HTTPS
date: 2023-05-22 22:16:00
categories: 
- 其他
---

![0](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/head.png)

## Serverless

Serverless 是一种云计算模型，它使开发人员能够构建和运行应用程序，而无需关心底层的服务器基础设施。在传统的应用程序开发中，开发人员需要管理服务器的配置、扩展和维护等任务。而在 Serverless 模型中，这些任务都由云服务提供商来处理，开发人员只需专注于编写应用程序的业务逻辑。

Serverless 模型适用于许多应用场景，如 Web 应用程序、移动后端、数据处理和物联网等。常见的 Serverless 平台包括:
国外：AWS Lambda、Azure Functions 和 Google Cloud Functions等
国内：阿里云的[函数计算 FC](https://www.aliyun.com/product/fc)，腾讯云的[云函数](https://cloud.tencent.com/product/scf)等

本文以**阿里云的函数计算FC**为例（阿里云每个月有免费的额度）

## Cloudflare

[Cloudflare](https://www.cloudflare-cn.com/) 是一家提供云计算和网络安全服务的公司。它提供了一系列的网络基础设施和安全功能，帮助网站和应用程序提供更快的加载速度、增强的安全性和高可靠性。

Cloudflare 的核心服务包括：CDN（内容分发网络），DDOS 保护，Web 应用程序防火墙（WAF），DNS服务，TLS 加密和边缘计算等。

本文需要使用到其中的**DNS服务**和**TLS加密**服务

## 应用准备

首先要准备好应用的部署文件，云服务厂商一般支持通过文件上传和容器镜像的方式进行部署。
如果是文件上传的方式部署，还需要选择运行环境，不同厂商支持的运行环境有所不同，需要提前了解好。
而镜像的方式就比较简单，只需要提供打包好的镜像即可。
所以个人推荐使用镜像的方式，这样可以拥有对运行环境完整的控制权，也方便版本管理。

本文接下来也将以镜像方式部署举例，其中镜像为已开源的[一个微信聊天机器人](https://github.com/MartinDai/weChatRobot-go)项目

<!--more-->

## 上传镜像到服务商平台

一般情况下需要把镜像文件上传到服务商平台以后才能进行版本管理和部署，或者通过服务商平台关联源码进行镜像打包，不同厂商可能有不同的策略，我这里选择的是在本地打包以后上传到平台的方式。

阿里云需要先在【容器镜像服务】里面开通个人版，然后【创建镜像仓库】以后根据操作指南执行即可
![1](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/1.png)

上传完以后可以点击左侧的【镜像版本】查看镜像版本列表
![2](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/2.png)

## 创建云函数

阿里云的云函数是挂在服务下的，所以需要先创建服务，然后再创建函数。
创建函数的时候选择【使用容器镜像创建】，请求处理程序类型选择【处理 HTTP 请求】，容器镜像泽点击下面的【选择 ACR 中的镜像】找到选择自己上传的仓库版本即可
![3](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/3.png)

后面还有【监听端口】不要忘记配置，接下来就是一些资源和环境变量相关的配置，可以根据自己的需要选择配置
![4](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/4.png)

最后是触发器配置，特别是请求方法记得要把应用内所声明过的类型都配上
![5](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/5.png)

最后点击【创建】即可完成函数的创建

## 验证云函数

云函数创建成功以后，回到函数列表，点击函数名称即可查看详情
![6](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/6.png)

切换到【测试函数】功能项，通过配置请求方式和路径即可向函数发起请求，如果函数能够如预期内响应，则表示函数已经部署成功
![7](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/7.png)

再切换到【触发器管理（URL）】功能项，可以看到该函数已经拥有了一个外网可以访问的域名，通过该域名也可以验证函数部署是否成功，**需要注意的是，该域名如果通过浏览器访问，则所有返回内容都会通过下载的方式响应**，这主要是因为国内提供网页服务是需要备案的。
![8](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/8.png)

## 自定义域名

完成上面的步骤以后，你就得到了一个可以通过后台提供服务的云函数了，像我这个微信机器人项目就是一个纯后台项目，所以是可以直接使用云函数提供的域名配置到微信公众号后台使用的。但是如果部署的是一个前台服务，那就必须要配置一个自定义的域名才能正常使用，下面就分别介绍一下自定义域名的两种情况。

### 使用阿里云已经备案的域名

如果你已经有一个在阿里云备案过的域名，那么可以在【函数计算 FC】功能首页找到【域名管理】功能
![9](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/9.png)

通过点击【添加自定义域名】，进入配置页面
设置好自定义的域名并在域名解析控制台配置好相应的CNAME
**HTTPS需要购买证书，或者手动上传（有的话可以选择）**
**CDN加速是要单独收费的，所以这里选择禁用**
最后设置路由配置到部署好的服务函数即可
![10](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/10.png)

### 没有已备案的域名

如果没有在阿里云已经备案的域名，则可以考虑把函数部署在海外服务节点，细心的读者可能已经发现了，我就是用的这种方式，上面的服务函数其实是部署在新加坡的，通过海外的节点提供服务就不需要提供的域名是备案过的，添加步骤跟上面备案的域名是一样的，只是在创建的时候少了域名备案校验这一步

配置完成以后，可以通过自定义域名访问验证函数资源

## CDN加速和HTTPS

前面我们在配置自定义域名的时候就发现**CDN加速**和**HTTPS**这两个都被设计为单独的收费项目了，但是我们可以使用Cloudflare免费使用这两项功能。

登录Cloudflare，选择【添加站点】，输入自己的域名添加
![11](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/11.png)

计划选择最下面的Free
![12](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/12.png)

继续按照步骤，登录到域名的服务商，把对应的DNS服务器改为Cloudflare的DNS服务器地址
还是以阿里云为例，在域名管理里面的【DNS管理】->【DNS修改】界面选择修改DNS服务器，两个都要改成Cloudflare的
![13](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/13.png)

完成以后在首页能看到添加的域名为有效即为设置成功
![14](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/14.png)

点击域名进入配置页面，选择左侧的【DNS】，把之前配置的云函数的CNAME在这里重新配置一遍

![15](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/15.png)

再点击左侧的【SSL/TLS】，勾选【完全】

![16](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/16.png)

至此，再次通过自定义域名访问验证，能够通过https访问并正常显示资源即表示成功

![17](https://imgs.doodl6.com/other/serverless-with-cloudflare-support-https/17.png)

**PS：Cloudflare自带免费的CDN加速功能，还有其他免费的功能可以自行研究**
