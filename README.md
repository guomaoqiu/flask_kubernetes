
## 序
此次部署是在上篇博文中搭建的K8S集群基础之上进行的，涉及使用到的容器已经推到Dockerhub

## 逻辑流程图

## 初始配置

##### 1.创建一个namespace 供此次部署使用
```
[root@linux-node1 ~]# kubectl create namespace flask-app-extions-stage
[root@linux-node1 ~]# kubectl get ns
NAME                      STATUS    AGE
default                   Active    29d
flask-app-extions-stage   Active    1m
kube-public               Active    29d
kube-system               Active    29d
```
##### 1. 创建用NFS存储目录
```
# 用于flask-app代码存放目录，当pod启动时通过NFS的方式挂载进去
[root@linux-node1 ~]# mkdir /data/flask-app-data
# 用于flask的mysql数据存放目录，当pod启动时通过NFS挂载进去
[root@linux-node1 ~]# mkdir /data/flask-app-db
```
##### 2. 安装NFS Server(略)
##### 3. 写入配置
```
[root@linux-node1 ~]# echo "/data/flask-app-db *(rw,sync,no_subtree_check,no_root_squash)" > /etc/exports
[root@linux-node1 ~]# echo "/data/flask-app-data *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
```
##### 4.重启验证
```
[root@linux-node1 ~]# systemctl restart nfs-server
[root@linux-node1 ~]# exportfs
/data/nginx-data
		<world>
/data/flask-app-db
		<world>
/data/flask-app-data
		<world>
```
##### 5.创建flask-app使用的pv及pvc
```
[root@linux-node1 ~]# kubectl create -f flask_k8s/flask-app/flask_app_data_pv.yaml
persistentvolume "flask-app-data-pv" created
[root@linux-node1 ~]# kubectl create -f flask_k8s/flask-app/flask_app_data_pvc.yaml
persistentvolumeclaim "flask-app-data-pv-claim" created
```
##### 6.创建flask-app-db使用的pv及pvc
```
[root@linux-node1 ~]# kubectl create -f flask_k8s/flask_app_db/flask_app_db_pv.yaml
persistentvolume "flask-app-db-pv" created
[root@linux-node1 ~]# kubectl create -f flask_k8s/flask_app_db/flask_app_db_pvc.yaml
persistentvolumeclaim "flask-app-db-pv-claim" created
[root@linux-node1 ~]# kubectl get pv -n flask-app-extions-stage
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                                             STORAGECLASS   REASON    AGE
flask-app-data-pv   5Gi        RWO            Recycle          Bound     flask-app-extions-stage/flask-app-data-pv-claim                            5m
flask-app-db-pv     5Gi        RWO            Recycle          Bound     flask-app-extions-stage/flask-app-db-pv-claim                              26s
[root@linux-node1 ~]# kubectl get pvc -n flask-app-extions-stage
NAME                      STATUS    VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
flask-app-data-pv-claim   Bound     flask-app-data-pv   5Gi        RWO                           4m
flask-app-db-pv-claim     Bound     flask-app-db-pv     5Gi        RWO                           28s
[root@linux-node1 ~]#
```
## 部署 flask-app-db
运行flask交互数据的数据库使用的是mysql，上面我已经创建了用于基于NFS存储mysql数据的持久存储目录 /data/flask-app-db
#### 1. 创建配置 MySQL 密码的 Secret
```
[root@linux-node1 ~]# kubectl create secret generic mysql-pass --from-literal=password=YOUR_PASSWORD
[root@linux-node1 ~]# kubectl  get secret -n flask-app-extions-stage
NAME                  TYPE                                  DATA      AGE
default-token-fr2sg   kubernetes.io/service-account-token   3         47m
mysql-pass            Opaque                                1         14s
```
#### 2. 部署 MySQL：
```
[root@linux-node1 ~]# kubectl create -f flask_k8s/flask_app_db/flask_app_db_deploy.yaml
deployment.apps "flask-app-db" created
[root@linux-node1 ~]# kubectl create -f flask_k8s/flask_app_db/flask_app_db_service.yaml
service "flask-app-db" created

# 查看状态
NAME                                   READY     STATUS    RESTARTS   AGE       IP            NODE
pod/flask-app-db-6f55458666-h2dk7      1/1       Running   1          22h       10.2.15.108   192.168.56.12
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE       SELECTOR
service/flask-app-db      NodePort    10.1.68.29     <none>        3306:30006/TCP   22h       app=flask-app-db
NAME                                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES                         SELECTOR
deployment.extensions/flask-app-db      1         1         1            1           22h       mysql        mysql:5.6                      app=flask-app-db,tier=mysql

# 以上可以知道该pod运行在节点192.168.56.12上面，我这里使用的是NodePort方式，然后映射了一个30006端口出来到节点上面

# 可以尝试登陆测试
[root@linux-node1 flask_app_db]# mysql -uroot -pdevopsdemo -h192.168.56.12 -P 30006
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.6.40 MySQL Community Server (GPL)

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.02 sec)

MySQL [(none)]>

# 顺便我们在这里手动创建一下flask-app需要用到的数据库 devopsdemo, 因为在flask-app在启动过程中需要去初始化数据库并创建数据表。
MySQL [(none)]> CREATE DATABASE devopsdemo;
Query OK, 1 row affected (0.01 sec)

#数据库目录,可以看到mysql数据已经通过nfs的方式存储到了我们nfs server所在主机映射出去的目录
[root@linux-node1 ~]# tree /data/flask-app-db/ -L 1
/data/flask-app-db/
├── auto.cnf
├── devopsdemo
├── ibdata1
├── ib_logfile0
├── ib_logfile1
├── mysql
└── performance_schema

3 directories, 4 files
```

