# Installing and Configuring Docker, Kube, Flannel VXLAN and F5 Container Ingress Services (CIS) using ubuntu 16.04

- This document assumes everything is installed under root  --  sudo su -
- This document will also set all Flannel VXLAN interfaces to a MTU of 1450
- As of v16 of the Virtual Edition licenses you no longer need the SDN services license for VXLAN -- the SDN services license in now included with all v16 licenses and beyond  

__TIP:__ Make sure your default route is configured on the correct Kube interface.  


! [Lab Diagram](https://github.com/grmarxer/kubernetes/blob/master/Documentation/Flannel_VXLAN/diagrams/Kube_FlannelVXLAN_CIS_Lab_drawing_040620.png)


## Preparing the Ubuntu Server

__Make sure you are running Ubuntu 16.04 on your Kube nodes__  
```lsb_release -a```

__Update Ubuntu__  
```apt-get update -y```  
```apt-get upgrade -y```

__Disable firewall ubuntu__  
```ufw disable```

__selinux is not installed by default__  
```sestatus``` should fail  

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
Create the underlay network vlan (the vlan name in this environment is vnic5 and bound to interface 1.1)  
```tmsh create net vlan vnic5 interfaces add { 1.1 }```

Create the underlay network self-ip (in this environment the self-ip is 172.22.10.1/24)  
```tmsh create net self 172.22.10.1 address 172.22.10.1/24 allow-service all vlan vnic5```

## Initialize Kube (Performed on the Kube Master ONLY)

If you are using a IP network other than the default of 192.168.0.0 you must include the cidr range in the init step.
 __( --pod-network-cidr=10.244.0.0/16 )__

__NOTE:__ Flannel uses a default network of 10.244.0.0/16

Also if you are connecting to an interface other than the primary interface on the Kube Master Ubuntu node you must call that out during the INIT.  If you do not, Kube will use the Primary Ubuntu ethernet interface for API traffic
__( --apiserver-advertise-address=172.22.10.10 )__

```kubeadm init --apiserver-advertise-address=172.22.10.10 --pod-network-cidr=10.244.0.0/16```

To start using your cluster, you need to run the following as a regular user:

```mkdir -p $HOME/.kube```  
 ```cp -i /etc/kubernetes/admin.conf $HOME/.kube/config```  
```chown $(id -u):$(id -g) $HOME/.kube/config``` 


Copy the following link, specific to your installation, to be used later to join the worker Kube nodes to the master node.
Example from my specific environment -- output of kubeadm init:
```
kubeadm join 172.22.10.10:6443 --token olpkaj.96bl7tt60pil4rxx \
    --discovery-token-ca-cert-hash sha256:ffa74d2b1d37229604672244a8f00d926dc7b0a2a6789d5417ebe08cf759df87
```
If you lost the join token run this command on the kube master  
```kubeadm token create --print-join-command```  

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
        - --iface=ens192
```
Flannel will bind by default to the first interface on your kube nodes.  In my environment the first interface is ens160 which is my management interface. I want flannel to bind to my underlay network interface which is __ens192__ so I added the __iface__ argument above under containers.

By default flannel uses the 10.244.0.0/16 network.  I did not modify this in my environment.  If you want flannel to use a network other than the default of 10.244.0.0/16 you modify it here:
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
```kubectl create -f kube-flannel.yaml```  

Verify flannel is running on your master node  
```kubectl get pods --all-namespaces -o wide | grep flannel```  
  
kube-system   kube-flannel-ds-amd64-dhrfl     1/1     Running   0   73s   172.22.10.10   kube8   <none>   <none>


## Join worker node to the kube master  (Performed on Kube Worker Nodes)
On each of your worker nodes run the link below specific to your environment that you copied above.
```
kubeadm join 172.22.10.10:6443 --token olpkaj.96bl7tt60pil4rxx \
    --discovery-token-ca-cert-hash sha256:ffa74d2b1d37229604672244a8f00d926dc7b0a2a6789d5417ebe08cf759df87
```
If you lost the join token run this command on the master  
```kubeadm token create --print-join-command```

Run the following commands on the master node to ensure all nodes are up and ready and running flannel  
```kubectl get nodes```  
```kubectl get pods --all-namespaces -o wide```

## Preparing BIG-IP to Connect to the KUBE cluster for CIS using Flannel VXLAN (Performed on BIG-IP)  
1. v16 license or newer is required on your BIG-IP  
2. Create a partition "kubernetes" with a default RD of zero  
3. If you are using a BIG-IP version prior to 14.0, before you can use the Configuration utility, you must enable the framework using the BIG-IP command line.   
  
        From the CLI, type the following command: touch /var/config/rest/iapps/enable.  
4. Download and install the latest AS3 RPM file on BIG-IP  ( f5-appsvcs-3.18.0-4.noarch.rpm )  When writing this document v3.18 was the latest.  
  
       https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html  
         
       https://github.com/F5Networks/f5-appsvcs-extension/releases  

## Setting up BIG-IP Overlay network for Flannel VXLAN (Performed on BIG-IP)  
Create VXLAN tunnel profile on BIG-IP  

__Note:__ We use port 8472 as most linux systems default port for VXLAN is 8472  

```tmsh create net tunnels vxlan fl-vxlan port 8472 flooding-type none```  

Create the VXLAN tunnel on BIG-IP using the tunnel name of __flannel_vxlan__ and the local address of your BIG-IP underlay network. 

__Note:__ The name __flannel_vxlan__ has significance and will be an argument in your CIS deployment.  They must match if you decide to alter the name from what I have here.  
I also used a MTU of 1450 which is the default for most Flannel systems Flannel.  You can use PMTU but I wanted to hard cord my MTU in my environment.  
```tmsh create net tunnels tunnel flannel_vxlan key 1 profile fl-vxlan local-address 172.22.10.1 mtu 1450```


Create the BIG-IP Overlay VXLAN tunnel self-ip.  

__NOTE:___  The subnet mask here must match that of your flannel network -- in this  environment it is 255.255.0.0  

```tmsh create net self 10.244.254.1 address 10.244.254.1/16 allow-service none vlan flannel_vxlan```


Find the VTEP MAC address for the self-ip you just created.  You can either use the tmsh command below or ifconfig flannel_vxlan to find the MAC address.  

__Note:__ You will need this MAC address later when you create your BIG-IP Kube node on your Kube Master.  

```tmsh show net tunnels tunnel flannel_vxlan all-properties```
```
-------------------------------------------------
Net::Tunnel: flannel_vxlan
-------------------------------------------------
MAC Address                     00:0c:29:0c:7f:32
Interface Name                      flannel_vxlan
```
```[root@flannel-bigip-15-1-0-2:Active:Standalone] config # ifconfig flannel_vxlan
flannel_vxlan: flags=4291<UP,BROADCAST,RUNNING,NOARP,MULTICAST>  mtu 1450
        ether 00:0c:29:0c:7f:32  txqueuelen 1000  (Ethernet)  
```