---
layout: post
title: "Setup Kubernetes locally through ubuntu virtual machines"
---

As a K8S newbie recently I managed to create experimental K8S cluster environment on my windows pc. This post is as a tutorial to show you **how to set up k8s on local machine step by step** 

## Virtual Machines Overview

For those prefer simplest toy, [minikuber](https://github.com/kubernetes/minikube) is the best way to run k8s locally. However here I setup a similar-to-product environment. Basically the cluster has three virtual machines, one as master while other two as slaves:



|     Name | IP | Config | Describe |
| ---- | ---- | ---- | ---- |
| k8s-master | 192.168.19.100 | **2CPU**, 2GB | K8S master node |
| k8s-node1 | 192.168.19.101 | 1CPU, 2GB | K8S slave node |
| k8s-node1 | 192.168.19.102 | 1CPU, 2GB | K8S slave node |

**Please note** for master node it **MUST have at least 2 CPUs**. 

![image](/assets/images/2cpus-vbox.png)



# Create VMs



## Create a K8S-base VM



### Install Ubuntu 



Download **Ubuntu Server 18.04.2 LTS** from [here](https://ubuntu.com/download/server) to local machine, then use [virtualbox](https://www.virtualbox.org/) to create a new virtual machine. Enable OpenSSH server installed in step 7 so we can SSH client to connect to for convenience.


### Disable selinux, swap and firewall

{% highlight bash %}

# Disable selinux
sudo apt install selinux-utils
sudo setenforce 0

# Diable firewall
sudo ufw disable

# Disable swap
sudo swapoff -a
sudo sed -i 's/\s\+swap/\n\#swap/g' /etc/fstab

{% endhighlight %}



### Container runtime setup

Though K8S support multiple container runtimes like Docker, CRI-O and Containerd here I use docker. If you want more info about other container runtime, please refer [here](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

#### Install Docker-CE

```bash
sudo apt update && sudo apt install apt-transport-https ca-certificates curl software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

sudo apt update && sudo apt install docker-ce=18.06.2~ce~3-0~ubuntu
```

#### Add current user into docker group

This is to allow current user run docker command without sudo

```bash

sudo addgroup docker
sudo usermod -aG docker $USER

```

#### Change cgroup driver

```bash

# Set cgroup driver

sudo mkdir -p /etc/docker && sudo touch  /etc/docker/daemon.json

sudo bash -c "cat > /etc/docker/daemon.json" << EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# reload and reboot docker
sudo systemctl daemon-reload
sudo systemctl restart docker

```

#### Check docker 

Run command to make sure docker is up:

```bash
docker version
```
Output should be similar to followings:

```shell
Client:
Version:           18.06.2-ce
API version:       1.38
Go version:        go1.10.3
Git commit:        6d37f41
Built:             Sun Feb 10 03:47:56 2019
OS/Arch:           linux/amd64
Experimental:      false

Server:
Engine:
Version:          18.06.2-ce
API version:      1.38 (minimum version 1.12)
Go version:       go1.10.3
Git commit:       6d37f41
Built:            Sun Feb 10 03:46:20 2019
OS/Arch:          linux/amd64
Experimental:     false
```



#####  !! If you are behind a corporate proxy

Please set proxy for docker daemon following steps below:


```bash

# set proxy for docker daemon

sudo mkdir -p /etc/systemd/system/docker.service.d
sudo touch /etc/systemd/system/docker.service.d/http-proxy.conf
 
sudo bash -c "cat > /etc/systemd/system/docker.service.d/http-proxy.conf" << EOF
[Service]    
Environment="HTTP_PROXY=<your prox>" "HTTPS_PROXY=<your proxy>" "NO_PROXY=localhost,127.0.0.1,::1"
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker

```
Then run command to check if proxy set:

```bash
systemctl show --property=Environment docker 
```



### Install K8S

Run commands to install K8s:

```bash
sudo apt update && sudo apt install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# add k8s repository to apt
sudo mkdir -p /etc/apt/sources.list.d
sudo bash -c "cat > /etc/apt/sources.list.d/kubernetes.list" << EOF
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

# install k8s packages
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

After that run 

```bash
kubeadm version
```

to make sure install is ok.



# change name


```bash

sudo hostnamectl set-hostname XXX

sudo sed -i 's/preserve_hostname: false/preserve_hostname: true/g' /etc/cloud/cloud.cfg 


```

# K8S Master (flannel network)

``` bash
# flunnel must use paramter

sudo kubeadm init --pod-network-cidr=10.244.0.0/16


Write down outpput like:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
kubeadm join 192.168.19.100:6443 --token vwgz01.kqq1e555e4b0ubdv \
    --discovery-token-ca-cert-hash sha256:8cd27109faa8e10b47792dab8c27fd5886ed65b91ff96e8a8cd6602d2ef1eaef
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

# set config var
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
export KUBECONFIG=/etc/kubernetes/admin.conf

# apply flunnel
sudo  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# check all good
sudo kubectl get pods --all-namespaces

```

{% codeblock title:"output" %}
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-5c98db65d4-5p58j             1/1     Running   0          40m
kube-system   coredns-5c98db65d4-7z4dx             1/1     Running   0          40m
kube-system   etcd-k8s.master                      1/1     Running   0          39m
kube-system   kube-apiserver-k8s.master            1/1     Running   0          39m
kube-system   kube-controller-manager-k8s.master   1/1     Running   0          39m
kube-system   kube-flannel-ds-amd64-gdzcl          1/1     Running   0          28m
kube-system   kube-proxy-67wdl                     1/1     Running   0          40m
kube-system   kube-scheduler-k8s.master            1/1     Running   0          39m
{% endcodeblock %}
