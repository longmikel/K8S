# Install RKE
## kubectl
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

## rke
```bash
curl -s https://api.github.com/repos/rancher/rke/releases/latest | grep download_url | grep amd64 | cut -d '"' -f 4 | wget -qi -
chmod +x rke_linux-amd64
sudo mv rke_linux-amd64 /usr/local/bin/rke
rke --version
```

## helm
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

# Install Kubernetes Cluster with RKE
## Triển khai với 5 Nodes:
- 3 Master Nodes - etcd and control plane ( 3 for HA)
- 2 Worker Nodes

### Cấu hình Node dự kiến:
- Master Nodes: 8GB RAM và 4 vCPUs
- Worker Nodes: 8GB RAM và 8 vCPUs

## Chuẩn bị môi trường để triển khai Kubernetes Cluster với RKE
RKE chạy trên hầu hết mọi hệ điều hành Linux có cài đặt Docker. Rancher đã được thử nghiệm và được hỗ trợ với:
- Red Hat Enterprise Linux
- Oracle Enterprise Linux
- CentOS Linux
- Ubuntu
- RancherOS

### Bước 1: Cập nhật hệ thống
```bash
--- CentOS ---
$ sudo yum -y update
$ sudo reboot

--- Ubuntu / Debian ---
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo reboot
```

### Bước 2: Tạo tài khoản khác với root dùng để kết nối trên toàn bộ nodes trong Kubernetes Cluster
Sử dụng Ansible với Playbook được định nghĩa sẵn như sau:
```yaml
---
- name: Create rke user with passwordless sudo
  hosts: rke-hosts
  remote_user: root
  tasks:
    - name: Add RKE admin user
      user:
        name: rke
        shell: /bin/bash

    - name: Create sudo file
      file:
        path: /etc/sudoers.d/rke
        state: touch

    - name: Give rke user passwordless sudo
      lineinfile:
        path: /etc/sudoers.d/rke
        state: present
        line: 'rke ALL=(ALL:ALL) NOPASSWD: ALL'

    - name: Set authorized key taken from file
      authorized_key:
        user: rke
        state: present
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

### Bước 3: Kích hoạt các Modules Kernel cần thiết cho từng Node trong Kubernetes Cluster
Sử dụng Ansible với Playbook được định nghĩa sẵn như sau:
```yaml
---
- name: Load RKE kernel modules
  hosts: rke-hosts
  remote_user: root
  vars:
    kernel_modules:
      - br_netfilter
      - ip6_udp_tunnel
      - ip_set
      - ip_set_hash_ip
      - ip_set_hash_net
      - iptable_filter
      - iptable_nat
      - iptable_mangle
      - iptable_raw
      - nf_conntrack_netlink
      - nf_conntrack
      - nf_conntrack_ipv4
      - nf_defrag_ipv4
      - nf_nat
      - nf_nat_ipv4
      - nf_nat_masquerade_ipv4
      - nfnetlink
      - udp_tunnel
      - veth
      - vxlan
      - x_tables
      - xt_addrtype
      - xt_conntrack
      - xt_comment
      - xt_mark
      - xt_multiport
      - xt_nat
      - xt_recent
      - xt_set
      - xt_statistic
      - xt_tcpudp

  tasks:
    - name: Load kernel modules for RKE
      modprobe:
        name: "{{ item }}"
        state: present
      with_items: "{{ kernel_modules }}"
```

### Bước 4: Vô hiệu hoá SWAP và thiết lập sysctl
P/S: đây là khuyến nghị của Kubernetes về việc vô hiệu hoá SWAP và thiết lập sysctl
Sử dụng Ansible với Playbook được định nghĩa sẵn như sau:
```yaml
---
- name: Disable swap and load kernel modules
  hosts: rke-hosts
  remote_user: root
  tasks:
    - name: Disable SWAP since kubernetes can't work with swap enabled (1/2)
      shell: |
        swapoff -a

    - name: Disable SWAP in fstab since kubernetes can't work with swap enabled (2/2)
      replace:
        path: /etc/fstab
        regexp: '^([^#].*?\sswap\s+.*)$'
        replace: '# \1'
    - name: Modify sysctl entries
      sysctl:
        name: '{{ item.key }}'
        value: '{{ item.value }}'
        sysctl_set: yes
        state: present
        reload: yes
      with_items:
        - {key: net.bridge.bridge-nf-call-ip6tables, value: 1}
        - {key: net.bridge.bridge-nf-call-iptables,  value: 1}
        - {key: net.ipv4.ip_forward,  value: 1}
