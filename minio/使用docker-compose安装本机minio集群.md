# 使用docker-compose安装本机minio集群

参考于：https://www.yuque.com/docs/share/af5676cb-c0ed-48e7-97a7-e8351d7a1dfd?#（密码：hymf） 《Docker安装本机minio集群》

### 安装docker

删除已有的

```
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

或者
yum autoremove $(rpm -qa | grep docker)
```

设置阿里源

```
yum-config-manager \
    --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

安装所需的软件包

```
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

安装docker

```
yum install docker-ce
```

启动

```
systemctl start docker
```

### 安装docker-compose

下载

```
curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

添加权限

```
chmod +x /usr/local/bin/docker-compose
```

编写docker-compose.yaml（列如放在/home底下）挂载绝对路径注释掉最下面volumes。data1-1等换成绝对路径

```
mkdir /data/minio-docker

mkdir -p /data/minio-docker/data/minio{1...4}/data{1...2}

vim minio-compose.yaml
version: '3.7'

# starts 4 docker containers running minio server instances.
# using nginx reverse proxy, load balancing, you can access
# it through port 9000.
services:
  minio1:
    image: minio/minio:RELEASE.2020-11-25T22-36-25Z
    volumes:
      - /data/minio-docker/data/data1-1:/data1
      - /data/minio-docker/data/data1-2:/data2
    ports:
      - "9001:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio2:
    image: minio/minio:RELEASE.2020-11-25T22-36-25Z
    volumes:
      - /data/minio-docker/data/data2-1:/data1
      - /data/minio-docker/data/data2-2:/data2
    ports:
      - "9002:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio3:
    image: minio/minio:RELEASE.2020-11-25T22-36-25Z
    volumes:
      - /data/minio-docker/data/data3-1:/data1
      - /data/minio-docker/data/data3-2:/data2
    ports:
      - "9003:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio4:
    image: minio/minio:RELEASE.2020-11-25T22-36-25Z
    volumes:
      - /data/minio-docker/data/data4-1:/data1
      - /data/minio-docker/data/data4-2:/data2
    ports:
      - "9004:9000"
    environment:
      MINIO_ACCESS_KEY: minio
      MINIO_SECRET_KEY: minio123
    command: server http://minio{1...4}/data{1...2}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  nginx:
    image: nginx:latest
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "9000:9000"
    depends_on:
      - minio1
      - minio2
      - minio3
      - minio4

## 挂载绝对路径注释掉下面。data1-1等换成绝对路径
volumes:
  /data/minio-docker/data/data1-1:
  /data/minio-docker/data/data1-2:
  /data/minio-docker/data/data2-1:
  /data/minio-docker/data/data2-2:
  /data/minio-docker/data/data3-1:
  /data/minio-docker/data/data3-2:
  /data/minio-docker/data/data4-1:
  /data/minio-docker/data/data4-2:
```

添加 /data/minio-docker/nginx.conf

```
vi /data/minio-docker/nginx.conf

user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    # include /etc/nginx/conf.d/*.conf;

    upstream minio {
        server minio1:9000;
        server minio2:9000;
        server minio3:9000;
        server minio4:9000;
    }

    server {
        listen       9000;
        listen  [::]:9000;
        server_name  localhost;

         # To allow special characters in headers
         ignore_invalid_headers off;
         # Allow any size file to be uploaded.
         # Set to a value such as 1000m; to restrict file size to a specific value
         client_max_body_size 0;
         # To disable buffering
         proxy_buffering off;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 300;
            # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            chunked_transfer_encoding off;

            proxy_pass http://minio;
        }
    }
}
```

拉取镜像

```
docker-compose pull
```

运行（后台运行加 -d）

```
docker-compose up
```

访问本机ip的9000端口，账户：minio密码：minio123

ps：本例以卷标形式挂载数据

查看docker volume 

```
docker volume ls
```

查看 volume挂载位置

```
docker volume inspect （指定volume name）
```

ps：删除挂载卷

docker-compose down --volumes

