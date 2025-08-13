# Typora图床

###### 		typora正常需要本地有图片才能正常显示，所以要将文档传给其他人，需要把图片一起打包发送会使得文件变大而且不方便发送。

##### 通过``内网穿透``+``lsky``+``群晖``  本地部署图床

##### docker拉取mysql镜像并运行

```cmd
docker run -itd --name mysql -p 33066:3306 --restart=always -e MYSQL_ROOT_PASSWORD=root mysql
```

##### lsky拉取镜像并运行

![image-20241112105825175](../images/image-20241112105825175.png)

##### lsky注册（`注意连接地址需要是ipconfig的VMip地址`

![img](https://i-blog.csdnimg.cn/blog_migrate/1eef44673ca5f4d357ed4125c27fac7b.png)
