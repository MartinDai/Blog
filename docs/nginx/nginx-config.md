# 前言

> Nginx的反向代理功能应该是Nginx诸多功能里面最常用的一个功能了，正向代理的话可能使用的场景比较少，平时接触的也不多，本章内容仅包含这两个功能的基本使用配置，因为是本地版本的，所以不包含负载均衡相关的内容。


# 完整配置和注释
```nginx
user   root owner;
worker_processes  4;

#error_log  /usr/local/etc/nginx/logs/error.log;
#error_log  /usr/local/etc/nginx/logs/info.log info;

pid        /Users/martin/nginx.pid;

events {
    worker_connections  256;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #日志的格式
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #访问日志
    #access_log  /usr/local/etc/nginx/logs/access_log_pipe  main;

    #sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    gzip  on;

    #反向代理配置

    server {
        listen       443 ssl;          #监听443端口
        server_name  app.doodl6.com;   #服务域名
        ssl          on;               #是否开启SSL加密
        ssl_certificate         /Users/martin/Documents/ssl/doodl6.crt; # SSL加密证书
        ssl_certificate_key     /Users/martin/Documents/ssl/doodl6.key; # SSL加密秘钥

        charset UTF-8;   #编码指定

        location ~* ^.+\.(xls|woff2|log|jpg|jpeg|gif|png|ico|html|cfm|cfc|afp|asp|lasso|pl|py|txt|fla|swf|zip|js|css|less)$ {   #代理指定后缀的请求，这里配的是常见的前端资源
            proxy_pass https://127.0.0.1:80;  #转向提供内容的真实服务器地址，也可以配置本地目录（见HTTP代理配置）
            proxy_set_header Host $http_host;  #写入Header值，
            proxy_set_header referer "$http_referer";
        }  

        location = / {        #代理域名请求，也就只有域名的请求，如：https://app.doodl6.com
            proxy_pass https://127.0.0.1:8080;
            proxy_set_header Host $http_host;
        } 

        location ~ / {       #代理所有请求，不符合上面两种配置的请求都会走这个代理配置
            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host $http_host;
        }
    }

    server {
        listen       80;
        server_name  app.doodl6.com;
        charset UTF-8; 

        location ~* ^.+\.(xls|woff2|log|jpg|jpeg|gif|png|ico|html|cfm|cfc|afp|asp|lasso|pl|py|txt|fla|swf|zip|js|css|less|ico)$ {
            expires 30s;   #内容缓存30秒
            root /Users/martin/project/app/front;  #指定文件根目录
        } 

        location ~ / {
            proxy_pass http://127.0.0.1:8080;
            proxy_set_header Host $http_host;
        }
    }

    #正向代理配置

    server{
        listen 82;   #监听端口 
        resolver 8.8.8.8;   #DNS
        resolver_timeout 10s;  # DNS解析超时时间
        location / {
            proxy_pass http://$http_host$request_uri;
            proxy_set_header Host $http_host;
            proxy_buffers 256 4k;
            proxy_max_temp_file_size 0;
            proxy_connect_timeout 30;
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 301 1h;
            proxy_cache_valid any 1m;
        }
    }

    #本地反向转正向代理

    server {
        listen       80;
        server_name  proxy.doodl6.com;
        charset UTF-8; 

        location ~ / {
            proxy_pass http://127.0.0.1:82;  #转到本地正向代理
            proxy_set_header Host $http_host;
        }
    }

}
```