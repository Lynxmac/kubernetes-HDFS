# kubernetes-HDFS
## 项目： [apache-spark-on-k8s/kubernetes-HDFS](https://github.com/apache-spark-on-k8s/kubernetes-HDFS)
## kubernetes HDFS搭建过程
*原项目已经提供了搭建教程，本文仅限于通过该项目的官方文档搭建不成功的同学使用，其中记录了搭建过程中碰到的问题，方便其他同学避免跟我踩同样的坑*



### 主要问题：
1. 1.5中kube-dns没有直接解析node节点的hostname,而是间接通过其他DNS服务器解析，所以直接使用项目搭建HDFS时，由于我的集群机器的/etc/resolv.conf没有可以解析node主机名的DNS服务器，导致pod启动失败不断重建，一段时间后所有hdfs的pod都处于CrashLoopBackOff状态。

报错信息：
``` 
...
/************************************************************
STARTUP_MSG: Starting DataNode
STARTUP_MSG:   host = java.net.UnknownHostException: k8s-node3: k8s-node3: Temporary failure in name resolution
STARTUP_MSG:   args = []
STARTUP_MSG:   version = 2.7.2
...
```

2. 通过ConfigMap将静态的/etc/hosts注入到pod的共享环境变量中，并没有生效，直接运行namenode的镜像，ping检查发现其并没有查询/etc/hosts文件，并且2.7.2版本的镜像都存在这个问题，只能重新构建镜像。

```
$ ping k8s-master
ping: unknown host
```
### 集群环境：
```
通过kubeadm安装，1台master,3台node，已安装kube-dns
192.168.1.101 k8s-master
192.168.1.102 k8s-node1
192.168.1.103 k8s-node2
192.168.1.104 k8s-node3

kubernetes版本:
Client Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.1", GitCommit:"82450d03cb057bab0950214ef122b67c83fb11df", GitTreeState:"clean", BuildDate:"2016-12-14T00:57:05Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"5", GitVersion:"v1.5.3", GitCommit:"029c3a408176b55c30846f0faedf56aae5992e9b", GitTreeState:"clean", BuildDate:"2017-02-15T06:34:56Z", GoVersion:"go1.7.4", Compiler:"gc", Platform:"linux/amd64"}

helm 版本:
Client: &version.Version{SemVer:"v2.5.0", GitCommit:"012cb0ac1a1b2f888144ef5a67b8dab6c2d45be6", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.5.0", GitCommit:"012cb0ac1a1b2f888144ef5a67b8dab6c2d45be6", GitTreeState:"clean"}



```
*安装helm时，最好通过nodeSelector让tiller的pod运行在kubernetes集群的master上，这样可以避免访问apiserver时验证失败，导致连接不上apiserver*

### 搭建过程

#### 一. 将项目下载到k8s集群的master上
```
git clone https://github.com/apache-spark-on-k8s/kubernetes-HDFS

```

#### 二. 创建ConfigMap，将resolv.conf写入到到配置中，方便hdfs-datanode与hdfs-namenode共享使用

1. 查看kube-dns的IP地址
```
$ kubectl get svc --all-namespaces | grep kube-dns

...
kube-system   kube-dns               10.96.0.10     <none>        53/UDP,53/TCP   3d
...
```

