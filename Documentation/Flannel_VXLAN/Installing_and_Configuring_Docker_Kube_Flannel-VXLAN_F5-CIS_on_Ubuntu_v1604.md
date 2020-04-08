# Installing and Configuring Docker, Kube, Flannel VXLAN and F5 Container Ingress Services (CIS) using ubuntu 16.04

- This document assumes everything is installed under root  --  sudo su -
- This document will also set all Flannel VXLAN interfaces to a MTU of 1450
- As of v16 of the Virtual Edition licenses you no longer need the SDN services license for VXLAN -- the SDN services license in now included with all v16 licenses and beyond  

__TIP:__ Make sure your default route is configured on the correct Kube interface.  
  

<br/>  
  
![Image](https://github.com/grmarxer/kubernetes/blob/master/Documentation/Flannel_VXLAN/diagrams/FlannelVXLAN_Lab_drawing_040620.png)  



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

__Note:___  The subnet mask here must match that of your flannel network -- in this  environment it is 255.255.0.0  

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

## Create a Kubernetes Node for the BIG-IP device (Kube Master Node Only)  

- You need to create a BIG-IP node on your Kube Master for the Kube cluster to know about BIG-IP  

- create the following file below on your Kube master  

- Replace the __VTEP address__ with that of the one specific to your environment  

- replace the __public IP__ to match that of your underlay self-ip in your environment  

- replace the pod CIDR to match your environment which is your overlay self-ip __BUT WITH A MASK OF 255.255.255.0__  
  - Your overlay on BIG-IP was created with a __255.255.0.0__ netmask but the podCIDR must have a mask within that subnet range so use __255.255.255.0__  

  vi f5-kctlr-bigip-node.yaml  
```  
apiVersion: v1  
kind: Node
metadata:
  name: bigip
  annotations:
    # Provide the MAC address of the BIG-IP VXLAN tunnel
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"00:0c:29:0c:7f:32"}'
    flannel.alpha.coreos.com/backend-type: "vxlan"
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    # Provide the IP address you assigned as the BIG-IP VTEP
    flannel.alpha.coreos.com/public-ip: 172.22.10.1
spec:
  #3# Define the flannel subnet you want to assign to the BIG-IP device.
  # Be sure this subnet does not collide with any other Nodes' subnets.
  podCIDR: 10.244.254.0/24  
```

```kubectl create -f f5-kctlr-bigip-node.yaml```  

Verify the node was create correctly.  
__Note:__ The status and version will not be populated.  
```
root@kube8:~# kubectl get nodes
NAME    STATUS    ROLES    AGE   VERSION
bigip   Unknown   <none>   7s
kube8   Ready     master   46m   v1.18.0
kube9   Ready     <none>   29m   v1.18.0  
```

## Download the F5 CIS image on the Kube Master Node  

This step will require a docker login (create one if you do not have one)  

```docker login```  

```docker pull f5networks/k8s-bigip-ctlr```  

## Create a kube "secret" that will contain the BIG-IP username and password (performed Kube Master Node)  

This step is necessary for executing AS3 scripts against the BIG-IP  
__Note:__ Change the username and password below to match your environment  

```kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=admin``` 

Verify the secret has been created using the following command:  

```kubectl get secret -n kube-system```

To view the secret that was created use the following command:  

```kubectl describe secret/bigip-login -n kube-system```

## Create a service account on the Kube Master Node  

This service account is required by the k8s-bigip-ctlr for the AS3 to issue commands against BIG-IP  

```kubectl create serviceaccount k8s-bigip-ctlr -n kube-system```  

## Create a Cluster Role on the Kube Master Node (Required for AS3 permissions)  

In the RBAC API, a role contains rules that represent a set of permissions. Permissions are purely additive (there are no “deny” rules).
A role can be defined within a namespace with a Role, or cluster-wide with a ClusterRole.  

A role binding grants the permissions defined in a role to a user or set of users. It holds a list of subjects (users, groups, or service accounts),
and a reference to the role being granted. Permissions can be granted within a namespace with a RoleBinding, or cluster-wide with a ClusterRoleBinding.  

ClusterRoleBinding may be used to grant permission at the cluster level and in all namespaces.  

```kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr```  


## Create the F5 CIS POD on the Kube System (Performed on the Kube Master)  

Create the following yaml file.  

Be sure to modify the __URL IP__ below to match that of your BIG-IP Underlay Self IP.  

Also be sure that the arg __flannel-name__ matches the VXLAN tunnel created in BIG-IP, and that you define __cluster__ rather than nodeport for pool-member-type.  

vi setup_cis_bigip1.yaml  

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-bigip1-ctlr-deployment
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: k8s-bigip1-ctlr
  # DO NOT INCREASE REPLICA COUNT
  replicas: 1
  template:
    metadata:
      name: k8s-bigip1-ctlr
      labels:
        app: k8s-bigip1-ctlr
    spec:
      # Name of the Service Account bound to a Cluster Role with the required
      # permissions
      serviceAccountName: k8s-bigip-ctlr
      containers:
        - name: k8s-bigip-ctlr
          image: "f5networks/k8s-bigip-ctlr:latest"
          imagePullPolicy: IfNotPresent
          env:
            - name: BIGIP_USERNAME
              valueFrom:
                secretKeyRef:
                  # Replace with the name of the Secret containing your login
                  # credentials
                  name: bigip-login
                  key: username
            - name: BIGIP_PASSWORD
              valueFrom:
                secretKeyRef:
                  # Replace with the name of the Secret containing your login
                  # credentials
                  name: bigip-login
                  key: password
          command: ["/app/bin/k8s-bigip-ctlr"]
          args: [
            # See the k8s-bigip-ctlr documentation for information about
            # all config options
            # https://clouddocs.f5.com/products/connectors/k8s-bigip-ctlr/latest
            "--bigip-username=$(BIGIP_USERNAME)",
            "--bigip-password=$(BIGIP_PASSWORD)",
            "--bigip-url=172.22.10.1",
            "--bigip-partition=kubernetes",
            "--flannel-name=flannel_vxlan",
            "--insecure=true",
            "--pool-member-type=cluster"
            ]
```  
  
Command to create the CIS POD_CIDR  

```kubectl create -f setup_cis_bigip1.yaml``` 

Confirm the CIS POD is up  

```kubectl get pods --all-namespaces -o wide | grep bigip1```  

## Deploy the F5 Hello POD that will be used as Pool Members sent to the BIG-IP via AS3  (Performed on the Kube Master)  

This will create two http application servers as docker containers inside your kube system.  

Create the following yaml file.  

vi 1-f5-hello-world-app-http-deployment.yaml  

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: f5-hello-world
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: f5-hello-world
  template:
    metadata:
      labels:
        app: f5-hello-world
    spec:
      containers:
      - env:
        - name: service_name
          value: f5-hello-world
        image: f5devcentral/f5-hello-world:latest
        imagePullPolicy: Always
        name: f5-hello-world
        ports:
        - containerPort: 8080
          protocol: TCP
```  

Deploy the yaml file created above  

```kubectl create -f 1-f5-hello-world-app-http-deployment.yaml```  

Confirm the POD was created and is up and running  

```kubectl get pods --all-namespaces -o wide | grep hello```  

## Deploy the F5 AS3 Service POD that will tie together the F5-hello PODs (Performed on the Kube Master)  

Create the following yaml file  

vi 2-f5-hello-world-app-http-service.yaml  

```
apiVersion: v1
kind: Service
metadata:
  name: f5-hello-world
  namespace: default
  labels:
    app: f5-hello-world
    cis.f5.com/as3-tenant: AS3
    cis.f5.com/as3-app: A1
    cis.f5.com/as3-pool: web_pool
spec:
  ports:
  - name: f5-hello-world
    port: 8080
    protocol: TCP
    targetPort: 8080
  type: ClusterIP
  selector:
    app: f5-hello-world
```  

Deploy the yaml file created above  

```kubectl create -f 2-f5-hello-world-app-http-service.yaml```  

Confirm the service POD was created and up and running.  

```kubectl get service --all-namespaces -o wide | grep hello```  

## Deploy the F5 AS3 Config Map that will push the AS3 objects created above to BIG-IP (Performed on the Kube Master)  

Create the following yaml file.  

vi 5-f5-as3-config.yaml  

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: f5-as3-declaration
  namespace: default
  labels:
    f5type: virtual-server
    as3: "true"
data:
  template: |
    {
        "class": "AS3",
        "action": "deploy",
        "declaration": {
            "class": "ADC",
            "schemaVersion": "3.10.0",
            "id": "urn:uuid:33045210-3ab8-4636-9b2a-c98d22ab915d",
            "label": "http",
            "remark": "A1 example",
            "AS3": {
                "class": "Tenant",
                "A1": {
                    "class": "Application",
                    "template": "http",
                    "serviceMain": {
                        "class": "Service_HTTP",
                        "virtualAddresses": [
                            "10.1.10.101"
                        ],
                        "pool": "web_pool"
                    },
                    "web_pool": {
                        "class": "Pool",
                        "monitors": [
                            "http"
                        ],
                        "members": [
                            {
                                "servicePort": 8080,
                                "serverAddresses": []
                            }
                        ]
                    }
                }
            }
        }
    }
```  

Deploy the yaml file created above.  

```kubectl apply -f 5-f5-as3-config.yaml```  

If all is working properly you should have a new partition on BIG-IP called "AS3", a virtual server and pool will be created and the pool members will be on port 8080 and green.  

__Note:__ CIS will also create the fdb entries for your kube nodes on BIG-IP.  

```
tmsh show net fdb tunnel flannel_vxlan
------------------------------------------------------------------
Net::FDB
Tunnel         Mac Address        Member                   Dynamic
------------------------------------------------------------------
flannel_vxlan  12:df:a1:5c:5f:fb  endpoint:172.22.10.10%0  no
flannel_vxlan  3a:33:13:2d:01:e6  endpoint:172.22.10.11%0 
```  

The MAC addresses above are the VTEP tunnel endpoints for each of your kube nodes.  You can verify this by looking at the mac address (ifconfig flannel.1) 
on each of your kube nodes or by running kubectl describe node/nodename - on the kube master.  

```
root@kube8:~# kubectl describe node/kube8
Name:               kube8
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=kube8
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VtepMAC":"12:df:a1:5c:5f:fb"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 172.22.10.10
```  

## Delete the F5 AS3 Config Map that was created on the BIG-IP above (Performed on the Kube Master)  

When using AS3 you just can't delete the config map from within Kube, you must send a blank AS3 config map to BIG-IP to remove the configuration objects created above.  

More information on why we need to send a blank AS3 file to BIG-IP to delete a previous AS3 deployment  -- https://clouddocs.f5.com/containers/v2/kubernetes/kctlr-as3-delete-configmap.html  

Create the following yaml file.  

vi 6-f5-delete-config.yaml  

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: f5-as3-declaration
  namespace: default
  labels:
    f5type: virtual-server
    as3: "true"
data:
  template: |
    {
        "class": "AS3",
        "declaration": {
        "class": "ADC",
        "schemaVersion": "3.10.0",
        "id": "33045210-3ab8-4636-9b2a-c98d22ab915d",
        "label":"http",
        "remark": "Remove AS3 declaration",
        "AS3": {
          "class": "Tenant"
        }
      }
    }
```  

Deploy the blank ConfigMap created above.  

```kubectl apply -f 6-f5-delete-config.yaml```  

Delete the configmap from your Kubernetes configuration.  

```kubectl delete configmap f5-as3-declaration```  

If all worked properly the AS partition and all configuration objects will have been deleted from BIG-IP.  

