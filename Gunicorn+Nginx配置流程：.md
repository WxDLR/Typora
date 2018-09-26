

## Gunicorn+Nginx配置流程：

#### 一、项目背景：

​	在部署开源项目WebODM(使用Django2.x版本)时遇到nginx配置问题导致网页静态文件无法访问。

#### 二、排查过程：

- 检查Django中settings配置是否正确，检查如下，无问题：

```python
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'build', 'static')
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'app', 'static'),
]
```
- 通过查询发现需要配置urls.py文件的路由配置，如下：

```python
from django.contrib.staticfiles.urls import staticfiles_urlpatterns

urlpatterns = [
    url("")...
]

urlpatterns += staticfiles_urlpatterns()
# 无效
```

- 推测可能由于nginx配置文件出错，检查WebODM原始项目nginx.conf文件如下：

```
...
http {
  include /etc/nginx/mime.types;

  # fallback in case we can't determine a type
  default_type application/octet-stream;
  access_log /tmp/nginx.access.log combined;
  sendfile on;

  upstream app_server {
    # fail_timeout=0 means we always retry an upstream even if it failed
    # to return a good HTTP response

    # for UNIX domain socket setups
    server unix:/tmp/gunicorn.sock fail_timeout=0;
  }

  server {
    listen 8000 deferred;
    client_max_body_size 0;

    server_name localhost;

    keepalive_timeout 5;

    proxy_connect_timeout 60s;
    proxy_read_timeout 300000s;

    # path for static files
    location /static {
      root /webodm/build;
    }

    # path for certain media files that don't need permissions enforced
    location /media/CACHE {
      root /webodm/app;
    }
    location /media/settings {
      autoindex on;
      root /webodm/app;
    }

    location / {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      # enable this if and only if you use HTTPS
      # proxy_set_header X-Forwarded-Proto https;
      proxy_set_header Host $http_host;

      # we don't want nginx trying to do something clever with
      # redirects, we set the Host: header above already.
      proxy_redirect off;
      proxy_pass http://app_server;
    }
  }
}
```

> 发现该文件中static块中的路径配置为相对路径，且文件夹名称出错。但更正后仍无法使用。推测可能原因是项目启动文件有问题：start.sh，其部分代码如下：

```shell
**start.sh**
if [ "$1" = "--setup-devenv" ] || [ "$2" = "--setup-devenv" ] || [ "$1" = "--no-gunicorn" ]; then
    congrats
    # 如果使用带参启动（"--setup-devenv"，"--setup-devenv"，"--no-gunicorn"），则使用django原生服务部署，未启动nginx代理。
    python manage.py runserver 0.0.0.0:8000
else
    if [ -e /webodm ] && [ ! -e /webodm/build/static ]; then
       echo -e "\033[91mWARN:\033[39m /webodm/build/static does not exist, CSS, JS and other files might not be available."
    fi

    echo "Generating nginx configurations from templates..."
    for templ in nginx/*.template
    do
        echo "- ${templ%.*}"
        envsubst '\$WO_PORT \$WO_HOST' < $templ > ${templ%.*}
    done

    # Check if we need to auto-generate SSL certs via letsencrypt
    if [ "$WO_SSL" = "YES" ] && [ -z "$WO_SSL_KEY" ]; then
        echo "Launching letsencrypt-autogen.sh"
        ./nginx/letsencrypt-autogen.sh
    fi

    # Check if SSL key/certs are available
    conf="nginx.conf"
    if [ -e nginx/ssl ]; then
        echo "Using nginx SSL configuration"
        conf="nginx-ssl.conf"
    fi

    congrats

    nginx -c $(pwd)/nginx/$conf
    gunicorn webodm.wsgi --bind unix:/tmp/gunicorn.sock --timeout 300000 --max-requests 250 --preload
fi
```

> 该脚本文件主要作用是确定Django中代理服务器的启动方式，以及nginx选择配置文件启动，无问题；
>
> 推测nginx配置问题，通过自己写配置文件以确定问题。以下分别为gunicorn配置以及nginx配置，如下：

```python
**gunicorn.conf.py**

import multiprocessing
bind = "127.0.0.1:8080" #监听端口
workers = 2 #线程数
errorlog = "/home/parallels/Desktop/Okaygis/WebODM/gunicorn.error.log" #日志文件
loglevel = "debug" #日志级别
proc_name = "WebODM" #项目名

#启动命令：
gunicorn webodm.wsgi:application -c ./gunicorn.conf.py
```

```shell
**mynginx.conf**

worker_processes 1;

# Change this if running outside docker!
user root root;
pid /tmp/nginx.pid;
error_log /tmp/nginx.error.log;

events {
  worker_connections 1024; # increase if you have lots of clients
  accept_mutex off; # set to 'on' if nginx worker_processes > 1
  use epoll;
}

http {
  include /etc/nginx/mime.types;

  # fallback in case we can't determine a type
  default_type application/octet-stream;
  access_log /tmp/nginx.access.log combined;
  sendfile on;

  upstream app_server {
    # fail_timeout=0 means we always retry an upstream even if it failed
    # to return a good HTTP response

    # for UNIX domain socket setups
    server unix:/tmp/gunicorn.sock fail_timeout=0;
  }

  server {
    listen 8000 deferred;
    client_max_body_size 0;

    server_name localhost;

    keepalive_timeout 5;

    proxy_connect_timeout 60s;
    proxy_read_timeout 300000s;

    # path for static files
    location /static {
      root /home/parallels/Desktop/Okaygis/WebODM/build;
    }

    # path for certain media files that don't need permissions enforced
    location /media/CACHE {
      root /webodm/app;
    }
    location /media/settings {
      autoindex on;
      root /webodm/app;
    }

    location / {
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      # enable this if and only if you use HTTPS
      # proxy_set_header X-Forwarded-Proto https;
      proxy_set_header Host $http_host;

      # we don't want nginx trying to do something clever with
      # redirects, we set the Host: header above already.
      proxy_redirect off;
      proxy_pass http://127.0.0.1:8080;
    }
  }
}

sudo nginx -c /home/parallels/Desktop/Okaygis/WebODM/nginx/mynginx.conf 
```

> nginx配置文件主要更改点为使用绝对路径定位静态文件：启动之后成功

#### 三、结论

- 静态文件无法访问问题流程：检查网络—检查Django配置文件—检查路由文件—检查Nginx配置；
- 初步了解gunicorn配置以及与nginx的搭配。