## 部署flask-app
#### 1.将flask程序代码放到/data/flask-app-data/
```
[root@linux-node1 ~]# tree /data/flask-app-data/ -L 1
/data/flask-app-data/
├── app
├── config.py
├── flask_uwsgi.ini
├── LICENSE
├── manage.py
├── README.md
├── requirements.txt
├── screenshots
├── supervisord.conf
└── tests

3 directories, 9 files
```
#### 2. 修改flask程序连接数据库的配置信息
```
[root@linux-node1 ~]# vim /data/flask-app-data/config.py
......
......
    db_host = 'flask-app-db' # 在pod启动过程中会去加载k8s的环境变量；这个flask-app-db 就是mysql的svc
    db_user = 'root'         # 默认为root
    db_pass = "devopsdemo"   # 数据库密码
    db_name = 'devopsdemo'   # 数据库名称
......
......
```

#### 3. 部署flask-app
```
[root@linux-node1 ~]# kubectl create -f flask_k8s/flask_app/flask_app_deployment.yaml
deployment.apps "flask-app" created
[root@linux-node1 ~]# kubectl create -f flask_k8s/flask_app/flask_app_service.yaml
service "flask-app" created

# 查看状态:
[root@linux-node1 ~]# kubectl get pod,svc,deployment,rc -o wide   -n flask-app-extions-stage
NAME                                   READY     STATUS    RESTARTS   AGE       IP            NODE
pod/flask-app-65646687ff-4gg7g         1/1       Running   0          17h       10.2.42.125   192.168.56.13
pod/flask-app-65646687ff-7d5g6         1/1       Running   0          17h       10.2.42.124   192.168.56.13
pod/flask-app-65646687ff-xkq6k         1/1       Running   0          17h       10.2.42.123   192.168.56.13
pod/flask-app-db-6f55458666-h2dk7      1/1       Running   1          22h       10.2.15.108   192.168.56.12

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE       SELECTOR
service/flask-app         ClusterIP   10.1.173.169   <none>        3032/TCP         22h       app=flask-app
service/flask-app-db      NodePort    10.1.68.29     <none>        3306:30006/TCP   22h       app=flask-app-db

NAME                                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES                         SELECTOR
deployment.extensions/flask-app         3         3         3            3           22h       flask-app    guomaoqiu/python27baseenv:v2   app=flask-app,tier=frontend
deployment.extensions/flask-app-db      1         1         1            1           22h       mysql        mysql:5.6                      app=flask-app-db,tier=mysql


# 以上可知 我的flask-app运行了三个副本；
# 并且采用的是ClusterIP Type,3032是flask+supervisor+uwsgi 之后 uwsgsi 暴露出来的二端口
# 此时我们访问是没有用的；因为uwgsi需要结合用到Nginx来访问；于是我下面将会部署nginx ---> uwgsi(flask-app)
```

-----

