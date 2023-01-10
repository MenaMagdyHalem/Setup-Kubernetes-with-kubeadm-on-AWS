# Setup Kubernetes with kubeadm on AWS

Table of Content:

- [Setup Kubernetes with kubeadm on AWS](#setup-kubernetes-with-kubeadm-on-aws)
- [Open AWS Console](#open-aws-console)
    - [We will create 3 EC2 instances `(One for the K8s master node)` and `(two for the worker nodes)`](#we-will-create-3-ec2-1-for-master-node-and-2-for-worker-node)
- [Install Docker on all Machines](#install-docker-on-all-machine)
- [Configure The Hostname](#configure-the-hostname)
- [Install Kubernetes on All Machines](#install-kubernetes-on-all-machines)
- [Initialize Kubernetes on the Master Node](#initialize-kubernetes-on-the-master-node)
- [Joining the Worker Nodes to the Kubernetes Cluster](#joining-worker-nodes-to-the-kubernetes-cluster)
- [Reset Kubeadm](#reset-kubeadm)

# Open AWS Console

### We will create 3 EC2 instances `(1 for master node)` and `(2 for worker node)`

- Specifications for the master node EC2 instance:
  - `Instance Type:` t3.medium
  - `OS:` ubuntu 20.04
  - `Storage:` 20 GB (gp2)

---

- Specifications for the worker nodes:
  - `Instance Type:` t3.medium
  - `OS:` ubuntu 20.04
  - `Storage:` 10 GB Storage, but recommend 20G

# Install Docker on all Machines

Update the existing packages on the machines:

```text
sudo apt update
```

Install a prerequisite package that allows apt to utilize HTTPS:

```text
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

Add GPG key for the official Docker repo to the Ubuntu system:

```text
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```


Add the Docker repo to the APT sources:

```text
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Update the database with the Docker packages from the added repo:

```text
sudo apt update
```

Install Docker:

```text
sudo apt install -y containerd.io docker-ce docker-ce-cli
```

Docker should now be installed, the daemon is started, and the process is enabled to start on boot. To verify:

```text
sudo systemctl status docker
```

Enable Docker to start automatically after rebooting the machine:

```text
sudo systemctl enable docker
```

Add user to docker **Groups**:

```text
sudo usermod -aG docker ${USER}
```

In AWS ec2 instance with Ubuntu linux, the user is 'ubuntu'

```text
sudo usermod -aG docker ubuntu
```

Create the required directories

```text
sudo mkdir -p /etc/systemd/system/docker.service.d
```

Configure the cgroup

```text
sudo tee /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

```text
sudo systemctl restart docker
```

Disable the swap memory (if running) on both the worker nodes and the master node:

```text
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

sudo swapoff -a
```

Install selinux (on all Machines):

```text
sudo apt install selinux-utils
```

Disable selinux (on all Machines):

```text
setenforce 0
```

Assign a unique hostname for every machine:

```text
sudo hostnamectl set-hostname master
```

```text
sudo hostnamectl set-hostname node1
```

```text
sudo hostnamectl set-hostname node2
```

> **Restart All Machines.**
---

# Configure The Hostname

Set the hostname for all machines

```text
sudo vim /etc/hosts
```

We will add:

```text
<ip-master>   <hostname-master>
<ip-node1>    <hosyname-node1>
<ip-node1>    <hosyname-node1>
```

---

# Install Kubernetes on All Machines

Add the Kubernetes signing key:

```text
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```

Add Xenial Kubernetes Repository:

```text
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

Update The existing packages:

```text
sudo apt update
```

Install Kubeadm:

```text
sudo apt-get install -y kubelet=1.21.5-00 kubectl=1.21.5-00 kubeadm=1.21.5-00 --allow-change-held-packages

sudo apt-mark hold kubelet kubeadm kubectl
```

Enable kernel modules

```text
sudo modprobe overlay
sudo modprobe br_netfilter
```

Update Iptables Settings

```text
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Reload sysctl

```text
sudo sysctl --system
```

Start and enable Services

```text
sudo systemctl daemon-reload

sudo systemctl restart docker

sudo systemctl enable docker

sudo systemctl enable kubelet
```

# Initialize Kubernetes on the Master Node

Run the following command as sudo on the master node:

```text
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

To start using your cluster, you need to run the following as a regular user:

```text
mkdir -p $HOME/.kube
```

```text
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

```text
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> In the output, you can see the `kubeadm join` command and a unique token that you will run on the worker node and all other worker nodes that you want to join onto this cluster. Next, copy-paste this command as you will use it need to join the cluster. We will do this when we join the worker nodes

To Create a new token as root

```text
sudo kubeadm token create --print-join-command
```

Deploy a Pod Network through the master node:

```text
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

# Joining Worker Nodes to the Kubernetes Cluster

You will use your kubeadm join command that was shown in your terminal when we initialized the master node.

The command would be similar to the following: 

```text
sudo kubeadm join 172.31.6.233:6443 --token 9lspjd.t93etsdpwm9gyfib --discovery-token-ca-cert-hash sha256:37e35d7ea83599356de1fc5c80c282285cc3c749443a1dafd8e73f40
```

---

# Reset Kubeadm

```text
sudo kubeadm reset -f
```
