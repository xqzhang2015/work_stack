<!-- MarkdownTOC -->

- [Cluster concepts](#cluster-concepts)
    - [Namespaces](#namespaces)
- [集群](#%E9%9B%86%E7%BE%A4)
    - [Replication Controller](#replication-controller)
    - [Service](#service)
    - [API server:](#api-server)
    - [kube-proxy](#kube-proxy)
      - [kube-proxy mode: iptables](#kube-proxy-mode-iptables)
    - [Node](#node)
    - [Kubernetes Master](#kubernetes-master)
    - [k8s Pod](#k8s-pod)
      - [What containers share in a pod?](#what-containers-share-in-a-pod)
    - [readiness and liveness](#readiness-and-liveness)
      - [Containers startup order](#containers-startup-order)
- [k8s container/pod/service dependencies](#k8s-containerpodservice-dependencies)
  - [Inspecting Dependencies in an Application](#inspecting-dependencies-in-an-application)
  - [Inspecting Dependencies in pod entry.sh](#inspecting-dependencies-in-pod-entrysh)
  - [Using init containers](#using-init-containers)
    - [Base: kubenetes pod life](#base-kubenetes-pod-life)
- [k8s trouble-shooting](#k8s-trouble-shooting)
    - [kubectl describe](#kubectl-describe)
- [References](#references)

<!-- /MarkdownTOC -->

# Cluster concepts
### Namespaces
1. Who is in a namespace scope?

* deployment
* pods
* services
* configmap


2. who is global scope?

* node ports
* node IP
* helm release

3. helm package: deployment vs configmap
In a package(or release), maybe there are multiple deployments for one configmap.


# 集群

集群是一组节点，这些节点可以是物理服务器或者虚拟机，之上安装了Kubernetes平台。下图展示这样的集群。注意该图为了强调核心概念有所简化。这里可以看到一个典型的Kubernetes架构图。

![kubernetes archtecture](../images/2018/k8s-1.png)

上图可以看到如下组件，使用特别的图标表示Service和Label：
* Pod
* Container（容器）
* Label(label)（标签）
* Replication Controller（复制控制器）
* Service（enter image description here）（服务）
* Node（节点）
* Kubernetes Master（Kubernetes主节点）

### Replication Controller
![kubernetes archtecture](../images/2018/k8s-2.gif)
当创建Replication Controller时，需要指定两个东西：
* Pod模板：用来创建Pod副本的模板
* Label：Replication Controller需要监控的Pod的标签。

### Service
如果Pods是短暂的，那么重启时IP地址可能会改变，怎么才能从前端容器正确可靠地指向后台容器呢？

Service是定义一系列Pod以及访问这些Pod的策略的一层抽象。Service通过Label找到Pod组。因为Service是抽象的，所以在图表里通常看不到它们的存在，这也就让这一概念更难以理解。

现在，假定有2个后台Pod，并且定义后台Service的名称为‘backend-service’，lable选择器为（tier=backend, app=myapp）。backend-service 的Service会完成如下两件重要的事情：

* 会为Service创建一个本地集群的DNS入口，因此前端Pod只需要DNS查找主机名为 ‘backend-service’，就能够解析出前端应用程序可用的IP地址。
* 现在前端已经得到了后台服务的IP地址，但是它应该访问2个后台Pod的哪一个呢？Service在这2个后台Pod之间提供透明的负载均衡，会将请求分发给其中的任意一个（如下面的动画所示）。通过每个Node上运行的代理（kube-proxy）完成。这里有更多技术细节。

下述动画展示了Service的功能。注意该图作了很多简化。如果不进入网络配置，那么达到透明的负载均衡目标所涉及的底层网络和路由相对先进。如果有兴趣，这里有更深入的介绍

![kubernetes archtecture](../images/2018/k8s-3.gif)

有一个特别类型的Kubernetes Service，称为'LoadBalancer'，作为外部负载均衡器使用，在一定数量的Pod之间均衡流量。比如，对于负载均衡Web流量很有用。

### API server: 

### [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)

`kube-proxy [flags]`, core args:

* `--cluster-cidr=100.96.0.0/11`
* `--proxy-mode ProxyMode` Which proxy mode to use: `userspace` (older) or `iptables` (faster) or `ipvs` (experimental).
* `--conntrack-max-per-core int32` Default: 32768. Maximum number of NAT connections to track per CPU core.
* `--master string` The address of the Kubernetes API server (overrides any value in kubeconfig)
* `--oom-score-adj int32` Default: -999. The oom-score-adj value for kube-proxy process. Values must be within the range [-1000, 1000]

e.g. `kubectl -n kube-system get pod/kube-proxy-ip-10-206-145-25.ec2.internal -o yaml`
```
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - mkfifo /tmp/pipe; (tee -a /var/log/kube-proxy.log < /tmp/pipe & ) ; exec /usr/local/bin/kube-proxy
      --cluster-cidr=100.96.0.0/11 --conntrack-max-per-core=131072 --hostname-override=ip-10-206-145-25.ec2.internal
      --kubeconfig=/var/lib/kube-proxy/kubeconfig --master=https://api.internal.adsk8s.replay.ads.aws.fwmrm.net
      --oom-score-adj=-998 --resource-container="" --v=2 > /tmp/pipe 2>&1
    image: k8s.gcr.io/kube-proxy:v1.10.6
    imagePullPolicy: IfNotPresent
    name: kube-proxy
```

#### kube-proxy mode: iptables



e.g. `sudo iptables -nvL --line-numbers -t nat`


* DNAT rule is physically configured in Chain __PREROUTING__ and __OUTPUT__.

```sh
Chain PREROUTING (policy ACCEPT 967 packets, 78624 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     848K   69M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain INPUT (policy ACCEPT 401 packets, 44664 bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 545 packets, 57994 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     455K   48M KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */

Chain POSTROUTING (policy ACCEPT 1116 packets, 92482 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     958K   79M KUBE-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
2     501K   30M RETURN     all  --  *      *       100.64.0.0/10        100.64.0.0/10
3      466 27960 MASQUERADE  all  --  *      *       100.64.0.0/10       !224.0.0.0/4
4        0     0 RETURN     all  --  *      *      !100.64.0.0/10        100.100.44.0/24
5        0     0 MASQUERADE  all  --  *      *      !100.64.0.0/10        100.64.0.0/10
```

* Chain KUBE-MARK-MASQ  (71 references)
```sh
num   pkts bytes target     prot opt in     out     source               destination
1        6   360 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            MARK or 0x4000
```

* Chain KUBE-SERVICES (2 references)
```sh
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 KUBE-MARK-MASQ  tcp  --  *      *      !100.96.0.0/11        100.64.0.1           /* default/kubernetes:https cluster IP */ tcp dpt:443
2        0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            100.64.0.1           /* default/kubernetes:https cluster IP */ tcp dpt:443
3        0     0 KUBE-MARK-MASQ  tcp  --  *      *      !100.96.0.0/11        100.70.48.69         /* monitoring/prometheus-service-test: cluster IP */ tcp dpt:9090
4        0     0 KUBE-SVC-QPT2TG7II3OPDVX2  tcp  --  *      *       0.0.0.0/0            100.70.48.69         /* monitoring/prometheus-service-test: cluster IP */ tcp dpt:9090

...

15       0     0 KUBE-MARK-MASQ  tcp  --  *      *      !100.96.0.0/11        100.69.14.216        /* test-timer/test-aerospike:access cluster IP */ tcp dpt:3000
16       0     0 KUBE-SVC-6AREQSOXFQ7P4D6N  tcp  --  *      *       0.0.0.0/0            100.69.14.216        /* test-timer/test-aerospike:access cluster IP */ tcp dpt:3000
```

* Chain KUBE-SVC-6AREQSOXFQ7P4D6N (2 references): __1 pod in the aerospike service__
```sh
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 KUBE-SEP-674TPCA5HHRYAKOV  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-timer/test-aerospike:access */
```
==> __2 pods in the aeropsike service__: rr(round robin)

```
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 KUBE-SEP-NHWO3IZRB7L4XFBG  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-timer/test-aerospike:access */ statistic mode random probability 0.50000000000
2        0     0 KUBE-SEP-7PGDGKTUS3BMZPYH  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-timer/test-aerospike:access */
```

* Chain KUBE-SEP-674TPCA5HHRYAKOV (1 references)
```sh
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 KUBE-MARK-MASQ  all  --  *      *       100.100.41.4         0.0.0.0/0            /* test-timer/test-aerospike:access */
2        0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* test-timer/test-aerospike:access */ tcp to:100.100.41.4:3000
```

### Node

节点（上图橘色方框）是物理或者虚拟机器，作为Kubernetes worker，通常称为Minion。每个节点都运行如下Kubernetes关键组件：

* Kubelet：是主节点代理。
* Kube-proxy：Service使用其将链接路由到Pod，如上文所述。
* Docker或Rocket：Kubernetes使用的容器技术来创建容器。

### Kubernetes Master

集群拥有一个Kubernetes Master（紫色方框）。Kubernetes Master提供集群的独特视角，并且拥有一系列组件，比如Kubernetes API Server。

API Server提供可以用来和集群交互的REST端点。master节点包括用来创建和复制Pod的Replication Controller。


### k8s Pod

#### What containers share in a pod?
Containers in a Pod run on a “logical host”. They use

* the same network namespace (in other words, the same IP address and port space),
* the same IPC namespace. 
* They can also use shared volumes. 

### readiness and liveness

* readiness: if pod is ready to accept traffic


* liveness: if pod is healthy and needs to restart

* example
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

#### Containers startup order

In a cloud native environment, it’s always better to plan for failures outside of your immediate control. 

__To change the application to `wait` for another container in this pod__.

# k8s container/pod/service dependencies

Refences: [Kubernetes: Solving Service Dependencies](https://www.alibabacloud.com/blog/kubernetes-demystified-solving-service-dependencies_594110)

## Inspecting Dependencies in an Application

* Example: A dependency between a Golang application and MySQL:

```golang
// ...
// Connect to database.
hostPort := net.JoinHostPort(config.Host, config.Port)
log.Println("Connecting to database at", hostPort)
dsn := fmt.Sprintf("%s:%s@tcp(%s)/%s?timeout=30s",
    config.Username, config.Password, hostPort, config.Database)

db, err = sql.Open("mysql", dsn)
if err != nil {
    log.Println(err)
}

var dbError error
maxAttempts := 20
for attempts := 1; attempts <= maxAttempts; attempts++ {
    dbError = db.Ping()
    if dbError == nil {
        break
    }
    log.Println(dbError)
    time.Sleep(time.Duration(attempts) * time.Second)
}
if dbError != nil {
    log.Fatal(dbError)
}

log.Println("Application started successfully.")
// ...
```
## Inspecting Dependencies in pod entry.sh
1. self-writing by util ...; do ...; done

```sh
until curl localhost:3306 2>/dev/null
do
  echo "[`date`] waiting for mysql"
  sleep 10
done
echo "[`date`] mysql is ready"
```

2. using tool such as [wait-for-it](https://github.com/vishnubob/wait-for-it)

## Using init containers

* Example: initContainers
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:4
        ports:
        - containerPort: 80
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql
        - name: WORDPRESS_DB_PASSWORD
          value: ""
      initContainers:
      - name: init-mysql
        image: busybox
        command: ['sh', '-c', 'until nslookup mysql; do echo waiting for mysql; sleep 2; done;']
```

### Base: kubenetes pod life

![kubenetes pod life](../images/2019/k8s_pod_life.png)

Pods contain three types of containers:

* Infra container: This is the famous `pause` container.
* Init container: This is an [initialization container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/?spm=a2c65.11461447.0.0.3b554f4eyEesyQ), generally used to initialize and prepare applications. The application container can start up only after waiting for all initialization containers to finish running.
* Main container: This is an application container.


# k8s trouble-shooting

### kubectl describe

There are three ways to determine when using kubectl:
* run `kubectl describe pod <EVICTED_POD>` to find out the reason why it was evicted in the events.
* run `kubectl describe node <THE_NODE_WHERE_THE_POD_WAS_EVICTED>` to check Conditions. 
* run `kubectl get events`.


# References
[十分钟带你理解Kubernetes核心概念](http://www.dockone.io/article/932)<br/>

[《Kubernetes权威指南》——Kubelet运行机制与安全机制](https://www.cnblogs.com/suolu/p/6841848.html)<br/>

[闲谈Kubernetes 的主要特性和经验分享](https://www.oschina.net/question/2432428_246729)<br/>

[Interactive Tutorial - Creating a Cluster](https://kubernetes.io/docs/tutorials/kubernetes-basics/create-cluster/cluster-interactive/)<br/>

[Tutorials: hello minikube](https://kubernetes.io/docs/tutorials/hello-minikube/)<br/>

[]()<br/>

[]()<br/>