```

### Bước 5: Cài đặt Docker cho từng Nodes trong Kubernetes Cluster
1. Install Docker:
```bash
https://releases.rancher.com/install-docker/19.03.sh | sudo bash -
```

2. Khởi động dịch vụ Docker:
```bash
sudo systemctl enable --now docker
```

3. Xác nhận phiên bản Docker của từng Nodes trong Kubernetes Cluster cùng 1 phiên bản:
```bash
sudo docker version --format '{{.Server.Version}}'
```

4. Thêm user rke khởi tạo ban đầu trên từng Nodes vào nhóm docker:
```bash
sudo usermod -aG docker rke
```

### Bước 6: Cho phép các cổng kết nối trên Firewall
Kiểm tra tất cả các cổng kết nối được sử dụng theo link: https://rancher.com/docs/rancher/v2.x/en/installation/requirements/#operating-systems-and-docker-requirements

### Bước 7: Cho phép chuyển tiếp SSH TCP
Cho phép chuyển tiếp TCP trên toàn hệ thống máy chủ SSH.
1. Thiết lập cho phép AllowTcpForwarding
```bash
sudo vi /etc/ssh/sshd_config
AllowTcpForwarding yes
```

2. Khởi động lại dịch vụ SSH
```bash
sudo systemctl restart ssh
```

### Bước 8: Tạo tập tin cấu hình Kubernetes Cluster
RKE sử dụng tệp cấu hình cụm, được gọi là cluster.yml để xác định các nút nào sẽ nằm trong cụm và cách triển khai Kubernetes Cluster.

Có nhiều tùy chọn cấu hình có thể được đặt trong cluster.yml. Tập tin này có thể được tạo sẵn bởi người quản trị hoặc được tạo bằng lệnh rke config.

Chạy lệnh rke config để tạo một tệp cluster.yml mới trong thư mục hiện tại của bạn.
rke config --name cluster.yml

Lệnh này sẽ nhắc bạn bổ sung tất cả các thông tin cần thiết để xây dựng một cụm. Thay vào đó, nếu bạn muốn tạo một tệp cluster.yml mẫu trống, chỉ định cờ --empty.

Đây là tệp cấu hình cụm mẫu trông giống như sau:
```yaml
# https://rancher.com/docs/rke/latest/en/config-options/
nodes:
- address: IP-Public
  internal_address: IP-Private
  hostname_override: rke-master-01
  role: [controlplane, etcd]
  user: rke
- address: IP-Public
  internal_address: IP-Private
  hostname_override: rke-master-02
  role: [controlplane, etcd]
  user: rke
- address: IP-Public
  internal_address: IP-Private
  hostname_override: rke-master-03
  role: [controlplane, etcd]
  user: rke
- address: IP-Public
  internal_address: IP-Private
  hostname_override: rke-worker-01
  role: [worker]
  user: rke
- address: IP-Public
  internal_address: IP-Private
  hostname_override: rke-worker-02
  role: [worker]
  user: rke

# using a local ssh agent
# Using SSH private key with a passphrase - eval `ssh-agent -s` && ssh-add
# ssh_agent_auth: true

#  SSH key that access all hosts in your cluster
ssh_key_path: ~/.ssh/id_rsa

# By default, the name of your cluster will be local
# Set different Cluster name
cluster_name: rke

# Fail for Docker version not supported by Kubernetes
ignore_docker_version: false

# prefix_path: /opt/custom_path

# Set kubernetes version to install: https://rancher.com/docs/rke/latest/en/upgrades/#listing-supported-kubernetes-versions
# Check with -> rke config --list-version --all
kubernetes_version:
# Etcd snapshots
services:
  etcd:
    backup_config:
      interval_hours: 12
      retention: 6
    snapshot: true
    creation: 6h
    retention: 24h

kube-api:
  # IP range for any services created on Kubernetes
  #  This must match the service_cluster_ip_range in kube-controller
  service_cluster_ip_range: 10.43.0.0/16
  # Expose a different port range for NodePort services
  service_node_port_range: 30000-32767
  pod_security_policy: false


kube-controller:
  # CIDR pool used to assign IP addresses to pods in the cluster
  cluster_cidr: 10.42.0.0/16
  # IP range for any services created on Kubernetes
  # # This must match the service_cluster_ip_range in kube-api
  service_cluster_ip_range: 10.43.0.0/16

kubelet:
  # Base domain for the cluster
  cluster_domain: cluster.local
  # IP address for the DNS service endpoint
  cluster_dns_server: 10.43.0.10
  # Fail if swap is on
  fail_swap_on: false
  # Set max pods to 150 instead of default 110
  extra_args:
    max-pods: 150

