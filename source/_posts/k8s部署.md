# k8s部署

## 节点配置

   > 192.168.1.6 k8s-master

   > 192.168.1.8 k8s-node1

   > 192.168.1.9 k8s-node2

- 处理器、内存、磁盘配置

   CPU：2核

   内存：4G

   硬盘：50G

## 修改系统设置

在`Ubuntu 22.04`安装完毕后，我们需要做以下检查和操作：

- 检查网络

   在每个节点安装成功后，需要通过`ping`命令检查以下几项：

   > 1.是否能够`ping`通`baidu.com`；

   > 2.是否能够`ping`通宿主机；

   > 3.是否能够`ping`通子网内其他节点；

- 检查时区

   时区不正确的可以通过下面的命令来修正：

```other
sudo tzselect
```

   根据系统提示进行选择即可；

- 配置ubuntu系统国内源

   因为我们需要在`ubuntu 22.04`系统上安装`k8s`，为了避免遭遇科学上网的问题，我们需要配置一下国内的源；

   - 备份默认源：

```other
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo rm -rf /etc/apt/sources.list
```

   - 配置国内源：

```other
sudo vi /etc/apt/sources.list
```

      内容如下：

```other
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
```

   - 更新

      修改完毕后，需要执行以下命令生效

```sql
sudo apt-get update
sudo apt-get upgrade
```

- **禁用** selinux

   默认ubuntu下没有这个模块，centos下需要禁用selinux；

- 禁用swap

   临时禁用命令：

```css
sudo swapoff -a
```

   永久禁用：

```other
sudo vi /etc/fstab
```

   将最后一行注释后重启系统即生效：

```shell
#/swap.img      none    swap    sw      0       0
```

- 修改内核参数：

```other
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

```other
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

   运行以下命令使得上述配置生效：

```css
sudo sysctl --system
```

# 修改containerd配置

修改这个配置是关键，否则会因为科学上网的问题导致k8s安装出错；比如后续`kubeadm init`失败，`kubeadm join`后节点状态一直处于`NotReady`状态等问题；

- 备份默认配置

```other
sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
```

- 修改配置

```other
sudo vi /etc/containerd/config.toml
```

   配置内容如下：

```other
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
temp = ""
version = 2

[cgroup]
  path = ""

[debug]
  address = ""
  format = ""
  gid = 0
  level = ""
  uid = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
  tcp_address = ""
  tcp_tls_ca = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"

  [plugins."io.containerd.grpc.v1.cri"]
    device_ownership_from_security_context = false
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    enable_selinux = false
    enable_tls_streaming = false
    enable_unprivileged_icmp = false
    enable_unprivileged_ports = false
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
      ip_pref = ""
      max_conf_num = 1

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      ignore_rdt_not_enabled_errors = false
      no_pivot = false
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          cni_conf_dir = ""
          cni_max_conf_num = 0
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_path = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true

      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]

    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = "node"

    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.internal.v1.tracing"]
    sampling_ratio = 1.0
    service_name = "containerd"

  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"

  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false

  [plugins."io.containerd.runtime.v1.linux"]
    no_shim = false
    runtime = "runc"
    runtime_root = ""
    shim = "containerd-shim"
    shim_debug = false

  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
    sched_core = false

  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]

  [plugins."io.containerd.service.v1.tasks-service"]
    rdt_config_file = ""

  [plugins."io.containerd.snapshotter.v1.aufs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.btrfs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.devmapper"]
    async_remove = false
    base_image_size = ""
    discard_blocks = false
    fs_options = ""
    fs_type = ""
    pool_name = ""
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.native"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.overlayfs"]
    root_path = ""
    upperdir_label = false

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

  [plugins."io.containerd.tracing.processor.v1.otlp"]
    endpoint = ""
    insecure = false
    protocol = ""

[proxy_plugins]

[stream_processors]

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar"

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar+gzip"

[timeouts]
  "io.containerd.timeout.bolt.open" = "0s"
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[ttrpc]
  address = ""
  gid = 0
  uid = 0
 ​
```

- 重启containerd服务

```other
sudo systemctl enable containerd
sudo systemctl daemon-reload && systemctl restart containerd
```

## 安装k8s组件

- 添加k8s的阿里云yum源

```other
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
 ​
sudo apt-add-repository "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
 ​
sudo apt-get update
```

- 安装k8s组件

