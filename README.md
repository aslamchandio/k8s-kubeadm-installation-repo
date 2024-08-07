# Setup a Multi Node Kubernetes Cluster Using Kubeadm

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/Main_Image.jpeg)

## What this does?
Using kubeadm, you can create a minimum viable Kubernetes cluster that conforms to best practices. In fact, you can use kubeadm to set up a cluster that will pass the Kubernetes Conformance tests. kubeadm also supports other cluster lifecycle functions, such as bootstrap tokens and cluster upgrades.

The kubeadm tool is good if you need:

A simple way for you to try out Kubernetes, possibly for the first time.
A way for existing users to automate setting up a cluster and test their application.
A building block in other ecosystem and/or installer tools with a larger scope.

### References
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## If you are using AWS EC2 , you need to allow specific traffic on specific ports as below

### ControlPLan Security Group
```

TCP	Inbound	6443	Kubernetes API server	All
TCP	Inbound	2379-2380	etcd server client API	kube-apiserver, etcd
TCP	Inbound	10250	Kubelet API	Self, Control plane
TCP	Inbound	10259	kube-scheduler	Self
TCP	Inbound	10257	kube-controller-manager	Self
```

### Worker Security Group
```

TCP	Inbound	10250	Kubelet API	Self, Control plane
TCP	Inbound	10256	kube-proxy	Self, Load balancers
TCP	Inbound	30000-32767	NodePort Services†	All
```


### References
- https://kubernetes.io/docs/reference/networking/ports-and-protocols/



### Run the below steps on the Master Node
- Debian-based distributions
```
sudo apt update -y
sudo apt upgrade -y
sudo apt autoclean -y
sudo hostnamectl set-hostname master01
sudo apt install zip unzip git nano vim wget net-tools vim nano htop tree  -y

printf "\n192.168.1.10  master01\n192.168.3.11  worker01\n192.168.5.12  worker02\n\n" >> /etc/hosts

```
### Run the below steps on the Worker Nodes 
- Debian-based distributions
```
sudo apt update -y
sudo apt upgrade -y
sudo apt autoclean -y
sudo hostnamectl set-hostname worker01 
# if you have another worker node 
sudo hostnamectl set-hostname worker02
sudo apt install zip unzip git nano vim wget net-tools vim nano htop tree  -y

printf "\n192.168.1.10  master01\n192.168.3.11  worker01\n192.168.5.12  worker02\n\n" >> /etc/hosts

```

## Install Containerd, Runc, CNI & CRICTL on Master and Worker Nodes

### Install Containerd

```
wget https://github.com/containerd/containerd/releases/download/v1.7.20/containerd-1.7.20-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.20-linux-amd64.tar.gz

sudo wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -P /etc/systemd/system/
ls -al /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

systemctl status containerd

```
### References
- https://github.com/containerd/containerd/blob/main/docs/getting-started.md
- https://github.com/containerd/containerd/blob/main/docs/getting-started.md

### Install Runc

```
wget https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64  -P /tmp/
sudo install -m 755 /tmp/runc.amd64 /usr/local/sbin/runc
```
### Install CNI
```
wget  https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz  -P /tmp/
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin /tmp/cni-plugins-linux-amd64-v1.5.0.tgz

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml   
cat /etc/containerd/config.toml

Notice  SystemdCgroup = false 

```

### Configuring a cgroup driver

Both the container runtime and the kubelet have a property called "cgroup driver", which is important for the management of cgroups on Linux machines.

Warning:
Matching the container runtime and kubelet cgroup drivers is required or otherwise the kubelet process will fail.

SystemdCgroup = false   change to    SystemdCgroup = true

- automatic process

```
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
cat /etc/containerd/config.toml
```

- manual process

```
sudo nano /etc/containerd/config.toml

SystemdCgroup = false   change to    SystemdCgroup = true

sudo systemctl restart containerd
systemctl status containerd
```

### Install CRICTL

```
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.30.1/crictl-v1.30.1-linux-386.tar.gz
sudo tar zxvf crictl-v1.30.1-linux-386.tar.gz -C /usr/local/bin

sudo cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
pull-image-on-create: false
EOF

sudo crictl pull aslam24/nginx-web-oxer:v1
sudo crictl images

unix:///var/run/containerd/containerd.sock

```

