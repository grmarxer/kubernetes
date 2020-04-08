# Installing kube, Flannel VXLAN and Configuring F5 Container Ingress Services (CIS) using ubuntu 16.04

- This document assumes everything is installed under root  --  sudo su -
- This document will also set all Flannel VXLAN interfaces to a MTU of 1450
- As of v16 of the Virtual Edition licenses you no longer need the SDN services license for VXLAN -- the SDN services license in now included with all v16 licenses and beyond

## Preparing the Ubuntu Server

__Make sure you are running Ubuntu 16.04 on your Kube nodes__  
```lsb_release -a```

__Update Ubuntu__  
```apt-get update -y```  
```apt-get upgrade -y```

__Disable firewall ubuntu__  
```ufw disable```

__selinux not installed by default__  
```sestatus should fail```

__NTP installed by default__  
Verify NTP is in sync  
```timedatectl```

## Install Docker
```apt-get update && apt-get install apt-transport-https ca-certificates curl software-properties-common```

```curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -```
```
 add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
```

```apt-get update && apt-get install docker-ce=18.06.2~ce~3-0~ubuntu -y```
```
 cat > /etc/docker/daemon.json <<EOF
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
```mkdir -p /etc/systemd/system/docker.service.d```  
```systemctl daemon-reload```  
```systemctl restart docker```  

## Disable SWAP Memory on kube nodes
Disable swap in current session  
```swapoff -a```

Disable swap permanently - comment out the "swap" line in this file  
```vi /etc/fstab```

## Install Kube

```apt-get update &&  apt-get install -y apt-transport-https curl```

```curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg |  apt-key add -```

```
cat <<EOF |  tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```apt-get update```  
```apt-get install -y kubelet kubeadm kubectl```  
```apt-mark hold kubelet kubeadm kubectl```  

<br/><br/>
## *Repeat the steps above for all nodes in your kube cluster* 
<br/><br/>



## Configure BIG-IP to support the underlay network (reference the diagram for this procedure)
Create the underlay network vlan (the vlan name in this  is vnic5 and bound to interface 1.1)
#tmsh create net vlan vnic5 interfaces add { 1.1 }

Create the underlay network self-ip (in this  the self-ip is 172.22.10.1/24)
#tmsh create net self 172.22.10.1 address 172.22.10.1/24 allow-service all vlan vnic5

## Initialize Kube (Performed on the Master ONLY)

If you are using a IP network other than the default of 192.168.0.0 you must include the cidr range in the init step.
 ( --pod-network-cidr=10.244.0.0/16 )

NOTE: Flannel uses a default network of 10.244.0.0/16

Also if you are connecting to an interface other than the primary interface on the Kube Master Ubuntu node you must call that out during the INIT.  If you do not Kube will use the Primary Ubuntu ethernet interface for API traffic
( --apiserver-advertise-address=172.22.10.10 )

#kubeadm init --apiserver-advertise-address=172.22.10.10 --pod-network-cidr=10.244.0.0/16

To start using your cluster, you need to run the following as a regular user:

#mkdir -p $HOME/.kube
#cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#chown $(id -u):$(id -g) $HOME/.kube/config

Copy the following link specific to your installation to be used later to join the worker Kube nodes to the master node.
Example from my specific output of kubeadm init
```
kubeadm join 172.22.10.10:6443 --token olpkaj.96bl7tt60pil4rxx \
    --discovery-token-ca-cert-hash sha256:ffa74d2b1d37229604672244a8f00d926dc7b0a2a6789d5417ebe08cf759df87
```
If you lost the join token run this command on the master
#kubeadm token create --print-join-command

## To Install the Network POD Flannel into Kube (Performed on the Master ONLY)

Copy this yaml file to your master node.  I recommend opening the link below in your browser and creating a file on your master node using a text editor called kube-flannel.yaml

https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel.yml

There are two sections of this file you may want to edit to match your environment.  For this specific environment I have the following:
```
      containers:
     - name: kube-flannel
        image: quay.io/coreos/flannel:v0.12.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
      __- --iface=ens192__
```
Flannel will bind by default to the first interface on your kube nodes.  In my environment the first interface is ens160 which is my management interface. I want flannel to bind to my underlay network interface which is ens192 so I added the argument above under containers.

By default flannel uses the 10.244.0.0/16 network by default.  I did not modify this in my .  If you want flannel to use a network other than the default of 10.244.0.0/16 you modify it here:
```
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```
Command to create flannel deployment
#kubectl create -f kube-flannel.yaml

Verify flannel is running on your master node
#kubectl get pods --all-namespaces -o wide | grep flannel
```
kube-system   kube-flannel-ds-amd64-dhrfl     1/1     Running   0   73s   172.22.10.10   kube8   <none>   <none>
```

## Join worker node to the kube master  (Performed on Worker Nodes)
On each of your worker nodes run the link below specific to your  that you copied above
```
kubeadm join 172.22.10.10:6443 --token olpkaj.96bl7tt60pil4rxx \
    --discovery-token-ca-cert-hash sha256:ffa74d2b1d37229604672244a8f00d926dc7b0a2a6789d5417ebe08cf759df87
```
If you lost the join token run this command on the master
#kubeadm token create --print-join-command

Run the following commands on the master node to ensure all nodes are up and ready and running flannel
#kubectl get nodes
#kubectl get pods --all-namespaces -o wide