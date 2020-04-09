# Installing and Configuring Docker, Kube, Calico BGP and F5 Container Ingress Services (CIS) using ubuntu 16.04

- This document assumes everything is installed under root  --  sudo su -  

__TIP:__ Make sure your default route is configured on the correct Kube interface.  
  

<br/>  

![Image](https://github.com/grmarxer/kubernetes/blob/master/Documentation/Calico_BGP/diagrams/Calico_BGP_lab_%20drawing_040720.png)  
 


<br/>  

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

<br/>  

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

<br/>  

## Disable SWAP Memory on kube nodes
Disable swap in current session  
```swapoff -a```

Disable swap permanently - comment out the "swap" line in this file  
```vi /etc/fstab```

<br/>  

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

## Initialize Kube (Performed on the Kube Master ONLY)

If you are using a IP network other than the default of 192.168.0.0 you must include the cidr range in the init step.
 __( --pod-network-cidr=172.16.0.0/16 )__

Also if you are connecting to an interface other than the primary interface on the Kube Master Ubuntu node you must call that out during the INIT.  If you do not, Kube will use the Primary Ubuntu ethernet interface for API traffic
__( --apiserver-advertise-address=172.18.10.10 )__

```kubeadm init --apiserver-advertise-address=172.18.10.10 --pod-network-cidr=172.16.0.0/16```

To start using your cluster, you need to run the following as a regular user:

```mkdir -p $HOME/.kube```  
```cp -i /etc/kubernetes/admin.conf $HOME/.kube/config```  
```chown $(id -u):$(id -g) $HOME/.kube/config``` 


Copy the following link, specific to your installation, to be used later to join the worker Kube nodes to the master node.
Example from my specific environment -- output of kubeadm init:
```
kubeadm join 172.18.10.10:6443 --token yiqzpj.0q898oj9wuiy0hja \
    --discovery-token-ca-cert-hash sha256:dc84cfc0d02d736e0358e9ecd2ec21879f81905cb892fb9ad91dd3a54d01e921
```
If you lost the join token run this command on the kube master  
```kubeadm token create --print-join-command```  

<br/>  

## Install the Network POD Calico into Kube (Performed on the Kube Master ONLY)  

Installing with the Kubernetes API datastore—50 nodes or less.  
Install Calico Version 3.10  
```curl https://docs.projectcalico.org/v3.10/manifests/calico.yaml -O```  

If you are using pod CIDR 192.168.0.0/16, skip the next step. If you are using a different pod CIDR, use the following commands to set the pod CIDR that matches your environment in the calico.yaml file.  

Changing the Calico default IP block to the block you used during Kube Init  

```POD_CIDR="<172.16.0.0/16>" sed -i -e "s?192.168.0.0/16?$POD_CIDR?g" calico.yaml```  

Apply the Calico.yaml file  

```kubectl apply -f calico.yaml```  

__Note:__ Most likely the IP CIDR block command above will not work correctly and you will need to use calicoctl to enter the correct CIDR Block.  Well will confirm the CIDR block is correct in future steps.  

<br/>  

## Join worker node to the kube master  (Performed on Kube Worker Nodes)
On each of your worker nodes run the link below specific to your environment that you copied above.
```
kubeadm join 172.18.10.10:6443 --token yiqzpj.0q898oj9wuiy0hja \
    --discovery-token-ca-cert-hash sha256:dc84cfc0d02d736e0358e9ecd2ec21879f81905cb892fb9ad91dd3a54d01e921
```
If you lost the join token run this command on the master  
```kubeadm token create --print-join-command```

Run the following commands on the master node to ensure all nodes are up and ready and running flannel  
```kubectl get nodes```  
```kubectl get pods --all-namespaces -o wide```

<br/>  

##  Install Calicoctl, the Command line tool for Calico (Performed on the Kube Master ONLY)  

Installing calicoctl as a binary on a single host  

```cd /usr/local/bin/```

```curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.10.1/calicoctl```

  
 ```chmod +x calicoctl```  

Configure calicoctl to connect to your datastore.  

__Note:__ Make sure you change the __kubeconfig__ path to match your environment.  Since my environment was installed under root my kubeconfig path would be "/root/.kube/config"  

Create the calico directory inside /etc/ and the calicoctl.cfg file.  

```mkdir /etc/calico```  

```vi /etc/calico/calicoctl.cfg```  

```
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "$HOME/.kube/config"  
```  

Confirm calicoctl installation  

```calicoctl version```  

```
Client Version:    v3.10.1
Git commit:        4aaff8e9
Cluster Version:   v3.10.0
Cluster Type:      k8s,bgp,kdd
```  

Commands to change calico IP range if not set properly above.  

```calicoctl get ipPool -o wide``` 

If the CIDR range provided with the above command is not correct follow the following steps.  

```calicoctl delete ippool default-ipv4-ippool```  

Make sure the CIDR range you enter below matches your environment.  

```
calicoctl create -f - << EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 172.16.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: true
  disabled: false
  nodeSelector: all()
EOF
```

Verify the CIDR range is now correct.

```calicoctl get ipPool -o wide```  

Reboot your master and worker kube nodes and make sure everything under comes up with the correct IP's.  

```kubectl get pods --all-namespaces -o wide```  

<br/>  

## Preparing BIG-IP to Connect to the KUBE cluster for CIS using Calico BGP (Performed on BIG-IP)  

1. v16 license or newer is required on your BIG-IP  
2. Create a partition "kubernetes" with a default RD of zero  
3. Enable BGP on Route Domain 0
4. If you are using a BIG-IP version prior to 14.0, before you can use the Configuration utility, you must enable the framework using the BIG-IP command line.   
  
        From the CLI, type the following command: touch /var/config/rest/iapps/enable.  
5. Download and install the latest AS3 RPM file on BIG-IP  ( f5-appsvcs-3.18.0-4.noarch.rpm )  When writing this document v3.18 was the latest.  
  
       https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/installation.html  
         
       https://github.com/F5Networks/f5-appsvcs-extension/releases  
6. Be sure to have port-lockdown set to allow all on the Self IP for which BIG-IP will advertise BGP  

<br/> 

##  Setting up BGP on BIG-IP  

From Bash  

```imish```  

Once inside the imish shell.  

```enable```  

```config terminal```  

Change the neighbor statements below to peer with each of your Kube Master and Worker Nodes IP's.  

```
router bgp 64512
neighbor calico-k8s peer-group
neighbor calico-k8s remote-as 64512
neighbor 172.18.10.10 peer-group calico-k8s
neighbor 172.18.10.11 peer-group calico-k8s
write 
end  
```  

```show ip bgp summary```  

```
VE5-13-1-kube.com[0]#show ip bgp summary
BGP router identifier 172.18.10.1, local AS number 64512
BGP table version is 1
0 BGP AS-PATH entries
0 BGP community entries

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.18.10.10    4 64512       0       0        0    0    0          Active
172.18.10.11    4 64512       0       0        0    0    0          Active

Total number of neighbors 2  
```  
<br/>   

## Setting up BGP on Calico (Kube Master Node Only)  

```
cat << EOF | calicoctl create -f -
 apiVersion: projectcalico.org/v3
 kind: BGPConfiguration
 metadata:
   name: default
   creationTimestamp: null
 spec:
   logSeverityScreen: Info
   nodeToNodeMeshEnabled: true
   asNumber: 64512
EOF
```  

Be sure to change the PeerIP below to match that of the BIG-IP self IP.  

```
cat << EOF | calicoctl create -f -
 apiVersion: projectcalico.org/v3
 kind: BGPPeer
 metadata:
   name: bigip1
 spec:
   peerIP: 172.18.10.1
   asNumber: 64512
EOF
```

<br/>  

## Confirm BGP is working properly (Performed on BIG-IP)  

From Bash  

```imish```  

Once inside the imish shell.  

```enable```  

```show ip route```  

You should see the BGP routes for the CIDR block you configured in Calico as BGP routes in the routing table.  

```
VE5-13-1-kube.com[0]#show ip route
Codes: K - kernel, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default

C       127.0.0.1/32 is directly connected, lo
C       127.1.1.254/32 is directly connected, tmm
B       172.16.85.192/26 [200/0] via 172.18.10.11, /Kubernetes/vnic3, 00:41:02
B       172.16.190.64/26 [200/0] via 172.18.10.10, /Kubernetes/vnic3, 00:40:57
C       172.18.10.0/24 is directly connected, /Kubernetes/vnic3
```

<br/>  

## Download the F5 CIS image on the Kube Master Node  

This step will require a docker login (create one if you do not have one)  

```docker login```  

```docker pull f5networks/k8s-bigip-ctlr```  

<br/>  

## Create a kube "secret" that will contain the BIG-IP username and password (performed Kube Master Node)  

This step is necessary for executing AS3 scripts against the BIG-IP  
__Note:__ Change the username and password below to match your environment  

```kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=admin``` 

Verify the secret has been created using the following command:  

```kubectl get secret -n kube-system```

To view the secret that was created use the following command:  

```kubectl describe secret/bigip-login -n kube-system```

<br/>  

## Create a service account on the Kube Master Node  

This service account is required by the k8s-bigip-ctlr for the AS3 to issue commands against BIG-IP  

```kubectl create serviceaccount k8s-bigip-ctlr -n kube-system```  

<br/>  

## Create a Cluster Role on the Kube Master Node (Required for AS3 permissions)  

In the RBAC API, a role contains rules that represent a set of permissions. Permissions are purely additive (there are no “deny” rules).
A role can be defined within a namespace with a Role, or cluster-wide with a ClusterRole.  

A role binding grants the permissions defined in a role to a user or set of users. It holds a list of subjects (users, groups, or service accounts),
and a reference to the role being granted. Permissions can be granted within a namespace with a RoleBinding, or cluster-wide with a ClusterRoleBinding.  

ClusterRoleBinding may be used to grant permission at the cluster level and in all namespaces.  

```kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr```  

<br/>  

## Create the F5 CIS POD on the Kube System (Performed on the Kube Master)  

Create the following yaml file.  

Be sure to modify the __bigip-url__ below to match that of your BIG-IP Self-IP.  

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
            "--bigip-url=172.18.10.1",
            "--bigip-partition=kubernetes",
            "--insecure=true",
            "--pool-member-type=cluster"
            ]
```  
  
Command to create the CIS POD_CIDR  

```kubectl create -f setup_cis_bigip1.yaml``` 

Confirm the CIS POD is up  

```kubectl get pods --all-namespaces -o wide | grep bigip1```  

<br/>  

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

<br/>  

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

<br/>  

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

If all is working properly you should have a new partition on BIG-IP called "AS3", a virtual server created with a pool attached that has members on port 8080 which are green.  

<br/>  

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

If all worked properly the AS3 partition and all configuration objects will have been deleted from BIG-IP.  






