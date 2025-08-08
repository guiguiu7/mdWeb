### 1.下载minio镜像

 默认下载最新版本

```
docker pull minio/minio
```

下载指定版本

```
docker pull minio/minio:版本号
```



### 2.创建目录

一共创建两个目录，一个用来存放配置，另一个用来存储上传的文件

```
mkdir -p /home/minio/config
mkdir -p /home/minio/data
或
mkdir -p  /home/minio/{conf,data}
```

### 3.创建minio容器并运行

9090端口是minio的客户端端口

MINIO_ACCESS_KEY ：账号

MINIO_SECRET_KEY ：密码（账号长度必须大于等于5，密码长度必须大于等于8位）

```
docker run -p 9000:9000 -p 9090:9090 \
     --net=host \
     --name minio \
     -d --restart=always \
     -e "MINIO_ACCESS_KEY=minioadmin" \
     -e "MINIO_SECRET_KEY=minioadmin" \
     -v /home/minio/data:/data \
     -v /home/minio/config:/root/.minio \
     minio/minio server \
     /data --console-address ":9090" -address ":9000"
```

