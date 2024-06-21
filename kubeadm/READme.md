#  **<span style="color:green">Sani Solutions, Aalborg Denmark.</span>**
### **<span style="color:green">Contacts: +45 71 573 047</span>**
### **Email: forchu.cha@gmail.com**



## Kubernetes Setup Using Kubeadm In AWS EC2 Ubuntu Servers.
##### Prerequisite
+ AWS Acccount.
+ Create 3 - Ubuntu Servers -- 18.04.
+ 1 Master (4GB RAM , 2 Core)  t2.medium
+ 2 Workers  (1 GB, 1 Core)     t2.micro
+ Create Security Group and open required ports for kubernetes.
   + Open all port for this illustration
# *Control plane*
Protocol	Direction	Port Range	Purpose	Used By
+ TCP	Inbound	6443	Kubernetes API server	All
+ TCP	Inbound	2379-2380	etcd server client API	kube-apiserver, etcd
+ TCP	Inbound	10250	Kubelet API	Self, Control plane
+ TCP	Inbound	10259	kube-scheduler	Self
+ TCP	Inbound	10257	kube-controller-manager	Self
 # *Worker node(s)*
+ Protocol	Direction	Port Range	Purpose	Used By
+ TCP	Inbound	10250	Kubelet API	Self, Control plane
+ TCP	Inbound	10256	kube-proxy	Self, Load balancers
+ TCP	Inbound	30000-32767	NodePort Services†	All


## Steps

### Step 1: Set the Hostname

Replace `node1` with the appropriate hostname for each node.

```bash
sudo hostnamectl set-hostname master
sudo su ubuntu
```
## **Disable Swap**:
   ```bash
   sudo swapoff -a
   sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
   ```
### Step 2: Disable Swap & Add Kernel Settings

Disable swap and modify `/etc/fstab`.

```bash
sudo swapoff -a
```
```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 3: Load Kernel Modules and Configure Sysctl

Load necessary kernel modules and configure sysctl settings.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```bash
sudo modprobe overlay
```
```bash
sudo modprobe br_netfilter
```
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```
```bash
sudo sysctl --system
```

### Step 4: Install Containerd Runtime

Install containerd and its dependencies.

```bash
sudo apt-get update -y
```
```bash
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

Add Docker’s official GPG key.

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Set up the Docker repository.

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install containerd.

```bash
sudo apt-get update -y
sudo apt-get install -y containerd.io
```

Generate the default configuration file for containerd.

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Update containerd configuration to use systemd as the cgroup driver.

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Restart and enable containerd service.

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Step 5: Install kubeadm, kubelet, and kubectl

Update the apt package index and install packages needed to use the Kubernetes apt repository.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

Download the Google Cloud public signing key.

```bash
curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

Add the Kubernetes apt repository.

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update apt package index, install kubelet, kubeadm, and kubectl, and pin their versions.

```bash
sudo apt-get update
```
```bash
sudo apt-get install -y kubelet kubeadm kubectl
```
```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

Enable and start kubelet service.

```bash
sudo systemctl daemon-reload
sudo systemctl start kubelet
sudo systemctl enable kubelet.service
```

### Step 6: Initialize the Control Plane Node

On the control plane node, run:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<control-plane-ip>
```

Set up kubeconfig for the `ubuntu` user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Step 7: Install Pod Network Add-on

Install the Flannel network add-on:

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### Step 8: Join Worker Nodes

On each worker node, run the command provided by the `kubeadm init` output on the control plane node. The command will look something like this:

```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Step 9: Verify the Cluster

On the control plane node, verify that all nodes are joined and ready:

```bash
kubectl get nodes
```

### Step 10: Deploy NGINX Pod

Create a deployment for NGINX:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
```

### Step 11: Access NGINX

Get the NodePort assigned to the NGINX service:

```bash
kubectl get svc nginx
```

Access NGINX using the NodePort and the IP address of any node in the cluster:

```
http://<node-ip>:<node-port>
```

This concludes the steps to set up a Kubernetes cluster with `kubeadm`, deploy an NGINX pod, and access it via a browser.
```


