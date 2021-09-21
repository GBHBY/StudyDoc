# Docker

- docker pull +镜像名<:tags>
  - 从远程抽取镜像 tags是版本的意思，不加tags，默认下载最新版本
- docker images 
  - 查看本地镜像
- docker run +镜像名<:tags> 
  - 创建容器，启动应用，如果本地没有镜像，那么会自动下载镜像，并创建启动
- docker ps 
  - 查看正在运行中的镜像
- docker rm <-f> +容器ID
  - 当容器不再使用，使用该命令并加上容器编号，就会删除，如果当前容器还在运行中，加上`-f`就会强制删除
- docker rmi <-f> +镜像名<:tags>
  - 删除镜像，加上`-f`，如果该镜像已经存在了容器，那么也会强制删除

## 启动容器

- docker run  -p 端口号：容器内部端口  镜像名 -d
  - -d:后台运行
- docker exec -it  容器编号 /bin/bash ：进入容器

## DockerFile编写

### 文件格式

````
docker build -t 机构/镜像名<:tags> DockerFile目录
//这个是用来执行下面这个文件的
````

````
DockerFile编写
//这是基准镜像，是在这个镜像中拓展
FROM tomcat:latest
//镜像的持有者
MAINTAINER mashibing.com
//应用内的工作目录
WORKDIR /usr/local/tomcat/webapps
// 第一个docker-web是指与该文件同级目录的文件夹，ADD会将文件夹下的内容复制到docker内部的docker-web内
// 第二个docker-web是webapps下的docker-web目录
//ADD test.tar.gz /路径 如果这样些话的 ADD是能够解压的
ADD docker-web ./docker-web
````

````
DockerFile 指令
RUN：在构建file的时候执行
CMD：在容器启动时运行，但不一定会运行，比如
// docker run xxxx ls ，此时，如果dickerfile中有cmd命令，但是因为在启动docker的时候，添加了ls，那么就会取代CMD，
ENTARYPOINT：一定会执行，和CMD一样，可以和CMD组合执行

````

