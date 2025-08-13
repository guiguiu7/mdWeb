### docker-desktop设置Resoureces页面下地址是docker文件的存储位置。

##### 首先关闭docker-desktop，使用命令`wsl --list -v`查看进程是否停止。

##### <img src="C:\Users\shadow\AppData\Roaming\Typora\typora-user-images\image-20241112134121886.png" alt="image-20241112134121886" style="zoom:200%;" />

##### 停止后执行`wsl --shutdown`关闭wsl

##### 导出 `wsl --export docker-desktop D://docker-desktop.tar"`

##### ![image-20241112134447110](C:\Users\shadow\AppData\Roaming\Typora\typora-user-images\image-20241112134447110.png)

##### 注销docker-desktop`wsl --unregister docker-desktop`

![image-20241112134631780](C:\Users\shadow\AppData\Roaming\Typora\typora-user-images\image-20241112134631780.png)

##### 导入`wsl --import docker-desktop E:\docker\data E:\docker\docker-desktop.tar --version 2`

![image-20241112134711599](C:\Users\shadow\AppData\Roaming\Typora\typora-user-images\image-20241112134711599.png)

##### 导入成功后打开docker-desktop