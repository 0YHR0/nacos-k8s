# Kubernetes Nacos K8SNacos

本项目包含一个可构建的Nacos Docker Image,旨在利用StatefulSets在[Kubernetes](https://kubernetes.io/)上部署[Nacos](https://nacos.io)



# 已知限制

* 暂时不支持动态增量扩容
* 必须使用持久卷,本项目实现NFS持久卷的例子,如果使用emptyDirs可能会导致数据丢失



# Docker 镜像

在[build](https://github.com/paderlol/nacos-k8s/tree/master/build)目录中包含了已经打好包的Nacos(基于**develop**分支,已提PR,目前的release版本都不支持k8s集群)项目包,以及镜像制作文件,镜像基础环境Ubuntu 16.04、Open JDK 1.8(JDK 8u111).目前镜像已经提交到[Docker Hub](https://hub.docker.com/)。



# 项目目录

| 目录   | 描述                                |
| ------ | ----------------------------------- |
| build  | 构建Nacos镜像的项目包以及Dockerfile |
| deploy | k8s部署yaml文件                     |



# 配置属性

* nacos-pvc-nfs.yaml 或者 nacos-quick-start.yaml 属性列表

| 名称                  | 是否必填 | 描述                                    |
| --------------------- | -------- | --------------------------------------- |
| mysql.master.db.name  | 是       | 数据库主库名称                          |
| mysql.master.port     | 否       | 数据库主库端口                          |
| mysql.slave.port      | 是       | 数据库从库端口                          |
| mysql.master.user     | 是       | 数据库主库用户名                        |
| mysql.master.password | 是       | 数据库主库密码                          |
| NACOS_REPLICAS        | 是       | 集群数量,必须和**replicas**属性保持一致 |
| NACOS_SERVER_PORT     | 否       | Nacos端口,不填写默认8848                |
| PREFER_HOST_MODE      | 是       | 开启Nacos集群节点域名支持               |



* **nfs**目录下deployment.yaml 属性列表

| 名称       | 是否必填 | 描述                     |
| ---------- | -------- | ------------------------ |
| NFS_SERVER | 是       | NFS server地址           |
| NFS_PATH   | 是       | NFS server配置的共享目录 |
| server     | 是       | NFS server地址           |
| path       | 是       | NFS server配置的共享目录 |



* mysql目录下yaml文件 属性列表

| 名称                       | 是否必填 | 描述                                                        |
| -------------------------- | -------- | ----------------------------------------------------------- |
| MYSQL_ROOT_PASSWORD        | 否       | root密码                                                    |
| MYSQL_DATABASE             | 是       | 数据库名称,从库无需配置                                     |
| MYSQL_USER                 | 是       | 数据用户名,从库无需配置                                     |
| MYSQL_PASSWORD             | 是       | 数据库用户密码,从库无需配置                                 |
| MYSQL_REPLICATION_USER     | 是       | 从库复制用户名,主从库配置文件中需要配置相同                 |
| MYSQL_REPLICATION_PASSWORD | 是       | 从库复制用户密码,主从库配置文件中需要配置相同               |
| Nfs:server                 | 是       | 如果没有部署nfs,请使用mysql-master-local  mysql-slave-local |
| Nfs:path                   | 是       | 如果没有部署nfs,请使用mysql-master-local  mysql-slave-local |



# 使用指南



## 前提要求

* 本项目的使用,是基于你已经对Kubernetes有一定的认知,所以对如何搭建K8S集群,请自行google或者百度
* NFS安装方面也不是本文的重点,请自行google或者百度



## 环境准备

* 机器配置(作者演示使用阿里云ECS)

| 机器内网IP  | 主机名     | 机器配置                                                     |
| ----------- | ---------- | ------------------------------------------------------------ |
| 172.17.79.3 | k8s-master | CentOS Linux release 7.4.1708 (Core) 单核 内存4G 普通云盘40G |
| 172.17.79.4 | node01     | CentOS Linux release 7.4.1708 (Core) 单核 内存4G 普通云盘40G |
| 172.17.79.5 | node02     | CentOS Linux release 7.4.1708 (Core) 单核 内存4G 普通云盘40G |

* Kubernetes 版本：**1.12.2** （如果你和我一样只使用了三台机器,那么记得开启master节点的部署功能）
* NFS 版本：**4.1** 在k8s-master进行安装Server端,并且指定共享目录,本项目指定的**/data/nfs-share**
* Git

 

## 搭建步骤

### Clone项目

在每台机器上都Clone本工程,演示工程就是导入根目录,所以部署路径都是root/nacos-k8s

```shell
git clone https://github.com/paderlol/nacos-k8s.git
```







### 部署NFS

* 创建角色 K8S在1.6以后默认开启了RBAC

```shell
kubectl create -f deploy/nfs/rbac.yaml
```

提示：如果你的K8S命名空间不是默认"default",那么在创建RBAC之前先执行以下脚本

```shell
# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
$ NAMESPACE=${NS:-default}
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/nfs/rbac.yaml

```



* 创建ServiceAccount 以及部署NFS-Client Provisioner

```shell
kubectl create -f deploy/nfs/deployment.yaml
```



* 创建NFS StorageClass

```shell
kubectl create -f deploy/nfs/class.yaml
```



* 查看NFS是否运行正常

```shell
kubectl get pod -l app=nfs-client-provisioner
```



### 部署数据库

数据库是以指定节点的方式部署,主库部署在node01节点,从库部署在node02节点.

* 部署主库

```shell
#进入clone下来的工程根目录
cd nacos-k8s 
# 在k8s上创建mysql主库
kubectl create -f deploy/mysql/mysql-master-nfs.yml
```



* 部署备库

```shell
#进入clone下来的工程根目录
cd nacos-k8s 
# 在k8s上创建mysql备库
kubectl create -f deploy/mysql/mysql-slave-nfs.yml
```



* 部署后查看数据库是否已经正常运行

```shell
#查看主库是否正常运行
kubectl get pod 
NAME                         READY   STATUS    RESTARTS   AGE
mysql-master-gf2vd                        1/1     Running   0          111m

#查看备库是否正常运行
kubectl get pod 
mysql-slave-kf9cb                         1/1     Running   0          110m
```


### 部署Nacos 



* 获取主库从库在K8S的地址

```shell
# 查看主库和从库的cluster ip
kubectl get svc

mysql            NodePort    10.105.42.247   <none>        3306:31833/TCP   2d23h
mysql-bak        NodePort    10.105.35.138   <none>        3306:31522/TCP   2d23h
```



* 修改配置文件**depoly/nacos/nacos-pvc-nfs.yaml**,找到如下配置,填入上一步查到的主库和从库地址

```yaml
data:
  mysql.master.db.name: "数据库名称"
  mysql.master.port: "主库端口"
  mysql.slave.port: "从库端口"
  mysql.master.user: "主库用户名"
  mysql.master.password: "主库用户密码"
```



* 创建并运行Nacos集群

``` shell
kubectl create -f nacos-k8s/deploy/nacos/nacos-pvc-nfs.yaml
```



* 查看是否运行正常

```shell
kubectl get pod -l app=nacos


AME      READY   STATUS    RESTARTS   AGE
nacos-0   1/1     Running   0          19h
nacos-1   1/1     Running   0          19h
nacos-2   1/1     Running   0          19h
```







## 测试

### 服务注册

```shell
curl -X PUT 'http://集群地址:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'
```



### 服务发现

```shell
curl -X GET 'http://集群地址:8848/nacos/v1/ns/instances?serviceName=nacos.naming.serviceName'
```



### 配置推送

```shell
curl -X POST "http://集群地址:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=helloWorld"
```



### 配置获取

```shell
curl -X GET "http://集群地址:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"
```



## 常见问题

Q:如果不想搭建NFS,并且想体验nacos-k8s?

 A:可以跳过部署nfs的步骤,最后创建运行nfs时,使用一下以下方式创建

```shell
kubectl create -f nacos-k8s/deploy/nacos/nacos-quick-start.yaml
```



Q:如果未搭建NFS,数据库怎么部署？

A:可以使用本地持久方式部署,如下

```shell
kubectl create -f nacos-k8s/deploy/mysql/nacos-master-local.yaml
kubectl create -f nacos-k8s/deploy/mysql/nacos-master-local.yaml
```

