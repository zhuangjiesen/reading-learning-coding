# docker 操作

#### 查看 docker 镜像列表命令：

```
docker images
```
#### 查看运行状态的 docker 列表命令：

```
docker ps
```



#### docker中  与宿主共享文件夹

```
创建一个普通容器当做数据卷容器（就是专门用来同步文件的容器吧）
docker run -v /home/docker_files:/usr/docker_files  --name dataVol centos /bin/bash
```


#### 设置--name 命名数据卷名称
```
docker run —name docker_nginx -it --volumes-from dataVol  centos /bin/bash
```

#### 接着创建容器并且设置使用这个数据卷:
```
docker run -it --volumes-from dataVol centos /bin/bash
然后在/usr/docker_files 目录下就可以看到同步的文件了
```

### 坑：
#### 在容器中会遇到访问共享目录中ls命令及一些其他操作发生问题

```
cannot open directory .: Permission denied
```

这时候要在宿主机上设置
selinux的状态
getenforce 可以查看状态

setenforce 命令设置临时关闭，不用重启生效。
setenforce 0

这样就可以正常的同步并且操作同步文件夹内的数据了



配置环境变量
source /etc/profile
使文件生效

#### docker 中批量删除空的镜像:

```
docker rmi $(docker images | awk '/^<none>/ { print $3 }')

docker rmi REPOSITORY_Name 
```

保存容器并作为镜像:
配置好 docker 环境后保存容器

```
docker commit 698 learn/ping
```

保存对容器的修改
当你对某一个容器做了修改之后（通过在容器中运行某一个命令），可以把对容器的修改保存下来，这样下次可以从保存后的最新状态运行该容器。docker中保存状态的过程称之为committing，它保存的新旧状态之间的区别，从而产生一个新的版本。
目标：
首先使用docker ps -l命令获得安装完ping命令之后容器的id。然后把这个镜像保存为learn/ping。
提示：
1. 运行docker commit，可以查看该命令的参数列表。
2. 你需要指定要提交保存容器的ID。(译者按：通过docker ps -l 命令获得)
3. 无需拷贝完整的id，通常来讲最开始的三至四个字母即可区分。（译者按：非常类似git里面的版本号)


修改镜像的REPOSITORY与TAG
docker tag IMAGEID new_repository:newTAG 


#### 指定镜像运行容器:
```
docker run -it --volumes-from dataVol new_repository:newTAG /bin/bash


docker run -it --volumes-from dataVol centos:my_centos /bin/bash


开放8080 端口：
docker run -it --volumes-from dataVol -p 8080:8080 centos_docker_nginx:centos_docker_nginx_tag /bin/bash

```


-p后面的端口
把宿主端口号暴露给docker的端口号
之后就可以使用宿主主机的IP地址和这个端口来访问docker容器


#### docker 停止容器
```
docker stop container_id

```

### docker 进入容器方法

##### dokcer 
```
获取运行的docker 容器：
docker ps 

为了连接到容器，你还需要找到容器的第一个进程的PID

docker inspect --format "{{ .State.Pid }}" <container-id>

通过这个PID，你就可以连接到这个容器：

nsenter --target $PID --mount --uts --ipc --net --pid


```

##### docker  导出 images:

```
先在宿主机查看 images 运行状态:
docker ps 

导出:
docker export <imageid> > <image_file>

例子:
docker export 5c106ce7677a > nginx_docker.tar



导入:
docker import <image_file> <REPOSITORY_Name>:<TAG>

例子:
docker import nginx_docker.tar  centos_docker_nginx_1:centos_docker_nginx_1_tag
```
#### docker 保存镜像
进入镜像后，安装了一系列应用或者配置完，需要将镜像保存



```
命令：
docker commit <contain_id> <REPOSITORY_Name>:<TAG> 

例子：
docker commit bb2704ef409f centos_docker_nginx:centos_docker_nginx_tag

可以把之前的 <REPOSITORY_Name>:<TAG> 重复的覆盖
```


### docker 进入docker 容器

```
docker exec -it <container_id> bash

docker exec -it d996c366d966 /bin/bash  

docker exec -it 7a88ebe7c2cc bash
```


