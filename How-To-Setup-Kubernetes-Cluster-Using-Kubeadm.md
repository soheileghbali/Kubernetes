# How To Setup Kubernetes Cluster Using Kubeadm

## 1 - Disable swap on all the Nodes</h3>

```bash
sudo swapoff -a

sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 2- Install Docker Container Runtime On All The Nodes

### Install the required packages for Docker 
``` bash
sudo apt-get update -y
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
### Add the Docker GPG key and apt repository

``` bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
### Install the Docker community edition

``` bash
sudo apt-get update -y
sudo apt-get install docker-ce docker-ce-cli containerd.io -y
```
### Add the docker daemon configurations to use systemd as the cgroup driver
``` bash
cat <<EOF | sudo tee /etc/docker/daemon.json
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
### Start and enable the docker service
```bash
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 3- Install Kubeadm & Kubelet & Kubectl on all Nodes

### Install the required dependencies
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
### Add the GPG key and apt repository
```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
### Update apt and install kubelet, kubeadm and kubectl
```bash
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
```
### Note: If you want to install a specific version of kubernetes, you can specify the version as shown below
```bash
sudo apt-get install -y kubelet=1.20.6-00 kubectl=1.20.6-00 kubeadm=1.20.6-00
```
### Add hold to the packages to prevent upgrades.
```bash
sudo apt-mark hold kubelet kubeadm kubectl
```
## 4 - Initialize Kubeadm On Master Node To Setup Control Plane

### initialize the master node control plane configurations using the following kubeadm command
```bash
sudo kubeadm init --ignore-preflight-errors Swap
```
#### Use the following commands from the output to create the kubeconfig in master so that you can use kubectl to interact with cluster API
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## &#9830; 5 - If the master node status is not ready ! , do the following commands on the master node 
```bash
sudo kubectl taint node $HOSTNAME key:NoSchedule
sudo  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubeadm reset

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubeadm init
```
## 6 - Join Worker Nodes To Kubernetes Master Node
#### join the worker node to the master node using the Kubeadm join command you have got in the output while setting up the master node
```bash
kubeadm token create --print-join-command
sudo kubeadm join 10.128.0.37:6443 --token j4eice.33vgvgyf5cxw4u8i \
    --discovery-token-ca-cert-hash sha256:37f94469b58bcc8f26a4aa44441fb17196a585b37288f85e22475b00c36f1c61
```
## 7 - Execute the kubectl command to check if the node is added to the master
```bash
kubectl get nodes
```



