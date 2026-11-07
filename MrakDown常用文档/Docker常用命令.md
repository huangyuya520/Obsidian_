### Docker基础命令
```text
systemctl start docker      # 启动docker
systemctl stop docker       # 停止docker
systemctl restart docker     # 重启docker
systemctl status docker     # 查看docker运行状态
docker version             # 查看版本信息
```



### Docker镜像命令
Docker镜像是容器运行的基础
```text
docker pull <镜像名>/<容器ID>   # 拉取镜像
docker images                  # 列出查看本地镜像
docker rmi <容器名/ID>            # 删除镜像
docker rmi -f <容器名/ID>        # 强制删除镜像
docker search <镜像名>           # 搜索镜像
```



### Docker容器命令
```text
docker ps                      # 列出正在运行的容器
docker ps -a                   # 列出所有容器（包括已停止）
docker exec -it <容器ID> bash    # 进入容器
docker stop <容器名/ID>          # 停止容器
docker run <镜像名>              # 运行镜像容器
-d：后台运行容器
-p：端口映射
--name：指定容器名称
-v：挂载卷;例如-v 主机路径:容器路径
docker run -d -p <xxx>:<xx> --name <自定义容器名> \ 
-v /host/path:/container/path \    
... \
 <容器名>

docker start [容器ID或容器名]      # 开始启动容器
docker stop [容器ID或容器名]       # 停止运行容器
docker restart [容器ID或容器名]    # 重启容器
docker rm <容器名/ID>            # 删除容器

docker rename <当前容器名称或ID> <新容器名称>     # 容器重命名
```







