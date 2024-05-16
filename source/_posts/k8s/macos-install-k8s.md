---
title: MacOS 安装k8s
date: 2022-07-21 19:15:00
categories: 
- k8s
---

## 安装前准备

确保本地已经安装并启动好了[Docker Desktop](https://www.docker.com/products/docker-desktop/)

## 拉取k8s镜像（如果本地网络好可以正常拉取到k8s官方镜像，可以跳过这一步）

克隆git仓库到本地
```
git clone https://github.com/gotok8s/k8s-docker-desktop-for-mac.git
```

进入项目目录，执行
```
./load_images.sh
```

等待所有镜像拉取完成
![1.png](https://imgs.doodl6.com/k8s/macos-install-k8s/1.webp)

## 部署k8s

进入`Docker Decktop`的设置页面，勾选`Kubernetes`设置页的配置，点击右下角的`Apply & Restart`按钮，等待k8s完成部署
![2.png](https://imgs.doodl6.com/k8s/macos-install-k8s/2.webp)

完成以后可以验证一下部署状态
```
kubectl cluster-info

kubectl get nodes

kubectl describe node
```
![3.png](https://imgs.doodl6.com/k8s/macos-install-k8s/3.webp)

## 安装k8s Dashboard

应用推荐配置，这里需要注意修改`v2.6.0`为兼容上面安装的kubelet的版本，具体可查看[https://github.com/kubernetes/dashboard/releases](https://github.com/kubernetes/dashboard/releases)
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml
```

![4.png](https://imgs.doodl6.com/k8s/macos-install-k8s/4.webp)

## 创建用户并获取token

### 创建admin-user

复制如下配置，保存文件为`dashboard-adminuser.yaml`

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```
然后执行
```
kubectl apply -f dashboard-adminuser.yaml
```

### 绑定cluster-admin授权

复制如下配置，保存文件为`dashboard-clusteradmin.yaml`

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```
然后执行
```
kubectl apply -f dashboard-clusteradmin.yaml
```

### 获取登录token

开启代理并且设置代理端口为8001

```
kubectl proxy --port=8001
```

打开新的命令窗口，执行

检查用户信息是否存在
```
curl 'http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/serviceaccounts/admin-user'
```

获取token不带参数
```
curl 'http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/serviceaccounts/admin-user/token' -H "Content-Type:application/json" -X POST -d '{}'
```

获取token带参数
```
curl 'http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/serviceaccounts/admin-user/token' -H "Content-Type:application/json" -X POST -d '{"kind":"TokenRequest","apiVersion":"authentication.k8s.io/v1","metadata":{"name":"admin-user","namespace":"kubernetes-dashboard"},"spec":{"audiences":["https://kubernetes.default.svc.cluster.local"],"expirationSeconds":7600}}'
```
![5.png](https://imgs.doodl6.com/k8s/macos-install-k8s/5.webp)

## 登录k8s Dashboard

复制上一步返回的token信息，浏览器访问如下地址，填入token即可登录
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
```

![6.png](https://imgs.doodl6.com/k8s/macos-install-k8s/6.webp)