```other
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

   可以通过`apt-cache madison kubelet`命令查看kubelet组件的版本；其他组件查看也是一样的命令，把对应位置的组件名称替换即可；

## 初始化master

- 生成kubeadm默认配置文件

```other
sudo kubeadm config print init-defaults > kubeadm.yaml
```

   修改默认配置

```other
sudo vi kubeadm.yaml
```

   总共做了四处修改：

   > 1.修改localAPIEndpoint.advertiseAddress为master的ip；

   > 2.修改nodeRegistration.name为当前节点名称；

   > 3.修改imageRepository为国内源：[registry.cn-hangzhou.aliyuncs.com/google_containers](http://registry.cn-hangzhou.aliyuncs.com/google_containers)

   > 4.添加networking.podSubnet，该网络ip范围不能与networking.serviceSubnet冲突，也不能与节点网络192.168.1.0/24相冲突；所以我就设置成10.10.0.0/16；
修改后的内容如下：

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.1.6
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: k8s-master
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: mycluster
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.26.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.0.0.0/16
  podSubnet: 10.10.0.0/16
scheduler: {}
```

- 执行初始化操作

```other
sudo kubeadm init —config kubeadm.yaml
```

   如果在执行init操作中有任何错误，可以使用`journalctl -u kubelet`查看到的错误日志；失败后我们可以通过下面的命令重置，否则再次init会存在端口冲突的问题：

```perl
sudo kubeadm reset
```

   初始化成功后，按照提示执行下面命令：

```other
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "KUBECONFIG=$HOME/.kube/config" >> ~/.bashrc
```

   再切换到`root`用户执行下面命令：

```javascript
export KUBECONFIG=/etc/kubernetes/admin.conf
```

   先不要着急`kubeadm join`其他节点进来，可以切换`root`用户执行以下命令就能看到节点列表：

```other
kubectl get nodes
```

   但是此时的节点状态还是`NotReady`状态，我们接下来需要安装网络插件；

## 给master安装calico网络插件

- 下载kube-flannel.yml配置

```other
curl https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml -O
```

   修改kube-flannel.yml配置

```bash
#将"Network": "10.10.0.0/16",修改为你podnetwork的网段，本文为10.10.0.0/16
```

   安装kube-flannel.yml插件

```other
kubectl apply -f kube-flannel.yml
```

   卸载该插件可以使用下面命令：

```other
kubectl delete -f kube-flannel.yml
```

flannel网络插件安装成功后，master节点的状态将逐渐转变成`Ready`状态；如果状态一直是`NotReady`，建议重启一下master节点；

## 接入两个工作节点

我们在master节点init成功后，会提示可以通过`kubeadm join`命令把工作节点加入进来。我们在master节点安装好calico网络插件后，就可以分别在两个工作节点中执行`kubeadm join`命令了：

```sql
kubeadm join 192.168.1.6:6443 --token ui3x3h.n3op589vk67med1t --discovery-token-ca-cert-hash sha256:ecd2d64f701233c28f78ce3406b0f8beb75acc4bbf7cebc25d862196e95a486f
```

如果我们忘记了master中生成的命令，我们依然可以通过以下命令让master节点重新生成一下`kubeadm join`命令：

```other
sudo kubeadm token create --print-join-command
```

我们在工作节点执行完`kubeadm join`命令后，需要回到master节点执行以下命令检查工作节点是否逐渐转变为`Ready`状态：

```other
kubectl get nodes
```

如果工作节点长时间处于`NotReady`状态，我们需要查看`pods`状态：

```sql
sudo kubectl get pods -n kube-system
```

查看目标`pod`的日志可以使用下面命令：

```sql
kubectl describe pod -n kube-system [pod-name]
```

当所有工作节点都转变成`Ready`状态后，我们就可以安装Dashboard了；

## 安装Dashboard

- 准备配置文件

   可以科学上网的小伙伴可以按照github上的文档来：[github.com/kubernetes/…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fkubernetes%2Fdashboard%2Ftree%2Fv2.7.0)，我选择的是2.7.0版本；

   不能科学上网的小伙伴就按照下面步骤来，在master节点操作：

```other
sudo vi recommended.yaml
```

   文件内容如下：

