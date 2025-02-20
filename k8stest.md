# k8s

## 部署k8s

**Kubernetes API 服务器**以及整个 **Kubernetes 集群**完全可以在单台服务器上进行配置和运行，这种方式通常用于开发、测试或学习 Kubernetes 的基本概念。通常，我们将这种部署方式称为 **单节点集群** 或 **单机集群**。

1. 执行`sudo kubeadm init --pod-network-cidr=192.168.0.0/16` 
其中，`--pod-network-cidr=192.168.0.0/16` 是给 Pod 网络分配的地址范围，具体的值取决于你使用的网络插件。例如，如果使用的是 **Calico**，可以保持默认值。
初始化完成后，需要将 kubeconfig 配置文件复制到的用户目录，这样 `kubectl` 就能连接到集群：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

1. 安装网络插件以支持pod互相通信（Calio或Flannel）

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

1. 检查集群状态，status必须为ready，如果notready可以`sudo systemctl restart kubelet`

```bash
(base) hjy@debian:~$ kubectl get nodes
NAME     STATUS     ROLES           AGE     VERSION
debian   Ready   control-plane   5m26s   v1.29.11
```

如果有其他节点加入集群，可以使用下列命令。由于此时为单节点集群，无需操作。

```bash
kubeadm join --token <token> <master-ip>:<port> --discovery-token-ca-cert-hash sha256:<hash>
```

## K8S API服务器启动

ApiServer : 资源操作的唯一入口，接收用户输入的命令，提供认证、授权、API注册和发现等机制
启动 Kubernetes API 服务器（`kube-apiserver`）通常是在 Kubernetes 集群初始化时由 Kubernetes 控制平面管理的。在大多数生产环境中，API 服务器会通过 Kubernetes 集群的控制平面自动启动和管理。如果你正在手动管理一个 Kubernetes 集群，或者在某些开发环境中工作，可能需要手动启动 Kubernetes API 服务器。

1. `sudo kubeadm init --pod-network-cidr=192.168.0.0/16` 自动启动了K8S API服务器，可以查看服务器状态，并能够看见`kube-apiserver` 相关的 Pod

```bash
kubectl get pods -n kube-system

NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-658d97c59c-d5htd   1/1     Running   0          13m
calico-node-hd5jm                          1/1     Running   0          13m
coredns-76f75df574-2mbvl                   1/1     Running   0          18m
coredns-76f75df574-nzljv                   1/1     Running   0          18m
etcd-debian                                1/1     Running   1          18m
kube-apiserver-debian                      1/1     Running   1          18m
kube-controller-manager-debian             1/1     Running   1          18m
kube-proxy-djns7                           1/1     Running   0          18m
kube-scheduler-debian                      1/1     Running   1          18m
```

1. 如果你的 `kubectl` 配置文件中错误地将 API 服务器指向 `localhost:8080`，你需要更新配置文件，使其指向正确的 Kubernetes API 服务器地址.

```bash
#查看当前配置
kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://10.19.61.125:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
#修改API服务器地址设置
kubectl config set-cluster my-cluster --server=https://<your-kubernetes-api-server>:6443
```

- 如果你确实需要将 Kubernetes API 服务器的端口号修改为其他端口，可以在 Kubernetes 配置中修改 `--port` 或 `--secure-port` 参数，但这种修改通常需要集群管理员权限。更常见的做法是修改客户端 `kubectl` 配置文件，指向正确的 API 服务器地址和端口。
- 如果 Kubernetes 集群部署在远程机器上，确保防火墙没有阻止访问 Kubernetes API 端口（通常是 6443 或其他配置的端口）
- 手动安装（没实操，用于调试不建议）
    
    `kube-apiserver` 是 Kubernetes 控制平面的一部分，通常与 `kube-controller-manager` 和 `kube-scheduler` 一起运行。
    
    - `-advertise-address` 是 Kubernetes API 服务器的公网 IP 地址，通常是控制平面节点的 IP。
    - `-secure-port=6443` 用于 HTTPS 通信，通常使用 6443 端口。
    - `-insecure-port=8080` 用于 HTTP 通信，一般不推荐开放。
    - `-etcd-servers` 用于指定 Etcd 集群的地址。
    
    ```bash
    #启动APIserver
    kube-apiserver \
    --advertise-address=<API_SERVER_IP> \
    --bind-address=0.0.0.0 \
    --insecure-port=8080 \
    --secure-port=6443 \
    --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
    --authorization-mode=Node,RBAC \
    --enable-bootstrap-token-auth \
    --service-cluster-ip-range=<Service_IP_Range> \
    --etcd-servers=http://<etcd-server-ip>:2379
    
    #启动APIserver作为后台进程
    nohup kube-apiserver \
      --advertise-address=192.168.1.100 \
      --bind-address=0.0.0.0 \
      --secure-port=6443 \
      --etcd-servers=http://127.0.0.1:2379 \
      --authorization-mode=Node,RBAC \
      --service-cluster-ip-range=10.96.0.0/12 \
      --allow-privileged=true &
    ```
    

1. 检查API server状态

```bash
kubectl get componentstatuses
#正常启动
NAME                 STATUS    MESSAGE   ERROR
scheduler            Healthy   ok        
controller-manager   Healthy   ok        
etcd-0               Healthy   ok       
```

总流程为：

containerd能正常运行→kubeadm init→创建kubelet