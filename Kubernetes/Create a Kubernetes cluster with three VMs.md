Map hostnames to their IP addresses:

```
printf "192.168.10.10 control-plane cp\n192.168.10.11 worker-1 w1\n192.168.10.12 worker-2 w2" >> /etc/hosts 
```

Disable SELinux:

```
setenforce 0 && sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Load kernel modules:

```
modprobe overlay && modprobe br_netfilter && printf "overlay\nbr_netfilter\n" > /etc/modules-load.d/k8s.conf
```

Configure sysctl for Kubernetes networking

```
printf "net.bridge.bridge-nf-call-ip6tables = 1\nnet.bridge.bridge-nf-call-iptables = 1\nnet.ipv4.ip_forward = 1" >> /etc/sysctl.d/k8s.conf && sysctl --system
```

Install the container runtime containerd:

```
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo && dnf install -y containerd.io
```

Configure containerd:

```
mkdir -p /etc/containerd && containerd config default > /etc/containerd/config.toml && sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Start containerd:

```
systemctl restart containerd && systemctl enable containerd
```

Install Kubernetes components:

```
printf "[kubernetes]\nname=Kubernetes\nbaseurl=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/\nenabled=1\ngpgcheck=1\ngpgkey=https://pkgs.k8s.io/core:/stable:/v1.28/rpm/repodata/repomd.xml.key" > /etc/yum.repos.d/kubernetes.repo && dnf install kubelet kubeadm kubectl -y --disableexcludes=kubernetes && systemctl enable --now kubelet
```

Disable firewalld:

```
systemctl disable firewalld --now
```

Add Bash auto-completion:

```
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null && chmod a+r /etc/bash_completion.d/kubectl && printf "alias k=kubectl\ncomplete -o default -F __start_kubectl k" >> ~/.bashrc && dnf install bash-completion -y && exec bash -l
```

Initialize the cluster on the Control Plane node:

```
kubeadm init --pod-network-cidr=10.22.0.0/8
```

Copy the config to the write directory:

```
mkdir -p $HOME/.kube && cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

Add calico, but make sure to change the subnet:

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml -O
```

```
sed -i 's/192.168.0.0\/16/10.244.0.0\/16/g' calico.yaml
```

```
kubectl apply -f calico.yaml
```

Join the worker nodes:

```
kubeadm join 192.168.10.10:6443 --token jjhapz.hrpaynzirspn4gut \
        --discovery-token-ca-cert-hash sha256:8d6b9e32671c33869a43a1e062b53696bb07f80aa94a2c907d14e8a311305eff
```