```yaml
# Copyright 2017 The Kubernetes Authors.
 #
 # Licensed under the Apache License, Version 2.0 (the "License");
 # you may not use this file except in compliance with the License.
 # You may obtain a copy of the License at
 #
 #     http://www.apache.org/licenses/LICENSE-2.0
 #
 # Unless required by applicable law or agreed to in writing, software
 # distributed under the License is distributed on an "AS IS" BASIS,
 # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 # See the License for the specific language governing permissions and
 # limitations under the License.
 ​
 apiVersion: v1
 kind: Namespace
 metadata:
   name: kubernetes-dashboard
 ​
 ---
 ​
 apiVersion: v1
 kind: ServiceAccount
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 ​
 ---
 ​
 kind: Service
 apiVersion: v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 spec:
   type: NodePort
   ports:
     - port: 443
       targetPort: 8443
   selector:
     k8s-app: kubernetes-dashboard
 ​
 ---
 ​
 apiVersion: v1
 kind: Secret
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard-certs
   namespace: kubernetes-dashboard
 type: Opaque
 ​
 ---
 ​
 apiVersion: v1
 kind: Secret
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard-csrf
   namespace: kubernetes-dashboard
 type: Opaque
 data:
   csrf: ""
 ​
 ---
 ​
 apiVersion: v1
 kind: Secret
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard-key-holder
   namespace: kubernetes-dashboard
 type: Opaque
 ​
 ---
 ​
 kind: ConfigMap
 apiVersion: v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard-settings
   namespace: kubernetes-dashboard
 ​
 ---
 ​
 kind: Role
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 rules:
   # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
   - apiGroups: [""]
     resources: ["secrets"]
     resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
     verbs: ["get", "update", "delete"]
     # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
   - apiGroups: [""]
     resources: ["configmaps"]
     resourceNames: ["kubernetes-dashboard-settings"]
     verbs: ["get", "update"]
     # Allow Dashboard to get metrics.
   - apiGroups: [""]
     resources: ["services"]
     resourceNames: ["heapster", "dashboard-metrics-scraper"]
     verbs: ["proxy"]
   - apiGroups: [""]
     resources: ["services/proxy"]
     resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
     verbs: ["get"]
 ​
 ---
 ​
 kind: ClusterRole
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
 rules:
   # Allow Metrics Scraper to get metrics from the Metrics server
   - apiGroups: ["metrics.k8s.io"]
     resources: ["pods", "nodes"]
     verbs: ["get", "list", "watch"]
 ​
 ---
 ​
 apiVersion: rbac.authorization.k8s.io/v1
 kind: RoleBinding
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: Role
   name: kubernetes-dashboard
 subjects:
   - kind: ServiceAccount
     name: kubernetes-dashboard
     namespace: kubernetes-dashboard
 ​
 ---
 ​
 apiVersion: rbac.authorization.k8s.io/v1
 kind: ClusterRoleBinding
 metadata:
   name: kubernetes-dashboard
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: kubernetes-dashboard
 subjects:
   - kind: ServiceAccount
     name: kubernetes-dashboard
     namespace: kubernetes-dashboard
 ​
 ---
 ​
 kind: Deployment
 apiVersion: apps/v1
 metadata:
   labels:
     k8s-app: kubernetes-dashboard
   name: kubernetes-dashboard
   namespace: kubernetes-dashboard
 spec:
   replicas: 1
   revisionHistoryLimit: 10
   selector:
     matchLabels:
       k8s-app: kubernetes-dashboard
   template:
     metadata:
       labels:
         k8s-app: kubernetes-dashboard
     spec:
       securityContext:
         seccompProfile:
           type: RuntimeDefault
       containers:
         - name: kubernetes-dashboard
           image: kubernetesui/dashboard:v2.6.1
           imagePullPolicy: Always
           ports:
             - containerPort: 8443
               protocol: TCP
           args:
             - --auto-generate-certificates
             - --namespace=kubernetes-dashboard
             # Uncomment the following line to manually specify Kubernetes API server Host
             # If not specified, Dashboard will attempt to auto discover the API server and connect
             # to it. Uncomment only if the default does not work.
             # - --apiserver-host=http://my-address:port
           volumeMounts:
             - name: kubernetes-dashboard-certs
               mountPath: /certs
               # Create on-disk volume to store exec logs
             - mountPath: /tmp
               name: tmp-volume
           livenessProbe:
             httpGet:
               scheme: HTTPS
               path: /
               port: 8443
             initialDelaySeconds: 30
             timeoutSeconds: 30
           securityContext:
             allowPrivilegeEscalation: false
             readOnlyRootFilesystem: true
             runAsUser: 1001
             runAsGroup: 2001
       volumes:
         - name: kubernetes-dashboard-certs
           secret:
             secretName: kubernetes-dashboard-certs
         - name: tmp-volume
           emptyDir: {}
       serviceAccountName: kubernetes-dashboard
       nodeSelector:
         "kubernetes.io/os": linux
       # Comment the following tolerations if Dashboard must not be deployed on master
       tolerations:
         - key: node-role.kubernetes.io/master
           effect: NoSchedule
 ​
 ---
 ​
 kind: Service
 apiVersion: v1
 metadata:
   labels:
     k8s-app: dashboard-metrics-scraper
   name: dashboard-metrics-scraper
   namespace: kubernetes-dashboard
 spec:
   ports:
     - port: 8000
       targetPort: 8000
   selector:
     k8s-app: dashboard-metrics-scraper
 ​
 ---
 ​
 kind: Deployment
 apiVersion: apps/v1
 metadata:
   labels:
     k8s-app: dashboard-metrics-scraper
   name: dashboard-metrics-scraper
   namespace: kubernetes-dashboard
 spec:
   replicas: 1
   revisionHistoryLimit: 10
   selector:
     matchLabels:
       k8s-app: dashboard-metrics-scraper
   template:
     metadata:
       labels:
         k8s-app: dashboard-metrics-scraper
     spec:
       securityContext:
         seccompProfile:
           type: RuntimeDefault
       containers:
         - name: dashboard-metrics-scraper
           image: kubernetesui/metrics-scraper:v1.0.8
           ports:
             - containerPort: 8000
               protocol: TCP
           livenessProbe:
             httpGet:
               scheme: HTTP
               path: /
               port: 8000
             initialDelaySeconds: 30
             timeoutSeconds: 30
           volumeMounts:
           - mountPath: /tmp
             name: tmp-volume
           securityContext:
             allowPrivilegeEscalation: false
             readOnlyRootFilesystem: true
             runAsUser: 1001
             runAsGroup: 2001
       serviceAccountName: kubernetes-dashboard
       nodeSelector:
         "kubernetes.io/os": linux
       # Comment the following tolerations if Dashboard must not be deployed on master
       tolerations:
         - key: node-role.kubernetes.io/master
           effect: NoSchedule
       volumes:
         - name: tmp-volume
           emptyDir: {}
```

