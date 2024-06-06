---
title: 使用cronolog切割nginx访问日志,定时清理旧日志
date: 2018-05-08 20:48:00
categories: 
- Nginx
---

## 准备工作

### 安装cronolog
`brew instal cronolog`
如果遇到这个错误

![1](https://imgs.doodl6.com/nginx/use-cronolog-split-nginx-log/1.webp)

执行
`sudo chown -Rwhoami:admin /usr/local/sbin`
如果没有`/usr/local/sbin`这个文件夹先执行
`mkdir /usr/local/sbin`

## 使用cronolog切割日志

### 创建日志源管道文件
`mkfifo /usr/local/etc/nginx/access.log.pipe`

### 配置nginx访问日志
`access_log  /usr/local/etc/nginx/access.log.pipe  main;`

### 启动cronolog，当access.log.pipe产生数据时，使用cronolog将access.log.pipe中的数据转移到access.log.%Y-%m-%d

`nohup cat /usr/local/etc/nginx/logs/access.log.pipe | nohup /usr/local/sbin/cronolog /usr/local/etc/nginx/logs/access.log.%Y-%m-%d &`

### 启动或重启nginx
`nginx start或nginx -s raload`

## 定时清理旧日志

### 创建清理脚本
`vi delete_nginx_logs.sh`

保存内容
```text
LOG_PATH="/usr/local/etc/nginx/logs"
save_days=7
find $LOG_PATH -mtime +$save_days -exec rm -rf {} \;
```
### 添加定时执行任务
`crontab -e`

每天0点执行
`0 0 0 * * sh /usr/local/nginx/delete_nginx_logs.sh`