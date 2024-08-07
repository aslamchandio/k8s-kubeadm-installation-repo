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
- Update Ubuntu
```
sudo apt update -y
sudo apt upgrade -y
sudo apt autoclean -y
sudo hostnamectl set-hostname master01
sudo apt install zip unzip git nano vim wget net-tools vim nano htop tree  -y

printf "\n192.168.1.10  master01\n192.168.3.11  worker01\n192.168.5.12  worker02\n\n" >> /etc/hosts

```
### Run the below steps on the Worker Nodes 
- Update Ubuntu
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
containerd config default | sudo tee /etc/containerd/config.toml   (Note : manually edit and change systemdCgroup to true)
cat /etc/containerd/config.toml

```

### Configuring a cgroup driver

Both the container runtime and the kubelet have a property called "cgroup driver", which is important for the management of cgroups on Linux machines.

Warning:
Matching the container runtime and kubelet cgroup drivers is required or otherwise the kubelet process will fail.

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

```

### Note 

apiserver Ip means Master Node Private IP : 192.168.5.10
Pod Network Cidr                          : 10.244.0.0/16
Service Network Cidr                          : 10.32.0.0/16

## ArgoCD installation 

Install ArgoCD in your Kubernetes cluster following this link - https://argo-cd.readthedocs.io/en/stable/getting_started/

### Requirements

- Installed kubectl command-line tool.
- Have a kubeconfig file (default location is ~/.kube/config).
  CoreDNS. Can be enabled for microk8s by microk8s enable dns && microk8s stop && microk8s start
 
### Step-01:
- Install Argo CD on K8S
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

This will create a new namespace, argocd, where Argo CD services and application resources will live.

```

### Access The Argo CD API Server
- Service Type Load Balancer
```

Change the argocd-server service type to LoadBalancer:

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

```

- Port Forwarding:
```

Kubectl port-forwarding can also be used to connect to the API server without exposing the service.

kubectl port-forward svc/argocd-server -n argocd 8080:443

```

### Step-02:
- ArgoCD install on Linux
```

curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

argocd admin initial-password -n argocd

This password must be only used for first time login. We strongly recommend you update the password using `argocd account update-password`.

argocd admin initial-password -n argocd

```

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/1-Argo1.jpg)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/2-Argo2.jpg)

# Configurations on Jenkins

## Create a First PipeLine on Jenkins

### Step-01: 
- Pipeline
```
Definition : Pipeline Script from SCM
SCM : Git

Repository URL : https://github.com/aslamchandio/project-app.git

Branch Specifier : Change */master to   */main


```

- First Repo : (Dockerfile,Jenkinsfile & app)

### References
- https://github.com/aslamchandio/project-app.git


### Dockerfile
```
FROM nginx
COPY  app  /usr/share/nginx/html

```
### app folder
- app (any code in app folder)
```

```

### Jenkinsfile
- Code
```
node {
    def app

    stage('Clone repository') {
      

        checkout scm
    }

    stage('Build image') {
  
       app = docker.build("aslam24/project-app")                  # Change Repo File
    }

    stage('Test image') {
  

        app.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push image') {
        
        docker.withRegistry('https://registry.hub.docker.com', 'dockerhub') {
            app.push("${env.BUILD_NUMBER}")
        }
    }
    
    stage('Trigger ManifestUpdate') {
                echo "triggering updatemanifestjob"
                build job: 'updatemanifest', parameters: [string(name: 'DOCKERTAG', value: env.BUILD_NUMBER)]
        }
}

```

## Create a Second PipeLine on Jenkins

### Step-01: 
- Pipeline
```

Name : updatemanifest > Pipeline

This project is parameterized    add parameter as string

Name: DOCKERTAG
Default Value: latest

Pipeline


Repository URL : https://github.com/aslamchandio/kubernetesmanifest.git

Branch Specifier : Change */master to   */main

Definition : Pipeline Script from SCM
SCM : Git

```

- Second Repo : (Jenkinsfile & deployment.yaml)

### References
- https://github.com/aslamchandio/kubernetesmanifest.git


### Jenkinsfile
- Code
```
node {
    def app

    stage('Clone repository') {
      

        checkout scm
    }

    stage('Update GIT') {
            script {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    withCredentials([usernamePassword(credentialsId: 'github', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        //def encodedPassword = URLEncoder.encode("$GIT_PASSWORD",'UTF-8')
                        sh "git config user.email aslam.chandio03@gmail.com"
                        sh "git config user.name Aslam Chandio"
                        //sh "git switch master"
                        sh "cat deployment.yaml"
                        sh "sed -i 's+aslam24/project-repo.*+aslam24/project-repo:${DOCKERTAG}+g' deployment.yaml"
                        sh "cat deployment.yaml"
                        sh "git add ."
                        sh "git commit -m 'Done by Jenkins Job changemanifest: ${env.BUILD_NUMBER}'"
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/kubernetesmanifest.git HEAD:main"
      }
    }
  }
}
}


```
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/3-Jenkins1.jpg)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/4-Jenkins2.jpg)

### deployment.yaml
- Code - Deployment & LoadBalancer Service  Manifest
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: firt-deployment
  labels:
    app:  first-deployment

spec:  
  replicas: 3
  selector:
    matchLabels:
      app: first-deployment
      
  template:
    metadata:
      name: first-deployment
      labels:
        app: first-deployment
    spec:
      containers:
        - name: first-deployment
          image: aslam24/project-app:latest
          ports:
            - containerPort: 80
                        

apiVersion: v1 
kind: Service
metadata: 
  name: deployment-lb-service
  labels:
    app: deployment-lb-service

spec:
  type: LoadBalancer
  selector:
    app: first-deployment
  ports:
    - name: http
      port: 80
      targetPort: 80

```

##  Automatic WebHook  
- on Github
...

aslamchandio /project-app   (repo on Github)

setting > Webhooks 

Payload URL : http://18.77.11.12:8080/github-webhook/  or  http://jenkins.chandiolab.site:8080/github-webhook/

Content type : application/json

Which events would you like to trigger this webhook? : just the push event.

active 


## Automatic WebHook
- on Jenkins
...

buildimage > pipeline > Build Triggers  >  GitHub hook trigger for GITScm polling ( check it)

...

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/7-web-hook.jpg)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/8-web-hook-jenkin.jpg)

## Kubernetes Cluster Configurations

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/5-K8S-1.jpg)
![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/6-K8S-2.jpg)

## App Access from domain name

![App Screenshot](https://github.com/aslamchandio/random-resources/blob/main/images/9-App-1.jpg)








































