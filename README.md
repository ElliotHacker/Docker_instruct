# Docker_instruct
Docker部署常用指令及redis添加主从机

# 移除旧版本docker
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

# 配置docker yum源，使用下载更快的源，官网文档慢
sudo yum install -y yum-utils
sudo yum-config-manager \
--add-repo \
http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#出现 repo saved to /etc/yum.repos.d/docker-ce.repo表示安装完成

# 安装 最新 docker docker 运行时容器环境：containerd.io 构建插件的工具：docker-buildx-plugin 批量工具：docker-compose-plugin
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

#显示 Complete! 安装完成

# 启动& 开机启动docker； enable + start 二合一
systemctl enable docker --now

# 配置加速 设置从国内网址进行下载，如果不配置，会默认从国外网址下载 配置好以后，最后两行重启docker后台进程和docker
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

#查看运行中的容器
docker ps
# ps菜单
#CONTAINER ID   IMAGE     COMMAND                  CREATED              STATUS                        PORTS     NAMES
#正在运行应用的ID  使用镜像名  启动命令                   启动时间              启动状态（up启动中，exited已退出）  占用端口   应用容器的名字（随机）
#b170083b7abe   nginx     "/docker-entrypoint.…"   About a minute ago   Up About a minute             80/tcp    nostalgic_allen
#查看所有容器，能看到已退出的
docker ps -a
#搜索镜像
docker search nginx
#下载镜像
docker pull nginx
#下载指定版本镜像
docker pull nginx:1.26.0
#查看所有镜像
docker images
#删除指定id的镜像 或者镜像的完整信息，比如nginx:1.26.0
docker rmi e784f4560448

# OPTIONS参数项 IMAGE镜像 COMMAND命令 ARG参数
# Usage:  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
#运行一个新容器
docker run nginx
#停止容器
docker stop nostalgic_allen
#启动容器 可以使用 CONTAINER ID或者 NAMES来启动，使用ID时，可以不输入全，能区分即可，类似模糊查询
docker start 592 或者 docker start nostalgic_allen
#重启容器，重启以后CONTAINER ID会重置
docker restart 592
#查看容器资源占用情况
docker stats 592
#查看容器日志
docker logs 592
#删除指定容器，必须先stop停止
docker rm 592
#强制删除指定容器，不需要停止
docker rm -f 592
# 后台启动容器，name mynginx给容器命名，不暴露端口，外网访问不了
docker run -d --name mynginx nginx
# 后台启动并暴露端口 暴露端口需要在服务器中设置可以访问
docker run -d --name mynginx -p 80:80 nginx
# 进入容器内部 it表示进入到交互模式 这样比较麻烦，使用映射减少指令操作
docker exec -it mynginx /bin/bash
#之后会进入mynginx容器，此时若使用linux命令，ls / 可以查看容器内目录

#可以使用cd /user/share/nginx/html 来进入html文件夹

#echo "<h1>Hello,Docker<h1>" > index.html 写入内容

#显示文件代码 类似于linux的vi
cat index.html
#使用docker存储可以把容器映射到主机中，类似快捷方式

# 提交容器变化打成一个新的镜像，引号内为此次更新备注
docker commit -m "update index.html" mynginx mynginx:v1.0
# 保存镜像为指定文件
docker save -o mynginx.tar mynginx:v1.0
# 删除多个镜像
docker rmi bde7d154a67f 94543a6c1aef e784f4560448
# 加载镜像
docker load -i mynginx.tar


# 登录 docker hub 用户名输入显示 密码输入不显示
docker login
# 重新给镜像打标签
docker tag mynginx:v1.0 用户名/mynginx:v1.0
# 推送镜像 推送后其他人就可以pull + tags下载，推送一个lastest版本防止用户不加tags就下载而报错
docker push leifengyang/mynginx:v1.0

#ps -a 打印所有容器，-q打印容器ID 组合得到打印所有容器ID
docker ps -aq
#强制删除所有正在运行的容器
docker rm -f $(docker ps -aq)

#存储
#目录挂载，将外部文件夹挂载到容器内，删除容器不影响外部文件夹，初始启动内外都是空的
docker run -d -p 80:80 -v /app/nghtml:/usr/share/nginx/html

#卷映射，使用目录挂载，可能会由于目录为空而报错，将config映射以后，文件也会映射过去，初始启动以内部为准
docker run -d -p 80:80 -v /app/nghtml:/usr/share/nginx/html -v ngconf:/etc/nginx --name xxxx nginx
#映射后的位置 卷映射的位置：/var/lib/docker/volumes/
cd /var/lib/docker/volumes/ngconf/_data
#可以使用linux的命令vi修改文件
vi nginx.config
#对所有卷操作
docker volume
#查看所有卷
docker volume ls
#创建卷 XXX
docker volume create XXX
#查看卷详情
docker volume inspect XXX

#网络
#查看docker网络，docker0是默认设置的容器IP
ip a
#docker 为每一个容器分配一个默认IP，使用容器IP+容器端口可以互相访问
#为了便于操作，预防IP变化，docker允许自定义网络
#创建网络
docker network create xxx 比如mynet
#查看现有网络
docker network ls

#redis主从机群实现读写分离，主机用于写请求，从机用于读请求
#使用bitnami 提供的镜像
#https://hub.docker.com/r/bitnami/redis 可能需要翻墙
docker pull bitnami/redis
#自定义网络
docker network create mynet
#主节点 斜线\是换行
docker run -d -p 6379:6379 \
#目录挂载
-v /app/rd1:/bitnami/redis/data \
#为了稳定访问，主机名作为域名
-e REDIS_REPLICATION_MODE=master \
#设置主机访问密码
-e REDIS_PASSWORD=123456 \
--network mynet --name redis01 \
bitnami/redis
#启动失败的话，很可能是没有权限
#递归添加权限 rd1为设定的文件夹名，需要在app目录下执行
chmod -R 777 rd1
#然后重启
docker restart redis01
#从节点 同理添加从节点权限
#可以先添加路径
mkdir rd2
chmod -R 777 rd2
#再来启动redis02
docker run -d -p 6380:6379 \
-v /app/rd2:/bitnami/redis/data \
#表示是从机
-e REDIS_REPLICATION_MODE=slave \
#主机地址
-e REDIS_MASTER_HOST=redis01 \
#主机端口
-e REDIS_MASTER_PORT_NUMBER=6379 \
#访问主机密码
-e REDIS_MASTER_PASSWORD=123456 \
#为从机设置密码
-e REDIS_PASSWORD=123456 \
--network mynet --name redis02 \
#指定镜像
bitnami/redis
#主从机配置完成