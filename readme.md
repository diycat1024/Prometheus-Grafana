[TOC]
## 概述
作为应用与Kubernetes的监控体系，Prometheus具备诸多的优势，如：

*   Kubernetes默认支持,非常适合容器和微服务
*   无依赖，安装方便，上手容易
*   社区活跃，它不仅仅是个工具，而是生态
*   已有很多插件或者exporter，可以适应多种应用场景的数据收集需要
*   Grafana默认支持,提供良好的可视化
*   高效，单一Prometheus可以处理百万级的监控指标，每秒处理数十万的数据点


在部署之前，先来了解一下Prometheus各个组件的作用吧！

*   MertricServer：是k8s集群资源使用情况的聚合器，收集数据给K8s集群内使用，如：kubectl，hpa,scheduler
*   PrometheusOperator：是一个系统检测和警报工具箱，用来存储监控数据；
*   NodeExporter：用于各node的关键度量指标状态数据；
*   kubeStateMetrics：收集k8s集群内资源对象数据，指定告警规则；
*   Prometheus：采用pull方式收集apiserver，scheduler，controller-manager，kubelet组件数据，通过http协议传输；
*   Grafana：是可视化数据统计和监控平台。

web管理部署一个就可以。如果安装了其他web管理可以删除
[https://github.com/prometheus/prometheus](https://github.com/prometheus/prometheus)

## 下载
**下载prometheus所需文件：**
```
链接: https://pan.baidu.com/s/12W5DGlVZqWYtMKVxgfR6GA 提取码: 8uw8

或者
git clone https://github.com.cnpmjs.org/diycat1024/Prometheus-Grafana
```
### 在kubernetest集群中创建namespace
```
apiVersion: v1
kind: Namespace
metadata: 
  name: ns-monitor
  labels:
    name: ns-monitor
    
kubectl apply -f namespace.yaml
```
### 安装node-exporter
在kubernetest集群中部署node-exporter，Node-exporter用于采集kubernetes集群中各个节点的物理指标，比如：Memory、CPU等。可以直接在每个物理节点是直接安装，这里我们使用DaemonSet部署到每个节点上，使用 hostNetwork: true 和 hostPID: true 使其获得Node的物理指标信息，配置tolerations使其在master节点也启动一个pod。
```
kubectl apply -f node-exporter.yaml
```
 检验node-exporter是否成功运行
```
[root@master1 ~]# kubectl get pod -n ns-monitor 
NAME                          READY   STATUS
grafana-677d945674-56m5n      1/1     Running
node-exporter-vkpt2           1/1     Running
node-exporter-zkh9s           1/1     Running
prometheus-6c9574d5ff-292bq   1/1     Running
[root@master1 ~]# kubectl get svc -n ns-monitor 
NAME                    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          
grafana-service         NodePort   10.96.101.190    <none>        3000:32405/TCP
node-exporter-service   NodePort   10.107.147.241   <none>        9100:31672/TCP
prometheus-service      NodePort   10.97.249.230    <none>        9090:30437/TCP

浏览器访问： http://主机ip:31672/metrics
```
[http://192.168.84.241:31672/metrics](http://192.168.84.241:31672/metrics)
## 安装nsf server 参考最下面的解决办法
搭建服务一条龙
```
sudo apt install nfs-kernel-server
```
```
sudo vim /etc/exports
/nfs/prometheus/data/ 192.168.84.75/24(rw,no_root_squash,no_all_squash,sync) 
/nfs/grafana/data/ 192.168.84.75/24(rw,no_root_squash,no_all_squash,sync) 
```
```
sudo mkdir -p /nfs/prometheus/data
sudo chmod -R 777 /nfs/prometheus/data/

sudo mkdir -p /nfs/grafana/data
sudo chmod -R 777 /nfs/grafana/data/
sudo exportfs -r

sudo systemctl start rpcbind
sudo systemctl status rpcbind
```
## 部署 Prometheus pod
prometheus.yaml 中包含rbac认证、ConfigMap等。
需要修改 PersistentVolume.server
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "prometheus-data-pv"
  labels:
    name: prometheus-data-pv
    release: stable
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /nfs/prometheus/data
    server: 192.168.84.75   # nsf server的ip
```

```
sudo kubectl apply -f prometheus.yaml 
```
检验是否正常运行
```
[root@master1 ~]# sudo kubectl get pod -n ns-monitor 
NAME                          READY   STATUS
grafana-677d945674-56m5n      1/1     Running
node-exporter-vkpt2           1/1     Running
node-exporter-zkh9s           1/1     Running
prometheus-6c9574d5ff-292bq   1/1     Running
[root@master1 ~]# sudo kubectl get svc -n ns-monitor 
NAME                    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          
grafana-service         NodePort   10.96.101.190    <none>        3000:32405/TCP
node-exporter-service   NodePort   10.107.147.241   <none>        9100:31672/TCP
prometheus-service      NodePort   10.97.249.230    <none>        9090:30437/TCP
```
如果是宿主机则可以访问：http://10.97.249.230:9090
浏览器访问： http://主机ip:30437/graph


## 在kubernetest中部署grafana
```
sudo kubectl apply -f grafana.yaml
```
检验是否正常运行
```
sudo kubectl get pod -n ns-monitor 

NAME                          READY   STATUS
grafana-677d945674-56m5n      1/1     Running
node-exporter-vkpt2           1/1     Running
node-exporter-zkh9s           1/1     Running
prometheus-6c9574d5ff-292bq   1/1     Running

sudo kubectl get svc -n ns-monitor 

NAME                    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          
grafana-service         NodePort   10.96.101.190    <none>        3000:32405/TCP
node-exporter-service   NodePort   10.107.147.241   <none>        9100:31672/TCP
prometheus-service      NodePort   10.97.249.230    <none>        9090:30437/TCP
```

浏览器访问： http://主机ip:32405/graph/login  默认用户名和密码：admin/admin

## grafana的配置
### 配置数据源
Grafana需要拉取prometheus的接口获取数据，才能画图。
![](http://yuerblog.cc/wp-content/uploads/2019/01/word-image.png)

然后添加datasource：
![](https://yuerblog.cc/wp-content/uploads/2019/01/word-image-1.png)
选择prometheus：
![](https://yuerblog.cc/wp-content/uploads/2019/01/word-image-2.png)
配置服务端地址,地址写上面的 http://prometheus-service:9090(域名:9090)， k8s的coreDNS会解析域名prometheus-service为对应的10.97.249.230地址 ：
![](https://yuerblog.cc/wp-content/uploads/2019/01/word-image-3.png)
### 导入Dashboard模板
![](https://www.itnotebooks.com/wp-content/uploads/2018/08/QQ%E6%B5%8F%E8%A7%88%E5%99%A8%E6%88%AA%E5%9B%BE20180823232633.png)
**这里使用的是kubernetes集群模板，模板编号315，在线导入地址https://grafana.com/dashboards/315**
![](https://www.itnotebooks.com/wp-content/uploads/2018/08/QQ%E6%B5%8F%E8%A7%88%E5%99%A8%E6%88%AA%E5%9B%BE20180823232658.png)
![](https://www.itnotebooks.com/wp-content/uploads/2018/08/QQ%E6%B5%8F%E8%A7%88%E5%99%A8%E6%88%AA%E5%9B%BE20180823232717.png)
### **效果展示：**
![](https://www.itnotebooks.com/wp-content/uploads/2018/08/QQ%E6%B5%8F%E8%A7%88%E5%99%A8%E6%88%AA%E5%9B%BE20180823232815.png)
![](https://www.itnotebooks.com/wp-content/uploads/2018/08/QQ%E6%B5%8F%E8%A7%88%E5%99%A8%E6%88%AA%E5%9B%BE20180823232815.png)
![](https://www.itnotebooks.com/wp-content/uploads/2018/08/QQ%E6%B5%8F%E8%A7%88%E5%99%A8%E6%88%AA%E5%9B%BE20180823232838.png)
## Kubernetes创建pod一直处于ContainerCreating排查和解决
通过 `sudo kubectl get pod -n ns-monitor`查看pod名字和状态
查看  prometheus     pod信息
```
sudo kubectl describe pod prometheus  -n ns-monitor 

mount: /var/lib/kubelet/pods/9b618933-87b6-496a-90da-25a4e9e782c3/volumes/kubernetes.io~nfs/prometheus-data-pv: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.
  Warning  FailedMount  18s  kubelet, catrefine-virtual-machine  MountVolume.SetUp failed for volume "prometheus-data-pv" : mount failed: exit status 32

```
## 解决办法
搭建nsf服务
```
sudo apt install nfs-kernel-server
```
默认情况下，在Ubuntu 18.04上，NFS版本2是禁用的。版本3和版本4已启用。您可以通过运行以下[`cat`](https://www.myfreax.com/linux-cat-command/)命令来验证：
```
sudo cat /proc/fs/nfsd/versions
```
写入 exports
```
cat  /etc/exports
sudo echo /nfs/prometheus/data/ 192.168.84.75/24(rw,no_root_squash,no_all_squash,sync) >>  /etc/exports
sudo echo /nfs/prometheus/data/ 192.168.84.75/24(rw,no_root_squash,no_all_squash,sync) >>  /etc/exports
```
```
可以设定的参数主要有以下这些：

rw：可读写的权限；
ro：只读的权限；
no_root_squash：登入到NFS主机的用户如果是root，该用户即拥有root权限；
root_squash：登入NFS主机的用户如果是root，该用户权限将被限定为匿名使用者nobody；
all_squash：不管登陆NFS主机的用户是何权限都会被重新设定为匿名使用者nobody。
anonuid：将登入NFS主机的用户都设定成指定的user id，此ID必须存在于/etc/passwd中。
anongid：同anonuid，但是变成group ID就是了！
sync：资料同步写入存储器中。
async：资料会先暂时存放在内存中，不会直接写入硬盘。
insecure：允许从这台机器过来的非授权访问。
```

验证配置的/nfs/prometheus/data/是否正确
```
sudo mkdir -p /nfs/prometheus/data
sudo chmod -R 777 /nfs/grafana/data/
sudo exportfs -r
```
启动服务
```
sudo systemctl start rpcbind
sudo systemctl status rpcbind
```

主节点，子节点检验：
```
[root@szy-k8s-master /]# showmount -e 192.168.84.75
Export list for 192.168.84.75:
/nfs/prometheus/data 192.168.84.75/24

[root@szy-k8s-salve/]# showmount -e 192.168.84.75
Export list for 192.168.84.75:
/nfs/prometheus/data 192.168.84.75/24
```
```
NFS客户端的操作：
1、showmout命令对于NFS的操作和查错有很大的帮助，所以我们先来看一下showmount的用法
showmout
-a ：这个参数是一般在NFS SERVER上使用，是用来显示已经mount上本机nfs目录的cline机器。
-e ：显示指定的NFS SERVER上export出来的目录。
2、mount nfs目录的方法：
mount -t nfs hostname(orIP):/directory /mount/point 
```