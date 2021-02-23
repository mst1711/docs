**OS: CentOS7 with SELinux disabled**  
**Kubernetes: 1.20.4**  
**CRI-O: 1.20**  

# Deploy CRI-O
```
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf  
overlay  
br_netfilter  
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params, these persist across reboots.  
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf  
net.bridge.bridge-nf-call-iptables  = 1  
net.ipv4.ip_forward                 = 1  
net.bridge.bridge-nf-call-ip6tables = 1  
EOF    

sudo sysctl --system    

export OS=CentOS_7  
export VERSION=1.20    

sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo  

sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo  

sudo yum install cri-o -y  

sudo systemctl --now enable crio  
sudo systemctl start crio
```

# Deploy Kubernetes bins
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

# THIS IS VERY VERY IMPORTANT!!!!
echo "KUBELET_EXTRA_ARGS=--container-runtime=remote --cgroup-driver=systemd --container-runtime-endpoint='unix:///var/run/crio/crio.sock'" > /etc/sysconfig/kubelet

# If one master node, if multiple master see below
kubeadm init --pod-network-cidr=192.168.0.0/16
```

# Deploy High Availbility for Kubernetes
On dedicated machine install haproxy with config
```
frontend kubernetes-frontend
    bind [ip]:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server kmaster1 [node_ip]:6443 check fall 3 rise 2
    server kmaster2 [node_ip]:6443 check fall 3 rise 2
    server kmaster3 [node_ip]:6443 check fall 3 rise 2
```
Execute on first master node
```
kubeadm init --control-plane-endpoint="lb.domain.name:6443" --upload-certs --apiserver-advertise-address=[current_node_ip] --pod-network-cidr=192.168.0.0/16
```
And after execute kubeadm join ... on other nodes with --apiserver-advertise-address=[current_node_ip] on each node

| :INFO: Чтобы на мастер ноде запускались поды
| kubectl taint nodes [nodename] node-role.kubernetes.io/master-
| или
| kubectl taint nodes --all node-role.kubernetes.io/master-