- 安装

   通过执行以下命令安装Dashboard：

```other
kubectl apply -f recommended.yaml
```

   此时就静静等待，直到`kubectl get pods -A`命令下显示都是`Running`状态：

   > kubernetes-dashboard dashboard-metrics-scraper-7bc864c59-tdxdd 1/1 Running 0 5m32s

   > kubernetes-dashboard kubernetes-dashboard-6ff574dd47-p55zl 1/1 Running 0 5m32s

- 查看端口

```other
kubectl get svc -n kubernetes-dashboard
```

![Image.png](k8s%E9%83%A8%E7%BD%B2/Image.png)

- 浏览器访问Dashboard
```bash
因为不能直接访问dashboard,因此使用proxy
kubectl proxy
```

![Image.png](k8s%E9%83%A8%E7%BD%B2/Image%20(2).png)

   使用`Token`登录

- 生成Token

   在master节点中执行下面命令创建`admin-user`：

```other
sudo vi dash.yaml
```

   配置文件内容如下：

```yaml
apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: admin-user
   namespace: kubernetes-dashboard
 ---
 apiVersion: rbac.authorization.k8s.io/v1
 kind: ClusterRoleBinding
 metadata:
   name: admin-user
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: cluster-admin
 subjects:
 - kind: ServiceAccount
   name: admin-user
   namespace: kubernetes-dashboard
```

   接下来我们通过下面命令创建`admin-user`：

```other
kubectl apply -f dash.yaml
```

   日志如下：

   > serviceaccount/admin-user created

   > [clusterrolebinding.rbac.authorization.k8s.io/admin-user](http://clusterrolebinding.rbac.authorization.k8s.io/admin-user) created
这就表示`admin-user`账号创建成功，接下来我们生成该用户的`Token`：

```sql
kubectl -n kubernetes-dashboard create token admin-user
```

![cb8265989dab4ccda66c641084e96c32~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp](k8s%E9%83%A8%E7%BD%B2/cb8265989dab4ccda66c641084e96c32~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

   我们把生成的`Token`拷贝好贴进浏览器对应的输入框中并点击`登录`按钮：

至此，我们就在`ubuntu 22.04`上完成了`k8s`的安装。

#### 备注：

参考文档里面有进行安装docker，但是我查了官方文档，k8s不需要同时安装docker跟containerd，只要一个容器运行时就行。我这边选用containerd。

参考文档：

[https://juejin.cn/post/7208910021970952252](https://juejin.cn/post/7208910021970952252)