2. 查看主机在网络中的主机名
```
  $ kubectl run -i -t --rm busybox --image=busybox --restart=Never  \
      --command -- cat /etc/resolv.conf
  文档出现的结果：
  ...
  search default.svc.cluster.local svc.cluster.local cluster.local MYCOMPANY.COM
  ...
  本地集群输出的结果：
  ...
   search default.svc.cluster.local svc.cluster.local cluster.local
  ...
```
*这里就是出现问题的关键点, 更详细的信息可以看这里[issue13](https://github.com/apache-spark-on-k8s/kubernetes-HDFS/issues/13)*

3. 修改hdfs-resolv-conf/templates/resolve-conf-configmap.yaml,添加hosts配置内容
```
...
data:
  hosts: |
    192.168.1.101 k8s-master
    192.168.1.102 k8s-node1
    192.168.1.103 k8s-node2
    192.168.1.104 k8s-node3
  resolv.conf: |
...
```
*替换成自己的集群hosts文件*

4. 创建ConfigMap

```
$ cd ~/kubernetes-HDFS-master/charts
$ helm package hdfs-resolv-conf
$ helm install -n my-hdfs-resolv-conf \
      --set clusterDnsIP=MY-KUBE-DNS-IP  \
      hdfs-resolv-conf
```
*将MY-KUBE-DNS-IP替换自己的kube-dns的IP地址*


#### 三. 运行HDFS namenode daemonset

1. 选择要运行Namenode的node节点，为节点打上标签

  ```
  $ kubectl label nodes YOUR-HOST hdfs-namenode-selector=hdfs-namenode-0
  ```
2. 修改~/kubernetes-HDFS-master/charts／hdfs-namenode-k8s/templates/namenode-statefulset.yaml
  修改下载镜像（解决hosts不生效的问题），如果有需要的话可以自己build一个，已经把相关Dockerfile放到build-images了文件夹中
  ```
  ...
         containers:
        - name: hdfs-namenode
          image: registry.cn-hangzhou.aliyuncs.com/kb_containers/hadoop-namenode:latest
          env:
  ...
  ```
  将hosts注入到pod的共享配置变量中
  ```
  ...
          - configMap:
            name: hdfs-resolv-conf
            items:
            - key: resolv.conf
              path: resolv.conf
            - key: hosts
              path: hosts
          name: resolv-conf-volume
  ...
  ```
  挂载hosts配置文件到/etc/hosts中
  ```
  ...
              # See https://github.com/dshulyak/kubernetes.github.io/commit/d58ba7b075bb4848349a2c920caaa08ff3773d70
            - name: resolv-conf-volume
              mountPath: /etc/resolv.conf
              subPath: resolv.conf
            - name: resolv-conf-volume
              mountPath: /etc/hosts
              subPath: hosts
      # Pin the pod to a node. You can label your node like below:
  ...
  ```

3. 运行namenode daemonset
  ```
  $ cd ~/kubernetes-HDFS-master/charts
  $ helm package hdfs-namenode-conf
  $ helm install -n my-hdfs-namenode hdfs-namenode-k8s
  ```
  
4. 查看运行情况
```
  $ kubectl get pods | grep hdfs-namenode
  hdfs-namenode-0       1/1       Running   0          9m
  ```

#### 四. 运行HDFS datanode daemonset

1. 为不需要运行datanode的节点打上hdfs-datanode-exclude=yes标签,

  ```
  $ kubectl label node YOUR-MASTER-NAME／NAMENODE hdfs-datanode-exclude=yes
  ```
2. 修改~/kubernetes-HDFS-master/charts／hdfs-datanode-k8s/templates/datanode-statefulset.yaml
  修改下载镜像
  ```
  ...
         containers:
        - name: hdfs-datanode
          image: registry.cn-hangzhou.aliyuncs.com/kb_containers/hadoop-datanode:latest
          env:
  ...
  ```
  将hosts注入到pod的共享配置变量中
  ```
  ...
          - configMap:
            name: hdfs-resolv-conf
            items:
            - key: resolv.conf
              path: resolv.conf
            - key: hosts
              path: hosts
          name: resolv-conf-volume
  ...
  ```
  挂载hosts配置文件到/etc/hosts中
  ```
  ...
              # See https://github.com/dshulyak/kubernetes.github.io/commit/d58ba7b075bb4848349a2c920caaa08ff3773d70
            - name: resolv-conf-volume
              mountPath: /etc/resolv.conf
              subPath: resolv.conf
            - name: resolv-conf-volume
              mountPath: /etc/hosts
              subPath: hosts
      # Pin the pod to a node. You can label your node like below:
  ...
  ```

3. 运行datanode daemonset
  ```
  $ cd ~/kubernetes-HDFS-master/charts
  $ helm package hdfs-datanode-conf
  $ helm install -n my-hdfs-ndatanode hdfs-datanode-k8s
  ```
  
4. 查看运行情况

  ```
  $ kubectl get pods | grep hdfs-datanode-
  hdfs-datanode-pfn5k   1/1       Running   0          10m
  hdfs-datanode-xc11w   1/1       Running   0          10m
  ```
  
  
#### 五. 查看运行状况


```
[root@k8s-master charts]# kubectl get all -n default
NAME                     READY     STATUS    RESTARTS   AGE
po/hdfs-datanode-pfn5k   1/1       Running   0          1h
po/hdfs-datanode-xc11w   1/1       Running   0          1h
po/hdfs-namenode-0       1/1       Running   0          1h

NAME                CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
svc/hdfs-namenode   None         <none>        8020/TCP   1h
svc/kubernetes      10.96.0.1    <none>        443/TCP    4d

NAME                         DESIRED   CURRENT   AGE
statefulsets/hdfs-namenode   1         1         1h


```

##### 若一切运行状态ok,则可以直接访问HTTP://NAMENODE-IP:50070测试


#### 其他问题，如果通过namenode的hadoop fs写入文件时出现No route to the host的问题，可以尝试关闭运行datanode的node节点的防火墙