# Creating an Ingress Resource with CIS and BIG-IP

<br/>  

## Summary  

In this procedure we will provide an overview of the Kubernetes Ingress Resource and provide instructions for configuring a Kubernetes Ingress Resource using BIG-IP and CIS as the ingress controller.  

<br/>  

## Kubernetes Ingress Overview   

External Sources: [What is Kubernetes Ingress?](https://www.ibm.com/cloud/blog/kubernetes-ingress#:~:text=is%20it%20useful%3F-,Kubernetes%20Ingress%20is%20an%20API%20object%20that%20provides%20routing%20rules,each%20service%20on%20the%20node.) and [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/); [F5 CIS Ingress services documentation](https://clouddocs.f5.com/containers/latest/userguide/ingress.html)  

<br/>  

Kubernetes Ingress is an API object that provides routing rules to manage external users' access to the services in a Kubernetes cluster.   
<br/>  


#### Options for exposing applications deployed in Kubernetes  

There are several ways to expose your application to the outside of your Kubernetes cluster, and you'll want to select the appropriate one based on your specific use case. 

There are four main options for exposing an application: ClusterIP, NodePort, LoadBalancer, and Ingress. Each provides a way to expose services and is useful in different situations. A service is essentially a frontend for your application that automatically reroutes traffic to available pods in an evenly distributed way. Services are an abstract way of exposing an application running on a set of pods as a network service. Pods are immutable, which means that when they die, they are not resurrected. The Kubernetes cluster creates new pods in the same node or in a new node once a pod dies. 

Similar to pods and deployments, services are resources in Kubernetes. A service provides a single point of access from outside the Kubernetes cluster and allows you to dynamically access a group of replica pods. 

For internal application access within a Kubernetes cluster, ClusterIP is the preferred method. It is a default setting in Kubernetes and uses an internal IP address to access the service.

To expose a service to external network requests, NodePort, LoadBalancer, and Ingress are possible options. Ingress is the focus of this procedure.  
<br/>  

#### What is Kubernetes Ingress and why is it useful?  

Kubernetes Ingress is an API object that provides routing rules to manage external users' access to the services in a Kubernetes cluster, typically via HTTPS/HTTP. With Ingress, you can easily set up rules for routing traffic without creating a bunch of Load Balancers or exposing each service on the node. This makes it the best option to use in production environments.   

In production environments, you typically need content-based routing, support for multiple protocols, and authentication. Ingress allows you to configure and manage these capabilities inside the cluster.  

Ingress is made up of an Ingress API object and the Ingress Controller. As we have discussed, Kubernetes Ingress is an API object that describes the desired state for exposing services to the outside of the Kubernetes cluster. An Ingress Controller is essential because it is the actual implementation of the Ingress API. An Ingress Controller reads and processes the Ingress Resource information and usually runs as pods within the Kubernetes cluster.  When using BIG-IP and CIS, BIG-IP is the ingress controller and does not run as a pod within the kubernetes cluster.  

An Ingress provides the following:

- Externally reachable URLs for applications deployed in Kubernetes clusters  
- Name-based virtual host and URI-based routing support  
- Load balancing rules and traffic, as well as SSL termination   
<br/>  

#### What is the Ingress Controller?  

If Kubernetes Ingress is the API object that provides routing rules to manage external access to services, Ingress Controller is the actual implementation of the Ingress API. The Ingress Controller is usually a load balancer for routing external traffic to your Kubernetes cluster and is responsible for L4-L7 Network Services.  

Layer 4 (L4) refers to the connection level of the OSI network stack—external connections load-balanced in a round-robin manner across pods. Layer 7 (L7) refers to the application level of the OSI stack—external connections load-balanced across pods, based on requests. Layer 7 is often preferred, but you should select an Ingress Controller that meets your load balancing and routing requirements.  

Ingress Controller is responsible for reading the Ingress Resource information and processing that data accordingly. The following is a sample Ingress Resource:  

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
spec:
  backend:
    serviceName: ServiceName
    servicePort: <Port Number>
```  

As an analogy, if Kubernetes Ingress is a computer, then Ingress Controller is a programmer using the computer and taking action. Furthermore, Ingress Rules act as the manager who directs the programmer to do the work using the computer. Ingress Rules are a set of rules for processing inbound HTTP traffic. An Ingress with no rules sends all traffic to a single default backend service. 

Looking deeper, the Ingress Controller is an application that runs in a Kubernetes cluster and configures an HTTP load balancer according to Ingress Resources. __The load balancer can be a software load balancer running in the cluster or a hardware or cloud load balancer running externally. Different load balancers require different Ingress Controller implementations.__    

<br/>  

## Creating a Ingress Resource using BIG-IP and CIS   

In the following steps we will create a simple deployment using two google images and demonstrate L7 (host and url) load balancing with CIS and BIG-IP using the Ingress Resource.

<br/>  

1.  In this step we will create two deployments in a single yaml file, web and web1.  Web will use the google image `gcr.io/google-samples/hello-app:1.0` and will produce a "version 1" web page when the path "/" is curled.  Web1 will use the google image `gcr.io/google-samples/hello-app:2.0` and will produce a "version 2" web page when the path "/v2" is curled.  
    <br/>  


    [Link to 1-web-deployment.yaml file below](https://github.com/grmarxer/kubernetes/blob/master/Documentation/Ingress_Resource/yaml-files/1-web-deployment.yaml)  

    ```
    vi 1-web-deployment.yaml
    ```  

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web
      namespace: default
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          containers:
          - env:
            - name: web
              value: web
            image: gcr.io/google-samples/hello-app:1.0
            imagePullPolicy: Always
            name: web
            ports:
            - containerPort: 8080
              protocol: TCP

    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: web1
      namespace: default
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: web1
      template:
        metadata:
          labels:
            app: web1
        spec:
          containers:
          - env:
            - name: web1
              value: web1
            image: gcr.io/google-samples/hello-app:2.0
            imagePullPolicy: Always
            name: web1
            ports:
            - containerPort: 8080
              protocol: TCP
    ```  
    ```
    kubectl create -f 1-web-deployment.yaml
    ```  


<br/>  

2. Create a service for the vi 1-web-deployment.yaml deployment.  We will use `ClusterIP` but `NodePort` may also be used.  
    <br/>  


    [Link to 2-web-service.yaml file below](https://github.com/grmarxer/kubernetes/blob/master/Documentation/Ingress_Resource/yaml-files/2-web-service.yaml)  



    ```
    vi 2-web-service.yaml
    ```    

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: web
      namespace: default
      labels:
        app: web
    spec:
      ports:
      - name: web
        port: 8080
        protocol: TCP
        targetPort: 8080
      type: ClusterIP
      selector:
        app: web

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: web1
      namespace: default
      labels:
        app: web1
    spec:
      ports:
      - name: web1
        port: 8080
        protocol: TCP
        targetPort: 8080
      type: ClusterIP
      selector:
        app: web1
    ```   

    ```
    kubectl create -f 2-web-service.yaml
    ```  
<br/>   

3.  Create an Ingress resource for the 2-web-service.yaml created above.  In order to create an Ingress resource using CIS and BIG-IP you need to define the necessary BIG-IP annotations in the Ingress resource.  

    - A complete list of the `BIG-IP Ingress Resource annotations` can be found [here](https://clouddocs.f5.com/containers/latest/userguide/ingress.html#ingress-resources)  

    - The `k8s-bigip1-ctlr-deployment` (setup_cis_bigip1.yaml) file referenced below can be found [here](https://github.com/grmarxer/kubernetes/blob/master/Documentation/Ingress_Resource/yaml-files/setup_cis_bigip1.yaml)  

    When an Ingress resource is created on BIG-IP using CIS the Ingress resource will `always` be created in the BIG-IP partition defined in the BIG-IP CIS (k8s-bigip1-ctlr-deployment) deployment.  In this example the partition is `"--bigip-partition=kubernetes"`  

    There are two ways to assign the VIP IP address using the Ingress resource and we demonstrate the text for both in this example.  Currently the text will assign the specific IP address `172.16.10.100` because `controller-default` is remmed out.    
      - The kubernetes Ingress resource traditionally uses the ingress controllers default IP address as the listener.  This is normally the IP address on the public side interface.  

        BIG-IP/CIS also supports a "default IP" for the Ingress resource listener.  Since BIG-IP/CIS cannot simply know what IP address to use as the "default IP" we must assign the IP address to be used as the "default IP" in the `k8s-bigip1-ctlr-deployment` as an argument.  

          - In this example we have added the argument `"--default-ingress-ip=172.16.10.90"` to our k8s-bigip1-ctlr-deployment.  Thus if the user rems out the `virtual-server.f5.com/ip: 172.16.10.100` annotation and selects the `virtual-server.f5.com/ip: "controller-default"` annotation in yaml file below the `default-ingress-ip` of `172.16.10.90` will be assigned as the VIP listener.  

            __Note:__ You can only use one or the other in a Ingress Resource definition, either specify the VIP specifically or use the controller default  

    __Additional Items to note in the example below__   

    - As you add additional backend resources (pools) you can additional monitors specific to that pool by adding additional json blobs under the `virtual-server.f5.com/health` annotation.  
    - When using `host` in your rules (under spec), you link the monitor to the correct backend service (pool) by using the `path` in the health monitor.  The path in the health monitor links to the backend resource service via the `hostname` and the URL `path`, (Example `web1svc.example.com/v2`)  
      - If you are not using `host` in your rules then you use the serviceName and URL path to link the monitor to the backend resource, (Example web1/v2)  


    <br/>  

    [Link to 3-web-ingress.yaml file below](https://github.com/grmarxer/kubernetes/blob/master/Documentation/Ingress_Resource/yaml-files/3-web-ingress.yaml)  
 
    ```
    vi 3-web-ingress.yaml
    ```  

    ```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: ingress-f5
      namespace: default
      annotations:
        virtual-server.f5.com/partition: "kubernetes"
    #
        virtual-server.f5.com/ip: 172.16.10.100
    #    virtual-server.f5.com/ip: "controller-default"
    #
        virtual-server.f5.com/http-port: "80"
        virtual-server.f5.com/balance: "round-robin"
        virtual-server.f5.com/health: |
          [
            {
              "path": "websvc.example.com/",
              "send": "GET / HTTP/1.1\r\nHost: \r\nConnection: Close:\r\n\r\n",
              "recv": "200",
              "interval": 5,
              "timeout":  10,
              "type": "http"
            },
            {
              "path": "web1svc.example.com/v2",
              "send": "GET /v2 HTTP/1.1\r\nHost: \r\nConnection: Close:\r\n\r\n",
              "recv": "200",
              "interval": 5,
              "timeout":  10,
              "type": "http"
            }
          ]

    spec:
      rules:
      - host: websvc.example.com
        http:
          paths:
          - path: /
            backend:
                serviceName: web
                servicePort: 8080
      - host: web1svc.example.com
        http:
          paths:
          - path: /v2
            backend:
                serviceName: web1
                servicePort: 8080
    ```  


    ```
    kubectl create -f 3-web-ingress.yaml
    ```  
<br/>  

4.  To delete an Ingress Resource all you have to do is delete the Ingress Resource created above   

    <br/>  

    ```
    kubectl delete -f 3-web-ingress.yaml
    ```  