## Perform bellow Stps on Master and Worker Nodes


### 1- Disable SWAP 

```
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

```

### 2- Forwarding IPv4 and letting iptables see bridged traffic

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

```

### 3- sysctl params required by setup, params persist across reboots

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot

sudo sysctl --system

```

### 4- Verify that the br_netfilter, overlay modules are loaded by running the following commands:

```
lsmod | grep br_netfilter
lsmod | grep overlay

sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward

```

## Install Kubeadm & Kubectl on Master and Worker Nodes
- Debian-based distributions

### 1- Update the apt package index and install packages needed to use the Kubernetes apt repository:

```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

```

### 2- Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:

```
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```

### 3- Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.30; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).

```
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

```

### 4- Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:

```

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```

### 5- (Optional) Enable the kubelet service before running kubeadm:

```
sudo systemctl enable --now kubelet

```

## Initializing CONTROL-PLANE (Run it on MASTER Node only)

```

sudo kubeadm init --pod-network-cidr "10.244.0.0/16" --service-cidr "10.32.0.0/16" --apiserver-advertise-address=192.168.5.10

## Note

apiserver Ip means Master Node Private IP : 192.168.5.10
Pod Network Cidr                          : 10.244.0.0/16
Service Network Cidr                          : 10.32.0.0/16

```

### Output of above command 

```

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:
                      
```

### Installing "Calico CNI" or "Weave CNI" (Pod-Network add-on):
- Calico CNI
- For Cloud Only (Open TCP/179 BGP Port on controlplane-SG and worker-SG) 

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/calico.yaml -O

kubectl apply -f calico.yaml
kubectl get pods -A

```

### References

- https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-50-nodes-or-less


- Weave CNI
- 6783/tcp,6784/udp for Weavenet CNI open these ports  only on Cloud Provider SG & Firewalls

```

kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml


wget https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml -O weave.yaml 

wget https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

/home/weave/launch.sh

also search 6784

DaemonSet

- --ipalloc-range=10.244.0.0/16

kubectl apply -f weave-daemonset-k8s.yaml

kubectl get pods -A

```

### References

- https://www.weave.works/docs/net/latest/kubernetes/kube-addon/
- https://www.weave.works/docs/net/latest/tasks/ipam/configuring-weave/


### Join worker Nodes (run only on worker nodes)

```
kubeadm join 192.168.5.10:6443 --token toeu11.54j1jcg3xz4e69iy \
        --discovery-token-ca-cert-hash sha256:392b358bc5ce90dd259b8cdda6679fe689b2d2013072d46d8a24d2fa231ccfd9
```

### Create New Token (run only on master node)

```
kubeadm token create --print-join-command 

```

## Install Kubecolor

```
sudo wget https://github.com/hidetatz/kubecolor/releases/download/v0.0.25/kubecolor_0.0.25_Linux_x86_64.tar.gz

  sudo tar zxv -f kubecolor_0.0.25_Linux_x86_64.tar.gz

 ./kubecolor get pods
  ./kubecolor get pods -A
  ./kubecolor get nodes
  ./kubecolor get nodes -o wide
  ./kubecolor get all
  ./kubecolor get all -n kube-system
  ./kubecolor

alias k=/home/ubuntu/
alias k=/home/ubuntu/kubecolor

```

### References

- https://github.com/hidetatz/kubecolor 


## Deployments 

```
k create deployment first-deployment --image aslam24/nginx-web-makaan:v1 

k expose deployment first-deployment --type NodePort --port=80  --name first-deployment-np-service

k scale  deployment first-deployment --replicas 2

k get pods
k get service
k get pods -A
k get all -n kube-system

```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/1-Argo1.jpg)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/2-Argo2.jpg)


![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/3-Jenkins1.jpg)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/4-Jenkins2.jpg)








![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/7-web-hook.jpg)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/8-web-hook-jenkin.jpg)

## Kubernetes Cluster Configurations

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/5-K8S-1.jpg)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/6-K8S-2.jpg)

## App Access from domain name

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/9-App-1.jpg)








































