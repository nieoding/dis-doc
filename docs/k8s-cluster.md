> 单点集群指 1个master + n个node，一般开发环境可以这样配置

## 初始化
k8s的节点名称显示都是使用节点的主机名，所以所有的物理节点都需要按照集群架构设置合理的主机名

```
hostnamectl set-hostname master
```

`kubeadm init`是初始化指令，目前的版本，k8s初始化的时候，第一件事是通知docker拉镜像，时间会比较长，命令行也不会有提示，官方给的建议，先执行如下指令预加载镜像

```
kubeadm config images pull
```

官方推荐使用flannel做为集群内部网络模式，设置cidr为10.244.0.0/16，k8s将使用flannel模式初始化集群

```
kubeadm init --pod-network-cidr=10.244.0.0/16
```
> 奇淫技巧：国内镜像加速

```
cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1alpha3
kind: ClusterConfiguration
kubernetesVersion: stable
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
networking:
    podSubnet: "10.244.0.0/16"
EOF
kubeadm config images pull --config kubeadm-config.yaml
kubeadm init --config kubeadm-config.yaml
```

在初始化过程中如果出现错误，必须执行`kubeadm reset` 重置才能再次初始化，安装成功以后会出现如下消息提示

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.0.142:6443 --token yu6t36.bitufx18dfy10od6 --discovery-token-ca-cert-hash sha256:bb247916192cd029421de621d7af3fee0894f96b77f216b8317f25d2c319ed52
```
提示里面的信息很重要！

其中`kubeadm join` 是工作节点加入集群的指令，第一次创建集群需要把这行代码复制下来，后面需要使用到，当然万一丢失了也没关系，后面会介绍补救方法。

我们需要将k8s系统生产的admin.conf拷贝到`~/.kube/config`，kubectl才能正常工作，否则kubectl执行的错误如下：

```bash
[root@master ~]# kubectl get nodes
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
正常的执行结果如下：

```bash
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES    AGE   VERSION
master   NotReady   master   12m   v1.12.2
```


接下来需要安装flannel组件，这样master节点才能变成Ready状态

## 安装flannel
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
执行正常屏幕会输出

```bash
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```

等一会使用`kubectl`可以看到master已经处于Ready状态

```bash
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   6m29s   v1.12.2
```

## 工作节点加入集群
使用镜像创建工作节点，记得将节点改个名字

```
hostnamectl set-hostname worker
```

使用上面master初始化成功屏幕输出的`kubeadm join...`指令即可加入集群

```
kubeadm join 192.168.0.142:6443 --token yu6t36.bitufx18dfy10od6 --discovery-token-ca-cert-hash sha256:bb247916192cd029421de621d7af3fee0894f96b77f216b8317f25d2c319ed52
```

此步一般不会出错，屏幕会出如下提示

```bash
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```
我们回到master上去看看节点状态

```
[root@master ~]# kubectl get nodes
NAME     STATUS     ROLES    AGE     VERSION
master   Ready      master   17m     v1.12.2
worker   NotReady   <none>   3m30s   v1.12.2
```

屏幕显示节点已经正常加入了，但是状态是NotReady，此时master会通知节点创建2个pod(kube-flannel, kube-proxy)，我们在master上可以通过`kubectl`观测pod的创建进度

```
[root@master ~]# kubectl get pods -n kube-system -o wide |grep worker
kube-flannel-ds-amd64-pd9xv      1/1     Running   0          16m   192.168.0.144   worker   <none>
kube-proxy-b4pgl                 1/1     Running   0          16m   192.168.0.144   worker   <none>
```
pod处于running以后，工作节点状态就会变成Ready了

```
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   20m     v1.12.2
worker   Ready    <none>   7m11s   v1.12.2
```

- 如果kubeadm join需要的token参数过期或者遗失怎么办：

> 在master上执行

```
[root@master ~]# kubeadm token create
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION   EXTRA GROUPS
qbyhi6.p8zg6r3eir4xqqmn   23h       2018-11-16T16:37:19+08:00   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
[root@master ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
dfc5b832302e76b277ca2bf79ba86d43d5337d15b3abdaf90be052ad58f0f2a9
```
拼接出来的join指令如下

```
kubeadm join --token qbyhi6.p8zg6r3eir4xqqmn --discovery-token-ca-cert-hash sha256:dfc5b832302e76b277ca2bf79ba86d43d5337d15b3abdaf90be052ad58f0f2a9  ip:6443
```