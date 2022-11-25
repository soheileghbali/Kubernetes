# Introduction

As you already know, Kubernetes is not supporting docker as a container runtime enviroment any more, and you need to choose another cri such as [CRIO](https://cri-o.io/) or [containerd](https://containerd.io/).
That said, it is possible to continue using docker as a cri with the help of [this](https://github.com/Mirantis/cri-dockerd) project.
In the following steps, I will show you how to install Kubernetes on premises with the help of [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/).

## Before you begin[](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)

-   A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
-   2 GB or more of RAM per machine (any less will leave little room for your apps).
-   2 CPUs or more.
-   Full network connectivity between all machines in the cluster (public or private network is fine).
-   Unique hostname, MAC address, and product_uuid for every node. See  [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#verify-mac-address)  for more details.
-   Certain ports are open on your machines. See  [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)  for more details.
-   Swap disabled. You  **MUST**  disable swap in order for the kubelet to work properly.
## Steps
### 1- Disable Swap
Disable all swaps from /proc/swaps.

    sudo swapoff -a
   Check if swap has been disabled by running the free command.

    free -h
Now disable Linux swap space permanently in `/etc/fstab`. Search for a swap line and add `#` (hashtag) sign in front of the line.

    $ sudo vim /etc/fstab
    #/swap.img	none	swap	sw	0	0
### 2- Install Packages
containerd prerequisites, and load two kernel modules and configure them to load on boot.
https://kubernetes.io/docs/setup/production-environment/container-runtimes/

    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
      
    sudo modprobe overlay
    sudo modprobe br_netfilter
sysctl params required by setup, params persist across reboots.

    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF

Apply sysctl params without reboot.

    sudo sysctl --system
### 3- Install containerd

    sudo apt-get update
    sudo apt-get install -y containerd
 Create a containerd configuration file.    

    sudo mkdir -p /etc/containerd
    sudo containerd config default | sudo tee /etc/containerd/config.toml
Set the cgroup driver for containerd to systemd which is required for the kubelet.
For more information on this config file see:
https://github.com/containerd/cri/blob/master/docs/config.md 
https://github.com/containerd/containerd/blob/master/docs/ops.md
At the end of this section, change SystemdCgroup = false to SystemdCgroup = true

            [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
                    ...
                    #          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
                                SystemdCgroup = true

You can use sed to swap in true.

    sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
Verify the change was made.

    sudo vi /etc/containerd/config.toml
Restart containerd with the new configuration.

    sudo systemctl restart containerd

### 4- Install Kubernetes packages - kubeadm, kubelet and kubectl   
Add Google's apt repository gpg key.

    sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
Add the Kubernetes apt repository.

    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

Update the package list and use apt-cache policy to inspect versions available in the repository.

    sudo apt-get update
    apt-cache policy kubelet | head -n 20 
Install the required packages, if needed we can request a specific version. 

    VERSION=1.24.3-00
    sudo apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION
    sudo apt-mark hold kubelet kubeadm kubectl containerd
To install the latest, omit the version parameters.

    sudo apt-get install kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl containerd
Check the status of our kubelet and our container runtime, containerd.
The kubelet will enter a crashloop until a cluster is created or the node is joined to an existing cluster.

    sudo systemctl status kubelet.service
    sudo systemctl status containerd.service 
    
Ensure both are set to start when the system starts up.

    sudo systemctl enable kubelet.service
    sudo systemctl enable containerd.service

## Creating Control PlaneNode
### A - Initialize control plane (run on first master node)

Login to the server to be used as master and make sure that the _br_netfilter_ module is loaded:

    lsmod | grep br_netfilter
Enable kubelet service.

    sudo systemctl enable kubelet
We now want to initialize the machine that will run the control plane components which includes _etcd_ (the cluster database) and the API Server.

Pull container images:

    sudo kubeadm config images pull
If you have multiple CRI sockets, please use  `--cri-socket`  to select one:

    # CRI-O
    sudo kubeadm config images pull --cri-socket /var/run/crio/crio.sock
    # Containerd
    sudo kubeadm config images pull --cri-socket /run/containerd/containerd.sock
    # Docker
    sudo kubeadm config images pull --cri-socket /run/cri-dockerd.sock 

### B- Bootstrap a Cluster

    sudo kubeadm init \ --pod-network-cidr=10.244.0.0/16
    
**Note**: If _10.244.0.0/16_ is already in use within your network you must select a different pod network CIDR, replacing 10.244.0.0/16 in the above command.

Configure kubectl using commands in the output:

    mkdir -p $HOME/.kube
    sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
Check cluster status:

    kubectl cluster-info

### C- Install Kubernetes network plugin   
In this guide we’ll use [Flannel network plugin](https://github.com/flannel-io/flannel). You can choose any other [supported network plugins](https://kubernetes.io/docs/concepts/cluster-administration/addons/). Flannel is a simple and easy way to configure a layer 3 network fabric designed for Kubernetes.

Download installation manifest.

    wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
If your Kubernetes installation is using custom `podCIDR` (not `10.244.0.0/16`) you need to modify the network to match your one in downloaded manifest.

    $ vim kube-flannel.yml net-conf.json: | { "Network": "10.244.0.0/16", "Backend": { "Type": "vxlan" } }
Then install Flannel by creating the necessary resources.

    kubectl apply -f kube-flannel.yml

  Confirm that all of the pods are running, it may take seconds to minutes before it’s ready.  

    kubectl get pods -n kube-flannel

Check out the systemd unit...it's no longer crashlooping because it has static pods to start
Remember the kubelet starts the static pods, and thus the control plane pods    

    sudo systemctl status kubelet.service 

Confirm master node is ready:

    kubectl get nodes -o wide



## Add worker nodes    
With the control plane ready you can add worker nodes to the cluster for running scheduled workloads.

Run the join command that has been given by running kubeadm init at the control plane.

    kubeadm join k8s-cluster.computingforgeeks.com:6443 \ --token sr4l2l.2kvot0pfalh5o4ik \ --discovery-token-ca-cert-hash sha256:c692fb047e15883b575bd6710779dc2c5af8073f7cab460abd181fd3ddb29a18

You can also generate token and print the join command (on the control plane):

    kubeadm token create --print-join-command

Run below command on the control-plane to see if the node joined the cluster.

    kubectl get nodes
