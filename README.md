# Kubernetes Installation on Ubuntu 22.04

Get the detailed information about the installation from the below-mentioned websites of **cri-o** and **Kubernetes**.

[cri-o](https://cri-o.io/)

[Kubernetes](https://kubernetes.io/)

### Set up the cri-o and Kubernetes repositories:

> Download and install the necessary tools for Cri-o
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common

# Export the OS and CRI_O version values
export OS_VERSION=xUbuntu_22.04
export CRIO_VERSION=1.26
```

> Download the GPG key for cri-o

```bash
sudo curl -fsSL https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS_VERSION/Release.key | sudo gpg --dearmor -o /usr/share/keyrings/libcontainers-archive-keyring.gpg

sudo curl -fsSL https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$CRIO_VERSION/$OS_VERSION/Release.key | sudo gpg --dearmor -o /usr/share/keyrings/libcontainers-crio-archive-keyring.gpg
```

> Add the cri-o repository

```bash
# we can get the latest release details from https://cri-o.io/

sudo echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS_VERSION/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list

sudo echo "deb [signed-by=/usr/share/keyrings/libcontainers-crio-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$CRIO_VERSION/$OS_VERSION/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$CRIO_VERSION.list

```

> Add the GPG key for kubernetes

```bash
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://dl.k8s.io/apt/doc/apt-key.gpg
```

> Add the kubernetes repository

**Check for the latest release in https://packages.cloud.google.com/apt/dists**

```bash
sudo echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

> Update the repository

```bash
# Update the repositiries
sudo apt-get update
```

> Install cri-o and Kubernetes packages.

**Note that if you want to use a newer version of Kubernetes, change the version installed for kubelet, kubeadm, and kubectl and be sure that all three use the same version.
These version should support the cri-o version.**

```bash
# Use the same versions to avoid issues with the installation.
sudo apt-get install -y cri-o cri-o-runc kubelet=1.27.3-00 kubeadm=1.27.3-00 kubectl=1.27.3-00
```

```bash
# Start the cri-o service
sudo systemctl daemon-reload
sudo systemctl enable crio
sudo systemctl start crio
```

> To hold the versions so that the versions will not get accidently upgraded.

```bash
sudo apt-mark hold cri-o kubelet kubeadm kubectl
```

> Enable the iptables bridge

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

### On the Kube master server

> Initialize the cluster by passing the cidr value.

**Use Calico**

```bash
# Calico network
# Make sure to copy and backup the join command
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Copy your join command and keep it safe.
# Below is a sample
kubeadm join 10.128.0.2:6443 --token swi0ci.jq9l75eg8lvpxz6g --discovery-token-ca-cert-hash sha256:2c3cdfa898334b0dfc0f73bbccb998d03f61252ee50f0405c85ba735ff90b4d1
```

> To start using the cluster with current user.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> To set up the Calico network

```bash
# Use this if you have initialised the cluster with Calico network add on.
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/custom-resources.yaml -O


kubectl create -f custom-resources.yaml

```

> Check the nodes

```bash
# Check the status on the master node.
kubectl get nodes
```

### On each of Kube node server

> Joining the node to the cluster:

```bash
sudo kubeadm join $controller_private_ip:6443 --token $token --discovery-token-ca-cert-hash $hash
```

**TIP**

> If the joining code is lost, it can retrieve using below command

```bash
kubeadm token create --print-join-command
```

### To install the metrics server

```bash
git clone https://github.com/mialeevs/kubernetes_installation_crio.git
cd kubernetes_installation_crio/
kubectl apply -f metrics-server.yaml
cd
rm -rf kubernetes_installation_crio/
```
