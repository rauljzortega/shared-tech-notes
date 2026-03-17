# Kubernetes Cluster Bootstrapping with kubeadm
This Kubernetes cluster was deployed on **Rocky Linux 9.7** nodes using the `kubeadm` utility.  
For the general official guide and system requirements, please refer to the [Kubernetes Production Environment Setup](https://kubernetes.io/docs/setup/production-environment/).  


## Setup summary (3 nodes example)
This will be our cluster setup:  
- Node 1: **Controlplane**  
- Nodes 2 & 3: **Worker Nodes**  

To create a Kubernetes cluster using **kubeadm** we need to follow the next steps on the specified nodes:  
1. [Prepare your nodes](#1-nodes-configuration) *(All nodes)*
1. [Install containerd](#2-install-containerd-all-nodes) *(All nodes)*
1. [Install kubernetes tools](#3-install-kubernetes-tools-all-nodes) *(All nodes)*
1. [Initialize Controlplane](#4-initializing-your-control-plane-node-controlplane) *(Controlplane)*
1. [Enable POD Network Connectivity](#5-cni-controlplane) *(Controlplane)*
1. [Join worker nodes to the cluster](#6-join-the-worker-nodes-worker-nodes) *(Worker nodes)*
 

# Create Kubernetes cluster (Installed on Rocky 9.7)  
## 1. Nodes configuration
### 1.1. Disable SELinux & Firewalld (All nodes)
Disable the firewall to allow all Kubernetes and CNI traffic:  
```bash
# Disable firewalld
sudo systemctl disable --now firewalld
```  

```bash
# Disable SELinux temporarily
sudo setenforce 0
# Make it persistent
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```  

### 1.2. Swap configuration (All nodes)
We must disable swap for Kubernetes to work properly.  
To disable swap temporarily we can use this command:  
```bash
sudo swapoff -a
```  
To disable swap after the system reboot we have to comment or remove the swap line in `/etc/fstab`.  

### 1.3. Network configuration (All nodes)

Set up the required kernel modules and make them persistent  
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```  

Set the required kernel parameters and make them persistent
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```  

We have to tell kubernetes what network we want to use for nodes connectivity. By default it takes the first network.  
In order to set it we use these commands:
```bash
PRIMARY_IP=<yourIPAddress> #Change for your internal IP. Each node has to set its own IP

cat <<EOF | sudo tee /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS='--node-ip ${PRIMARY_IP}'
EOF
```

## 2. Install containerd (All nodes)

Run the following commands to install containerd:  
```bash
# Add docker repository to install latest containerd version
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y containerd.io
```  
Set containerd config: 
```bash
containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' | sudo tee /etc/containerd/config.toml
```  
Enable containerd:
```bash
sudo systemctl enable --now containerd
```

## 3. Install Kubernetes tools (All nodes)
Add Kubernetes yum repo.  
*NOTE: The repository URL points to v1.35. If you are installing a newer version in the future, make sure to update the URL accordingly (e.g.: v1.36).*  
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```  
Install Kubernetes tools (kubectl is optional on worker nodes):  
```bash
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```  
Enable kubelet:  
```bash
sudo systemctl enable --now kubelet
```  


### Configure crictl (optional)
Configure crictl in case we need it to examine running containers:
```bash
sudo crictl config \
    --set runtime-endpoint=unix:///run/containerd/containerd.sock \
    --set image-endpoint=unix:///run/containerd/containerd.sock
```




## 4. Initializing your control-plane node (Controlplane)
To initialize the control-plane we have to use this command: ```kubeadm init <args>```  
However, we should use some useful flags to modify the default configuration.  
First, set the IP address you want to use for communication between nodes:
```bash
PRIMARY_IP=<yourIPAddress>
```  
Also, if you want to set the IP range your Pods and Kubernetes Services are going to use, set these environment variables:
```bash
#IP ranges example:
POD_CIDR=10.244.0.0/16
SERVICE_CIDR=10.96.0.0/16
```  
Now, run the next command (using the previous env variables):
```bash
sudo kubeadm init --pod-network-cidr $POD_CIDR --service-cidr $SERVICE_CIDR --apiserver-advertise-address $PRIMARY_IP
```  

After it finishes you should follow these instructions, which should be shown in your output.  
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user: 

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

In the case your are working as root:

  export KUBECONFIG=/etc/kubernetes/admin.conf

[...]
```

NOTE: if something went wrong, you solved it and you want to use ```kubeadm init``` again, you must do these steps before:  
```bash
kubeadm reset -f
rm -rf /etc/kubernetes/*
rm -rf /var/lib/etcd
sudo rm -rf /etc/cni/net.d
systemctl restart kubelet
```  

## 5. CNI (Controlplane)

At this moment, if we run ```kubectl get node``` we will see the node is not ready. It's because we have to install a CNI (Container Network Interface) plugin.  
List of addons: https://kubernetes.io/docs/concepts/cluster-administration/addons/  

For example, we are going to install calico CNI (Container Network Interface) plugin.  
Follow the official web calico steps: https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico  
```
NOTE: After the 2nd step we have to make a change in the YAML file we download.  
Use a text editor to change the configuration file. In 'IpPools:' we must change the CIDR to the range we attached before to the env variable POD_CIDR. In our case it is 10.244.0.0/16.  
```
Then, just follow the official documentation steps.  

Afterwards, by running ```kubectl get node``` we will see that the Control Plane node status is ready., so now we can join the worker nodes to the cluster.  


## 6. Join the worker nodes (Worker Nodes)

Now you just have to paste the command printed previously when you initialized the control-plane.
If you lost the join command with the token, you can get it again running on the master node:
```bash
sudo kubeadm token create --print-join-command
```
Once the workers have joined the cluster, run ```kubectl get nodes``` on control-plane node and you should see something like this:
```
[root@head01 ~]# kubectl get nodes
NAME     STATUS   ROLES           AGE     VERSION
head01   Ready    control-plane   119m    v1.35.0
host2    Ready    <none>          18m     v1.35.0
host3    Ready    <none>          2m47s   v1.35.0
```  


## Post-installation (Add-ons)
### Helm
If you need Helm in your cluster, follow [this docu](helm/helm.md).  

### StorageClass
In order to define how storage should be dynamically provisioned in your cluster, you need to create a StorageClass object.  
You can easily install it using helm. Follow [this docu](helm/storageclass.md) to do it.  

### Ingress Controller
In order to be able to access the web, we must install an Ingress Controller.  
You can easily install it using helm. Follow [this docu](helm/ingresscontroller.md) to do it.  


## Useful links and references

If you need to expand this setup, configure High Availability, or check specific API versions, refer to the official Kubernetes documentation below:

* **Kubernetes Releases:** https://kubernetes.io/releases/  
* **Network Configuration:** https://kubernetes.io/docs/setup/production-environment/container-runtimes/#network-configuration  
* **Install containerd**https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd*  
* **Installing kubeadm:** https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/  
* **Creating a Cluster:** https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/  
* **High Availability (HA):** https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/  
* **Kubernetes objects (API Groups - v1.34):** https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.34/  