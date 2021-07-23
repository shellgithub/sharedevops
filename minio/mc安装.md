## 安装 mc 并配置 minio

```

wget -c https://dl.min.io/client/mc/release/linux-amd64/mc \
-O /data/minio/bin/mc

chmod +x /data/minio/bin/mc

echo 'PATH=$PATH:/data/minio/bin/' >> /etc/profile
echo 'export MINIO_ACCESS_KEY=miniominio' >> /etc/profile
echo 'export MINIO_SECRET_KEY=miniominio' >> /etc/profile

source /etc/profile

# myminio 是 minio server 给的名字，url 是 endpoint，minioadmin 是用户名，密码

mc config host add myminio http://192.168.1.5:9001 miniominio miniominio
```

