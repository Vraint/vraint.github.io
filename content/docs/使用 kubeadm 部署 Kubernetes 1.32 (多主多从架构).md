---
type: docs
prev: false
next: false
---

## 环境准备

### 操作系统
- **操作系统**: Ubuntu 22.04
- **节点配置**:
  - 主节点 (Master Nodes): 3 台
  - 工作节点 (Worker Nodes): N 台

### 容器运行时
- **容器运行时**: Containerd

### 网络插件
- **网络插件**: Flannel

### Kubernetes 版本
- **版本**: 1.32
- **软件包仓库**: `pkgs.k8s.io`

## 步骤

### 1. 更新系统并安装依赖

在所有节点上执行以下命令：

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y apt-transport-https ca-certificates curl
```

### 2. 配置 Linux 内核

#### 2.1 启用必要的内核模块

确保以下内核模块已加载：

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

为了确保这些模块在系统重启后仍然加载，编辑 `/etc/modules-load.d/k8s.conf` 文件：

```bash
echo "overlay" | sudo tee -a /etc/modules-load.d/k8s.conf
echo "br_netfilter" | sudo tee -a /etc/modules-load.d/k8s.conf
```

#### 2.2 设置内核参数

编辑 `/etc/sysctl.d/kubernetes.conf` 文件，添加以下内容：

```bash
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

应用新的内核参数：

```bash
sudo sysctl --system
```

### 3. 配置 Containerd

在所有节点上安装并配置 Containerd：

```bash
# 安装 containerd
sudo apt-get install -y containerd

# 配置 containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# 修改配置文件以使用 systemd cgroups
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# 重启 containerd
sudo systemctl restart containerd
```

### 4. 添加 Kubernetes 软件包仓库

在所有节点上添加新的 Kubernetes 软件包仓库：

```bash
# 添加 GPG 密钥
curl -fsSL https://pkgs.k8s.io/core:/stable:/1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

# 添加软件包仓库
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/1.32/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 更新软件包索引
sudo apt-get update
```

### 5. 安装 kubeadm、kubelet 和 kubectl

在所有节点上安装 kubeadm、kubelet 和 kubectl：

```bash
sudo apt-get install -y kubeadm=1.32.0-00 kubelet=1.32.0-00 kubectl=1.32.0-00
```

确保锁定版本以防止意外升级：

```bash
sudo apt-mark hold kubeadm kubelet kubectl
```

### 6. 配置负载均衡器

在多主节点架构中，你需要一个负载均衡器来分发流量到多个主节点。可以使用硬件负载均衡器或软件负载均衡器（如 HAProxy 或 MetalLB）。确保负载均衡器配置正确，并且所有主节点都能通过负载均衡器的 IP 地址访问。

### 7. 初始化主节点

在第一个主节点上执行 `kubeadm init` 命令。以下是详细的命令和选项说明：

```bash
sudo kubeadm init \
  --control-plane-endpoint="LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=NODE_IP \
  --image-repository=registry.k8s.io \
  --kubernetes-version=v1.32.0
```

#### 选项说明

- `--control-plane-endpoint="LOAD_BALANCER_DNS:LOAD_BALANCER_PORT"`: 指定负载均衡器的 DNS 名称和端口。所有主节点和工作节点将通过这个地址访问 API 服务器。
- `--upload-certs`: 将证书上传到集群，以便其他主节点可以使用这些证书加入集群。
- `--pod-network-cidr=10.244.0.0/16`: 指定 Pod 网络的 CIDR 范围。Flannel 默认使用 `10.244.0.0/16`。
- `--apiserver-advertise-address=NODE_IP`: 指定主节点的 IP 地址，API 服务器将使用这个地址进行通信。
- `--image-repository=registry.k8s.io`: 指定镜像仓库，确保使用官方的 Kubernetes 镜像。
- `--kubernetes-version=v1.32.0`: 指定要安装的 Kubernetes 版本。

### 8. 配置 kubectl

初始化完成后，配置 `kubectl` 以便你可以与集群进行交互：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 9. 安装 Flannel 网络插件

安装 Flannel 网络插件以确保 Pod 之间的网络通信：

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### 10. 加入其他主节点

在其他主节点上执行以下命令加入集群：

```bash
sudo kubeadm join LOAD_BALANCER_DNS:LOAD_BALANCER_PORT \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>
```

- `<token>` 和 `<hash>` 可以从第一个主节点的 `kubeadm init` 输出中获取。
- `<cert-key>` 是在第一个主节点上执行 `kubeadm init` 时生成的证书密钥。

### 11. 加入工作节点

在工作节点上执行以下命令加入集群：

```bash
sudo kubeadm join LOAD_BALANCER_DNS:LOAD_BALANCER_PORT \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

- `<token>` 和 `<hash>` 同样可以从第一个主节点的 `kubeadm init` 输出中获取。

### 12. 验证集群状态

在主节点上验证集群状态：

```bash
kubectl get nodes
```

确保所有节点的状态为 `Ready`。

## 注意事项

- **时间同步**：确保所有节点的时间同步（NTP），以避免证书验证问题。
- **防火墙规则**：确保防火墙规则允许必要的端口通信，例如：
  - API 服务器端口（默认 6443）
  - 容器运行时端口（例如 Containerd 的 10250）
  - Flannel 网络端口（例如 8285 和 8472）
- **主机名和 IP 地址**：确保所有节点的主机名和 IP 地址配置正确，并且可以通过 DNS 或 `/etc/hosts` 文件解析。

[^1]: 脚注测试