# Configure  network plug-ins
# KE provides the following network plug-ins that are deployed as add-ons: flannel, calico, weave, and canal
# After you launch the cluster, you cannot change your network provider.
# Setting the network plug-in
network:
    plugin: canal
    options:
      canal_flannel_backend_type: vxlan

# Specify DNS provider (coredns or kube-dns)
dns:
  provider: coredns

# Currently, only authentication strategy supported is x509.
# You can optionally create additional SANs (hostnames or IPs) to
# add to the API server PKI certificate.
# This is useful if you want to use a load balancer for the
# control plane servers.
authentication:
  strategy: x509
  sans:
    - "rke.matbao.io"

# Set Authorization mechanism
authorization:
    # Use `mode: none` to disable authorization
    mode: rbac

# Currently only nginx ingress provider is supported.
# To disable ingress controller, set `provider: none`
# `node_selector` controls ingress placement and is optional
ingress:
  provider: nginx
  options:
     use-forwarded-headers: "true"
```

Trong tệp cấu hình này, các Master Notes chỉ có vai trò etcd và controlplane. Nhưng chúng có thể được sử dụng schedule pod bằng cách thêm vai trò worker.
```bash
role: [controlplane, etcd, worker]
```

### Bước 9: Cài đặt Kubernetes Cluster với RKE
Khi bạn đã tạo tệp cluster.yml, bạn có thể triển khai cụm của mình bằng một lệnh đơn giản.
```bash
rke up
```

Lệnh này giả sử tệp cluster.yml nằm trong cùng thư mục với nơi bạn đang chạy lệnh. Nếu sử dụng tên tệp khác, chỉ định nó như bên dưới.
```bash
rke up --config ./rancher_cluster.yml
```

Nếu sử dụng khóa riêng SSH với mật khẩu - eval ssh-agent -s && ssh-add

Đảm bảo thiết lập không hiển thị bất kỳ lỗi nào trong đầu ra, ví dụ:
```bash
......
INFO[0181] [sync] Syncing nodes Labels and Taints       
INFO[0182] [sync] Successfully synced nodes Labels and Taints
INFO[0182] [network] Setting up network plugin: canal   
INFO[0182] [addons] Saving ConfigMap for addon rke-network-plugin to Kubernetes
INFO[0183] [addons] Successfully saved ConfigMap for addon rke-network-plugin to Kubernetes
INFO[0183] [addons] Executing deploy job rke-network-plugin
INFO[0189] [addons] Setting up coredns                  
INFO[0189] [addons] Saving ConfigMap for addon rke-coredns-addon to Kubernetes
INFO[0189] [addons] Successfully saved ConfigMap for addon rke-coredns-addon to Kubernetes
INFO[0189] [addons] Executing deploy job rke-coredns-addon
INFO[0195] [addons] CoreDNS deployed successfully..     
INFO[0195] [dns] DNS provider coredns deployed successfully
INFO[0195] [addons] Setting up Metrics Server           
INFO[0195] [addons] Saving ConfigMap for addon rke-metrics-addon to Kubernetes
INFO[0196] [addons] Successfully saved ConfigMap for addon rke-metrics-addon to Kubernetes
INFO[0196] [addons] Executing deploy job rke-metrics-addon
INFO[0202] [addons] Metrics Server deployed successfully
INFO[0202] [ingress] Setting up nginx ingress controller
INFO[0202] [addons] Saving ConfigMap for addon rke-ingress-controller to Kubernetes
INFO[0202] [addons] Successfully saved ConfigMap for addon rke-ingress-controller to Kubernetes
INFO[0202] [addons] Executing deploy job rke-ingress-controller
INFO[0208] [ingress] ingress controller nginx deployed successfully
INFO[0208] [addons] Setting up user addons              
INFO[0208] [addons] no user addons defined              
INFO[0208] Finished building Kubernetes cluster successfully
```

### Bước 10: Truy cập Kubernetes Cluster
Là một phần của quy trình tạo Kubernetes, tệp kubeconfig đã được tạo và ghi tại kube_config_cluster.yml.

Đặt biến KUBECONFIG cho tệp được tạo: kube_config_cluster.yml.

Kiểm tra danh sách Nodes trong cluster
```bash
kubectl get nodes
NAME            STATUS   ROLES               AGE     VERSION
rke-master-01   Ready    controlplane,etcd   2d19h   v1.17.6
rke-master-02   Ready    controlplane,etcd   2d19h   v1.17.6
rke-master-03   Ready    controlplane,etcd   2d19h   v1.17.6
rke-worker-01   Ready    worker              2d19h   v1.17.6
rke-worker-02   Ready    worker              2d19h   v1.17.6
```

Bạn có thể sao chép tệp này vào $ HOME/.kube/config nếu bạn không có bất kỳ cụm kubernetes nào khác.