## 部署flask-app-nginx
因为这个nginx的配置我这里需要修改做一些定制方面的配置，所以有一个将单个配置文件挂载到pod里面的需求；于是这里使用到了k8s的ConfigMap功能
#### 1. 找到需要挂载进pod的nginx配置文件:
```
[root@linux-node1 ~]# kubectl create  configmap nginx-conf --from-file=/root/flask_k8s/flask_app_nginx/nginx.conf -n  flask-app-extions-stage
[root@linux-node1 ~]# kubectl get configmap -n flask-app-extions-stage
NAME         DATA      AGE
nginx-conf   1         11s
# 通过describe就可以看到这个conf的详细内容，其实就在我们nginx.conf配置文件的基础上加上了k8s一些特有的属性值
[root@linux-node1 ~]# kubectl describe configmap/nginx-conf -n flask-app-extions-stage
```
#### 2.部署flask-app-nginx
```
[root@linux-node1 flask_app_nginx]# kubectl create -f flask_app_nginx_deploy.yaml
deployment.apps "flask-app-nginx" created
[root@linux-node1 flask_app_nginx]# kubectl create -f flask_app_nginx_service.yaml
service "flask-app-nginx" created

# 查看状态：
[root@linux-node1 ~]# kubectl get pod,svc,deployment,rc -o wide   -n flask-app-extions-stage
NAME                                   READY     STATUS    RESTARTS   AGE       IP            NODE
pod/flask-app-65646687ff-4gg7g         1/1       Running   0          17h       10.2.42.125   192.168.56.13
pod/flask-app-65646687ff-7d5g6         1/1       Running   0          17h       10.2.42.124   192.168.56.13
pod/flask-app-65646687ff-xkq6k         1/1       Running   0          17h       10.2.42.123   192.168.56.13
pod/flask-app-db-6f55458666-h2dk7      1/1       Running   1          22h       10.2.15.108   192.168.56.12
pod/flask-app-nginx-657fd4c57c-p6qdx   1/1       Running   12         18h       10.2.42.119   192.168.56.13
pod/flask-app-nginx-657fd4c57c-v4qsp   1/1       Running   12         18h       10.2.42.117   192.168.56.13
pod/flask-app-nginx-657fd4c57c-xtpmm   1/1       Running   12         18h       10.2.42.118   192.168.56.13

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE       SELECTOR
service/flask-app         ClusterIP   10.1.173.169   <none>        3032/TCP         22h       app=flask-app
service/flask-app-db      NodePort    10.1.68.29     <none>        3306:30006/TCP   22h       app=flask-app-db
service/flask-app-nginx   ClusterIP   10.1.193.82    <none>        80/TCP           21h       app=flask-app-nginx

NAME                                    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS   IMAGES                         SELECTOR
deployment.extensions/flask-app         3         3         3            3           22h       flask-app    guomaoqiu/python27baseenv:v2   app=flask-app,tier=frontend
deployment.extensions/flask-app-db      1         1         1            1           22h       mysql        mysql:5.6                      app=flask-app-db,tier=mysql
deployment.extensions/flask-app-nginx   3         3         3            3           21h       nginx        nginx:latest                   app=flask-app-nginx

# 以上 运行了三个flak-app-nginx ,采用的是ClusterIP Type
# 访问验证,直接访问的是flask-app-nginx 的VIP(ClusterIP)
[root@linux-node1 ~]# curl -I 10.1.193.82/auth/login
HTTP/1.1 200 OK
Server: nginx/1.15.0
Date: Wed, 27 Jun 2018 04:35:12 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 4026
Connection: keep-alive
Set-Cookie: session=eyJjc3JmX3Rva2VuIjp7IiBiIjoiT0RrNVpXUmpZV014T1daalkyUmlOamRsWXpBNVpqUTVZMk0wTkRnMU9ERm1NVFUzTURrME5BPT0ifX0.DhSlgA.Biu7EYC7qfj4x8--HlR8VUZFUgk; HttpOnly; Path=/
```
以上说明我们能够成功的访问到我们的flask-app服务啦,在用linux 终端下的w3m http://10.1.193.82/auth/login 访问一下呢，说明也是没问题的；那后端uwgsi or flask-app or flask-nginx的访问日志此时毋庸置疑是已经有了的。
![](https://github.com/guomaoqiu/flask_kubernetes/blob/master/screenshots/15300742642956.jpg?raw=true)

那问题来了；此时我只是在集群内部能够访问，那如何在外面访问呢？那就主要将flask_nginx_service.yaml中的Type该为nodePort,然后从新部署一下flask_app_nginx_services

```
[root@linux-node1 ~]# more /root/flask_k8s/flask_app_nginx/flask_app_nginx_service.yaml
kind: Service
apiVersion: v1
metadata:
  name: flask-app-nginx
  namespace: flask-app-extions-stage
spec:
  type: NodePort
  selector:
    app: flask-app-nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30001
  selector:
    app: flask-app-nginx

[root@linux-node1 ~]# kubectl describe svc/flask-app-nginx -n flask-app-extions-stage
Name:              flask-app-nginx
Namespace:         flask-app-extions-stage
Labels:            <none>
Annotations:       <none>
Selector:          app=flask-app-nginx
Type:              ClusterIP
IP:                10.1.193.82
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.2.42.117:80,10.2.42.118:80,10.2.42.119:80
Session Affinity:  None
Events:            <none>
```

此时如果是外网访问就应该是NodeIP+nodePort啦：
![](https://github.com/guomaoqiu/flask_kubernetes/blob/master/screenshots/15300750247449.jpg?raw=true)


但是这种方式并不是很理想；本来就是要让他直接访问80 port的，于是采用另外一种暴露端口的方式------Ingress

（这里我还是把flask-app-nginx的service改回了ClusterIP)

-----

## 部署 Ingress-Nginx
Ingress 使用开源的反向代理负载均衡器来实现对外暴漏服务，比如 Nginx、Apache、Haproxy等。Nginx Ingress 一般有三个组件组成：

* Nginx 反向代理负载均衡器
* Ingress Controller 可以理解为控制器，它通过不断的跟 Kubernetes API 交互，实时获取后端 Service、Pod 等的变化，比如新增、删除等，然后结合 Ingress 定义的规则生成配置，然后动态更新上边的 Nginx 负载均衡器，并刷新使配置生效，来达到服务自动发现的作用。
* Ingress 则是定义规则，通过它定义某个域名的请求过来之后转发到集群中指定的 Service。它可以通过 Yaml 文件定义，可以给一个或多个 Service 定义一个或多个 Ingress 规则。

#### 1.获取官方提供的yaml
```
[root@linux-node1 ~]# cd flask_8s &&  git clone https://github.com/kubernetes/ingress-nginx.git
[root@linux-node1 ~]# tree flask_8s/ingress-nginx/
flask_8s/ingress-nginx/
├── configmap.yaml               :提供configmap可以在线更行nginx的配置
├── default-backend.yaml         :提供一个缺省的后台错误页面 404
├── mandatory.yaml               :这个文件包含了这个目录下面所有yaml的内容，可以不用
├── namespace.yaml               :创建一个独立的命名空间 ingress-nginx
├── rbac.yaml					   :创建对应的role rolebinding 用于rbac
├── tcp-services-configmap.yaml  :修改L4负载均衡配置的configmap
├── udp-services-configmap.yaml  :修改L4负载均衡配置的configmap
└── with-rbac.yaml               :有应用rbac的nginx-ingress-controller组件   

0 directories, 8 files
```

#### 2.修改官方的配置

1. kind: DaemonSet：官方原始文件使用的是deployment，replicate 为 1，这样将会在某一台节点上启动对应的nginx-ingress-controller pod。外部流量访问至该节点，由该节点负载分担至内部的service。测试环境考虑防止单点故障，改为DaemonSet然后删掉replicate ，配合亲和性部署在制定节点上启动nginx-ingress-controller pod，确保有多个节点启动nginx-ingress-controller pod，后续将这些节点加入到外部硬件负载均衡组实现高可用性。
2. hostNetwork: true：添加该字段，暴露nginx-ingress-controller pod的服务端口（80）
3. nodeSelector: 增加亲和性部署，有custom/ingress-controller-ready 标签的节点才会部署该DaemonSet

#### 3.为需要部署nginx-ingress-controller的节点设置lable
```
[root@linux-node1 ~]# kubectl label nodes 192.168.56.12 custom/ingress-controller-ready=true
[root@linux-node1 ~]# kubectl label nodes 192.168.56.13 custom/ingress-controller-ready=true
[root@linux-node1 ~]# kubectl get nodes --show-labels
NAME            STATUS    ROLES     AGE       VERSION   LABELS
192.168.56.12   Ready     <none>    28d       v1.10.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,custom/ingress-controller-ready=true,kubernetes.io/hostname=192.168.56.12
192.168.56.13   Ready     <none>    27d       v1.10.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,custom/ingress-controller-ready=true,kubernetes.io/hostname=192.168.56.13

# 以上，因为我这里就两个计算节点；所以都打上custom/ingress-controller-ready=true这个标签
```

#### 4.执行创建yaml文件:
```
[root@linux-node1 ~]# kubectl create -f namespace.yaml
[root@linux-node1 ~]# kubectl create -f default-backend.yaml
[root@linux-node1 ~]# kubectl create -f configmap.yaml
[root@linux-node1 ~]# kubectl create -f tcp-services-configmap.yaml
[root@linux-node1 ~]# kubectl create -f udp-services-configmap.yaml
[root@linux-node1 ~]# kubectl create -f rbac.yaml
[root@linux-node1 ~]# kubectl create -f with-rbac.yaml
```

#### 5.查看创建状态：

创建过程会去拉镜像比较慢，可能会不成功，前面说过那都是因为`%#@%￥#@·的原因，
所以还是记得给docker一把梯子。

```
[root@linux-node1 ~]# kubectl get pod,svc,deployment,rc -o wide   -n ingress-nginx
NAME                                       READY     STATUS    RESTARTS   AGE       IP              NODE
pod/default-http-backend-5c6d95c48-dwl56   1/1       Running   0          16h       10.2.42.130     192.168.56.13
pod/nginx-ingress-controller-55trv         1/1       Running   0          15h       192.168.56.12   192.168.56.12
pod/nginx-ingress-controller-58nf4         1/1       Running   0          15h       192.168.56.13   192.168.56.13

NAME                           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE       SELECTOR
service/default-http-backend   ClusterIP   10.1.89.141   <none>        80/TCP    16h       app=default-http-backend

NAME                                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS             IMAGES                                        SELECTOR
deployment.extensions/default-http-backend   1         1         1            1           16h       default-http-backend   gcr.io/google_containers/defaultbackend:1.4   app=default-http-backend


# 以上可以看到nginx-ingress-controller已经成功运行在了这两个打了标签的节点上面
```
此时只是把ingress给搭建好，如果要用得根据实际情况写转发规则了，当前我们的目的是通过它定义某个域名的请求过来之后转发到集群中指定的 Service，即此次部署的 `flask-app-nginx`

#### 6.创建规则：
```
[root@linux-node1 ingress-nginx]# cat > test.ingress.yaml < EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
    namespace: flask-app-extions-stage
spec:
  rules:
  - host: test.flaskapp.ingress
    http:
      paths:
      - path: /
        backend:
          serviceName: flask-app-nginx
          servicePort: 80
EOF
// host: 对应的域名 
// path: url上下文 
// backend:后向转发 到对应的 serviceName: servicePort:

[root@linux-node1 ingress-nginx]# kubectl apply -f test-ingress.yaml
[root@linux-node1 ingress-nginx]# kubectl get ingress -n flask-app-extions-stage
NAME           HOSTS                   ADDRESS   PORTS     AGE
test-ingress   test.flaskapp.ingress             80        15h
[root@linux-node1 ingress-nginx]# kubectl describe ingress/test-ingress -n flask-app-extions-stage
Name:             test-ingress
Namespace:        flask-app-extions-stage
Address:
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host                   Path  Backends
  ----                   ----  --------
  test.flaskapp.ingress
                         /   flask-app-nginx:80 (<none>)
Annotations:
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  15h   nginx-ingress-controller  Ingress flask-app-extions-stage/test-ingress
  Normal  CREATE  15h   nginx-ingress-controller  Ingress flask-app-extions-stage/test-ingress
  Normal  CREATE  15h   nginx-ingress-controller  Ingress flask-app-extions-stage/test-ingress
  Normal  CREATE  15h   nginx-ingress-controller  Ingress flask-app-extions-stage/test-ingress
```
#### 7.测试：
既然是通过域名访问，那这里我就在node1上面直接修改hosts的方式然后访问

```
[root@linux-node1 ~]# echo "192.168.56.12 test.flaskapp.ingress" >> /etc/hosts
[root@linux-node1 ~]# echo "192.168.56.13 test.flaskapp.ingress" >> /etc/hosts

[root@linux-node1 ingress-nginx]# curl -I test.flaskapp.ingress/auth/login
HTTP/1.1 200 OK
Server: nginx/1.13.12
Date: Thu, 28 Jun 2018 02:59:16 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 4030
Connection: keep-alive
Vary: Accept-Encoding
Set-Cookie: session=eyJjc3JmX3Rva2VuIjp7IiBiIjoiTXpGbFlUWmlOelV4WXpabVlqTXlNVFJpWTJVMk1HWTVObVV5TURRd09HSTNPVFV4WXprd01RPT0ifX0.DhXghA.3HHbX3fvShem9KlkINr8jDGwcSc; HttpOnly; Path=/

# 或者指定模拟的域名:

[root@linux-node1 ingress-nginx]# curl -vI http://192.168.56.12/auth/login -H 'host: test.flaskapp.ingress'
* About to connect() to 192.168.56.12 port 80 (#0)
*   Trying 192.168.56.12...
* Connected to 192.168.56.12 (192.168.56.12) port 80 (#0)
> HEAD /auth/login HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host: test.flaskapp.ingress
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Server: nginx/1.13.12
Server: nginx/1.13.12
< Date: Thu, 28 Jun 2018 03:00:21 GMT
Date: Thu, 28 Jun 2018 03:00:21 GMT
< Content-Type: text/html; charset=utf-8
Content-Type: text/html; charset=utf-8
< Content-Length: 4030
Content-Length: 4030
< Connection: keep-alive
Connection: keep-alive
< Vary: Accept-Encoding
Vary: Accept-Encoding
< Set-Cookie: session=eyJjc3JmX3Rva2VuIjp7IiBiIjoiT1RNMVpUWTBZV1V6TkdWaU5EazBZMkl5TmpZMFl6VTNPVFF3TVdaa09UVXpNall3T1RRNE1RPT0ifX0.DhXgxQ.BnjwR-swXvmD-kUGhtlvhpHuUIY; HttpOnly; Path=/
Set-Cookie: session=eyJjc3JmX3Rva2VuIjp7IiBiIjoiT1RNMVpUWTBZV1V6TkdWaU5EazBZMkl5TmpZMFl6VTNPVFF3TVdaa09UVXpNall3T1RRNE1RPT0ifX0.DhXgxQ.BnjwR-swXvmD-kUGhtlvhpHuUIY; HttpOnly; Path=/

<
* Connection #0 to host 192.168.56.12 left intact
```
在外部如果要访问也需要绑定域名到node节点，然后访问
![](https://github.com/guomaoqiu/flask_kubernetes/blob/master/screenshots/15301578467095.jpg?raw=true)


ok至此该项目就部署的差不多啦；
那pod如何做到伸缩呢；执行以下命令就行了

```
# 目前我的flask-app是运行了3个，怼20个flask-app应用
[root@linux-node1 ~]# kubectl get deploy/flask-app -n flask-app-extions-stage
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
flask-app   3         3         3            3           23h
[root@linux-node1 ~]# kubectl scale --replicas=20 deployment.extensions/flask-app -n flask-app-extions-stage
deployment.extensions "flask-app" scaled
[root@linux-node1 ~]# kubectl get deploy/flask-app -n flask-app-extions-stage
NAME        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
flask-app   20        20        20           20          23h
# 所以副本数就是靠deploment中的 replicas 或者 命令行的scale --replicas来控制

```

-----

### 部署总结：


坑:
在上面我部署好ingress-nginx后，通过访问哪一步报错了；于是去查了 pod/nginx-ingress-controller-58nf4 的日志，错误日志刷屏啊
![](https://github.com/guomaoqiu/flask_kubernetes/blob/master/screenshots/15301582604316.jpg?raw=true)

官方文档不完整啊，少了创建ingress-services的内容；
解决办法：

```
[root@linux-node1 ingress-nginx]# cat > ingress-nginx-services.yaml < EOF
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  type: NodePort
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
  selector:
    app: ingress-nginx
EOF
[root@linux-node1 ingress-nginx]# kubectl create -f ingress-nginx-services.yaml
```

部署遇到的主要知识点：

* K8S deployment的yaml文件编写；
* K8S 持久化存储NFS方式；配置管理ConfigMap;
* K8S 集群中各种端口/IP类型的工作模式以及服务暴露方式；




