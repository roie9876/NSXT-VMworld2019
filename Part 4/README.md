# NSX-T & K8S - PART 4
[Home Page](https://github.com/dumlutimuralp/nsx-t-k8s)

# Table Of Contents

[Current State](#Current-State)   
[NSX Container Plugin (NCP) Installation](#NSX-Container-Plugin-Installation)  
[NSX Node Agent Installation](#NSX-Node-Agent-Installation)  
[Test Workload Deployment](#Test-Workload-Deployment)

# Current State
[Back to Table of Contents](#Table-Of-Contents)

## K8S Cluster

Previously in Part 3, K8S cluster has successfully been formed using kubeadm. 

<pre><code>
root@master:/home/localadmin# kubectl get node
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   2d22h   v1.14.2
node01   Ready          <none>   2d22h   v1.14.2
node02   Ready          <none>   2d18h   v1.14.2

</code></pre>

The namespaces that are provisioned by default can be seen using the following kubectl command.

<pre><code>
root@master:/home/localadmin# kubectl get ns
NAME              STATUS   AGE
default           Active   2d22h
kube-node-lease   Active   2d22h
kube-public       Active   2d22h
kube-system       Active   2d22h
root@master:/home/localadmin#

</code></pre>

To see which infrastructure Pods are automatically provisioned during the initialization of K8S cluster, following command can be used.

<pre><code>
root@master:/home/localadmin# kubectl get pods --all-namespaces
NAMESPACE     NAME                             READY   STATUS              RESTARTS   AGE
kube-system   coredns-584795fc57-hwxrv         0/1     ContainerCreating   0          31s
kube-system   coredns-584795fc57-zpjh6         0/1     ContainerCreating   0          6s
kube-system   etcd-master                      1/1     Running             0          2d22h
kube-system   kube-apiserver-master            1/1     Running             0          2d22h
kube-system   kube-controller-manager-master   1/1     Running             0          2d22h
kube-system   kube-proxy-9bvvf                 1/1     Running             1          2d18h
kube-system   kube-proxy-9vhcf                 1/1     Running             1          2d22h
kube-system   kube-proxy-xpkhk                 1/1     Running             0          2d22h
kube-system   kube-scheduler-master            1/1     Running             0          2d22h


</code></pre>

_**Notice "coredns-xxx" Pods are stuck in "ContainerCreating" phase, the reason is although kubelet agent on K8S worker Node sent a request to NSX-T CNI Plugin module to start provisioning the individual network interface for these Pods, since the NSX Node Agent is not installed on the K8S worker nodes yet (Nor the NSX Container Plugin for attaching NSX-T management plane to K8S API) , kubelet can not move forward with the Pod creation.**_


We can learnd it from the description of the coredns pod:  

<pre><code>
root@master:/home/localadmin# kubectl describe pod coredns-584795fc57-hwxrv  -n kube-system
Name:               coredns-584795fc57-hwxrv
Namespace:          kube-system
Priority:           2000000000
PriorityClassName:  system-cluster-critical
Node:               node02/192.168.110.73
Start Time:         Fri, 25 Oct 2019 11:22:00 +0300
Labels:             k8s-app=kube-dns
                    pod-template-hash=584795fc57
Annotations:        <none>
Status:             Pending
IP:
Controlled By:      ReplicaSet/coredns-584795fc57
Containers:
  coredns:
    Container ID:
    Image:         k8s.gcr.io/coredns:1.3.1
    Image ID:
    Ports:         53/UDP, 53/TCP, 9153/TCP
    Host Ports:    0/UDP, 0/TCP, 0/TCP
    Args:
      -conf
      /etc/coredns/Corefile
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Limits:
      memory:  170Mi
    Requests:
      cpu:        100m
      memory:     70Mi
    Liveness:     http-get http://:8080/health delay=60s timeout=5s period=10s #success=1 #failure=5
    Readiness:    http-get http://:8080/health delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /etc/coredns from config-volume (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from coredns-token-8xg74 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      coredns
    Optional:  false
  coredns-token-8xg74:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  coredns-token-8xg74
    Optional:    false
QoS Class:       Burstable
Node-Selectors:  beta.kubernetes.io/os=linux
Tolerations:     CriticalAddonsOnly
                 node-role.kubernetes.io/master:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason                  Age               From               Message
  ----     ------                  ----              ----               -------
  Normal   Scheduled               45s               default-scheduler  Successfully assigned kube-system/coredns-584795fc57-hwxrv to node02
  Warning  FailedCreatePodSandBox  44s               kubelet, node02    Failed create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "8958bc4b62fa3292d8209e423b34d49d78fa9749b153db733b7ffcd30bae3ac2" network for pod "coredns-584795fc57-hwxrv": NetworkPlugin cni failed to set up pod "coredns-584795fc57-hwxrv_kube-system" network: Failed to connect to nsx_node_agent: [Errno 111] Connection refused, failed to clean up sandbox container "8958bc4b62fa3292d8209e423b34d49d78fa9749b153db733b7ffcd30bae3ac2" network for pod "coredns-584795fc57-hwxrv": NetworkPlugin cni failed to teardown pod "coredns-584795fc57-hwxrv_kube-system" network: Failed to connect to nsx_node_agent: [Errno 111] Connection refused]
  Normal   SandboxChanged          2s (x4 over 44s)  kubelet, node02    Pod sandbox changed, it will be killed and re-created.
</code></pre>






# NSX Container Plugin Installation
[Back to Table of Contents](#Table-Of-Contents)

Once again, NSX Container Plugin (NCP) image file is in the NSX container folder that was copied to each K8S node will be used in this section. 

## Load The Docker Image for NSX NCP (and NSX Node Agent) on K8S Nodes

For the commands below, "sudo" can be used with each command or privilege can be escalated to root by using "sudo -H bash" in advance.

On each K8S node, navigate to "/home/vmware/nsx-container-2.4.1.13515827/Kubernetes" folder then execute the following command to load respective image to the local Docker repository of each K8S Node. 

_**NSX Container Plugin (NCP) and NSX Node Agent Pods use the same container image.**_

## Install NCP image on all masters and Workers 

CNI is a Cloud Native Computing Foundation (CNCF) project. It is a set of specifications and libraries to configure network interfaces in Linux containers. It has a pluggable architecture hence third party plugins are supported.

NSX-T CNI Plugin comes within the "NSX-T Container" package. The package can be downloaded (in _.zip_ format) from Downloads page for NSX-T, shown below.

![](2.png)

In the current NSX-T version (2.5.0) , the zip file is named as "**nsx-container-2.5.0.14628220**" . 

* Extract the zip file to a folder. 

![](2019-24-10-19-06-21.png)

* Use SCP/SSH to copy the folder to the Ubuntu node. Winscp is used as the SCP tool on Windows client and the folder is copied to /home/vmware location on Ubuntu node.

* **For all the installation steps mentioned below and following sections in this guide root level access will be used.**

* Escalate to root in the shell


<pre><code>
vmware@k8s-master:~$ <b>sudo -H bash</b>
[sudo] password for vmware:
root@k8s-master:/home/vmware#
</code></pre>

* On the Ubuntu shell, navigate to "/home/vmware/nsx-container-2.5.0.14628220/Kubernetes/" folder, and then install NCP image shown below.

<pre><code>
root@k8s-master:/home/vmware/nsx-container-2.5.0.14628220/Kubernetes/# 
<b>docker load  -i nsx-ncp-ubuntu-2.5.0.14628220.tar</b>
Loaded image: registry.local/2.5.0.14628220/nsx-ncp-ubuntu:latest
</code></pre>

then we need to tag the ncp image as nsx-ncp:
<pre><code>
docker tag registry.local/2.5.0.14628220/nsx-ncp-ubuntu:latest nsx-ncp
</code></pre>

<pre><code>
root@k8s-master:/home/vmware/nsx-container-2.5.0.14628220/Kubernetes# <b> docker tag registry.local/2.5.0.14628220/nsx-ncp-ubuntu:latest nsx-ncp:latest</b>
root@k8s-master:/home/vmware/nsx-container-2.5.0.14628220/Kubernetes#
</code></pre>

Verify the image name has changed. 

<pre><code>
root@master:/home/localadmin# docker images
REPOSITORY                                                         TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-apiserver                                          v1.14.8             1e94481e8f30        9 days ago          209MB
k8s.gcr.io/kube-proxy                                              v1.14.8             849af609e0c6        9 days ago          82.1MB
k8s.gcr.io/kube-controller-manager                                 v1.14.8             36a8001a79fd        9 days ago          158MB
k8s.gcr.io/kube-scheduler                                          v1.14.8             f1e3e5f9f93e        9 days ago          81.6MB
registry.local/2.5.0.14628220/nsx-ncp-ubuntu                       latest              40aae9a4aeda        6 weeks ago         744MB
nsx-ncp                                                            latest              40aae9a4aeda        6 weeks ago         744MB
284299419820.dkr.ecr.us-west-2.amazonaws.com/ws-client             v1.0.0              d87c306ddd38        6 weeks ago         79.9MB
284299419820.dkr.ecr.us-west-2.amazonaws.com/k8s-cluster-manager   v0.7.3              c9225cfd6d91        6 weeks ago         148MB
k8s.gcr.io/coredns                                                 1.3.1               eb516548c180        9 months ago        40.3MB
k8s.gcr.io/etcd                                                    3.3.10              2c4adeb21b4f        10 months ago       258MB
tiswanso/install-cni                                               v0.1-dev            84a707a64677        11 months ago       42.8MB
k8s.gcr.io/pause                                                   3.1                 da86e6ba6ca1        22 months ago       742kB

</code></pre>

## Confiure the NCP for K8s Cluster 


In NCP version 2.5 the process of installation and configuration of the NCP improved a lots by automated the all proecess.
The only things that we need to do is configure one yaml file name ncp-ubuntu.yaml. we can find this file in the folder of the NCP installation. 

![](2019-10-25-11-33-56.png)

example of thee original file (before we modiey it) can be found in this link:
[here](https://github.com/roie9876/NSXT-VMworld2019/blob/master/Part%204/ncp-ubuntu.yaml)

We now start to explain the diffrent filed of this file



##  (NCP) Configuration 

The file, "ncp-ubuntu.yml" will be used to deploy NSX Container Plugin. Node-Agent, Kube-Proxy , OVS and other related NCP configurations. 
However, before moving forward, NSX-T specific environmental parameters need to be configured. The yml file contains a configmap for the configuration of the ncp.ini file for the NCP.  Basically most of the parameters are commented out with a "#" character. The definitions of each parameter are in the yml file itself. 

The "ncp-ubuntu.yml" file can simply be edited with a text editor. The parameters in the file that are used in this environment has "#" removed. Below is a list and explanation of each :

staring with NCP 2.5 we can work with the policy API. with Policy API we need to defined the UUIDs of the objects.
pay attendtion thet UUIDs of the policy API are diffrents then the UUIDs of the manager API.

how we can find the IDs ? 
with Chrome broser open developr  

**policy_nsxapi = True** : User to specify that NCP will work with the Policy API.  

**single_tier_topology = True** : configure single tier1 per K8s/OpenShift Cluster. 

**cluster = k8scluster** : Used to identify the NSX-T objects that are provisioned for this K8S cluster. Notice that K8S Node logical ports in "K8s-Contaainers" are configured with the "k8scluster" tag and the "ncp/cluster" scope also with the hostname of Ubuntu node as the tag and "ncp/node_name" scope on NSX-T side.

**enable_snat = True** : This parameter basically defines that all the K8S Pods in each K8S namespace in this K8S cluster will be SNATed (to be able to access the other resources in the datacenter external to NSX domain) . The SNAT rules will be autoatically provisioned on Tier 0 Router in this lab. The SNAT IP will be allocated from IP Pool named "K8S-NAT-Pool" that was configured back in Part 3.

**** , **apiserver_host_port = 6443** : These parameters are for NCP to access K8S API. usaly this is the IP address of the master node, if we have clusters of k8s masters we nede to have LB with VIP address.

**ingress_mode = nat** : This parameter basically defines that NSX will use SNAT/DNAT rules for K8S ingress (L7 HTTPS/HTTP load balancing) to access the K8S service at the backend.

**nsx_api_managers = 192.168.110.34,192.168.110.35,192.168.110.36** , **nsx_api_user = admin** ,  **nsx_api_password = XXXXXXXXXXXXXX**  : These parameters are for NCP to access/consume the NSX Manager. in my lab i have 3 NSX managers, we need to provide the IPs of all managers (not the VIP of the managers). we can work with static username and passowrd (clear text) or we can work with certificate. in this demo i'm working with username and password.

**insecure = True** : NSX Manager server certificate is not verified. in lab enviroment where the NSX manager is self sigeded its better to use this way.

**ttier0_gateway = tier0** : The UUID of the Logical Gateway that will be used for connection top the phisical router (tier0).

**overlay_tz = a8811e7a-4d09-4076-94b3-7a9082750c64** : The UUID of the existing overlay transport zone that will be used for creating new logical switches/segments for K8S namespaces and container networking.

**subnet_prefix = 24** : The size of the IP Pools for the namespaces that will be carved out from the main "K8S-POD-IP-BLOCK" configured in Part 3 (172.25.0.0 /16). Whenever a new K8S namespace is created a /24 IP pool will be allocated from thatthat IP block.

**use_native_loadbalancer = True** : This setting is to use NSX-T load balancer for K8S Service Type : Load Balancer. Whenever a new K8S service is exposed with the Type : Load Balancer then a VIP will be provisioned on NSX-T LB attached to a Tier 1 Logical Router dedicated for LB function. The VIP will be allocated from the IP Pool named "K8S-LB-Pool" that was configured back in Part 3.

**default_ingress_class_nsx = True** : This is to use NSX-T load balancer for K8S ingress (L7 HTTP/HTTPS load balancing) , instead of other solutions such as NGINX, HAProxy etc. Whenever a K8S ingress object is created, a Layer 7 rule will be configured on the NSX-T load balancer.

**service_size = 'SMALL'** : This setting configures a small sized NSX-T Load Balancer for the K8S cluster. Options are Small/Medium/Large. This is the Load Balancer instance which is attached to a dedicated Tier 1 Logical Router in the topology.

**container_ip_blocks = K8S-POD-IP-BLOCK** : This setting defines from which IP block each K8S namespace will carve its IP Pool/IP address space from. (172.25.0.0 /16 in this case) Size of each K8S namespace pool was defined with subnet_prefix parameter above)

**external_ip_pools = K8S-NAT-Pool** : This setting defines from which IP pool each SNAT IP will be allocated from. Whenever a new K8S namespace is created, then a NAT IP will be allocated from this pool. (10.190.7.100 to 10.190.7.150 in this case)

**external_ip_pools_lb = K8S-LB-Pool** : This setting defines from which IP pool each K8S service, which is configured with Type : Load Balancer, will allocate its IP from. (10.190.6.100 to 10.190.6.150 in this case)

**top_firewall_section_marker = Section1** and **bottom_firewall_section_marker = Section2** : This is to specify between which sections the K8S orchestrated firewall rules will fall in between. 

_**One additional configuration that is made in the yml file is removing the "#" from the line where it says "serviceAccountName: ncp-svc-account" . So that the NCP Pod has appropriate role and access to K8S cluster resources**_ 

The edited yml file, "ncp-deployment-custom.yml" in this case, can now be deployed from anywhere. In this environment this yml file is copied to /home/vmware folder in K8S Master Node and deployed in the "nsx-system" namespace with the following command.




Verify that the new namespace is created. 

<pre><code>
root@k8s-master:~# <b>kubectl get namespaces</b>
NAME              STATUS   AGE
default           Active   5h22m
kube-node-lease   Active   5h22m
kube-public       Active   5h22m
kube-system       Active   5h22m
<b>nsx-system</b>        Active   5m47s
root@k8s-master:~#
</code></pre> 


<pre><code>
root@k8s-master:/home/vmware# <b>kubectl create -f ncp-deployment-custom.yml --namespace=nsx-system</b>
configmap/nsx-ncp-config created
deployment.extensions/nsx-ncp created
root@k8s-master:/home/vmware#
root@k8s-master:/home/vmware# <b>kubectl get pods --all-namespaces</b>
NAMESPACE     NAME                                 READY   STATUS              RESTARTS   AGE
kube-system   coredns-fb8b8dccf-b592z              0/1     ContainerCreating   0          5h45m
kube-system   coredns-fb8b8dccf-j66fg              0/1     ContainerCreating   0          5h45m
kube-system   etcd-k8s-master                      1/1     Running             0          5h44m
kube-system   kube-apiserver-k8s-master            1/1     Running             0          5h44m
kube-system   kube-controller-manager-k8s-master   1/1     Running             0          5h44m
kube-system   kube-proxy-bk7rs                     1/1     Running             0          97m
kube-system   kube-proxy-j4p5f                     1/1     Running             0          5h45m
kube-system   kube-proxy-mkm4w                     1/1     Running             0          122m
kube-system   kube-scheduler-k8s-master            1/1     Running             0          5h44m
<b>nsx-system</b>    <b>nsx-ncp-7f65bbf6f6-mr29b </b>            1/1     <b>Running</b>             0          18s
root@k8s-master:/home/vmware#
</code></pre>

As NCP is deployed as replicaset (replicas :1 is specified in deployment yml) , K8S will make sure that at a given time a single NCP Pod is running and healthy.

**Notice the changes to the existing logical switches/segments, Tier 1 Logical Routers, Load Balancer below . All these newly created objects have been provisioned by NCP (as soon as NCP Pod has been successfully deployed) by identifying the  the K8S desired state and mapping the K8S resources in etcd to the NSX-T Logical Networking constructs.**

LOGICAL SWITCHES  
![](2019-06-07_20-39-24.png)

LOGICAL ROUTERS
![](2019-06-03_20-39-41.png)

IP POOLS per Namespace
![](2019-06-03_21-06-24.png)

SNAT Pool
![](2019-06-03_20-41-55.png)

SNAT RULES
![](2019-06-03_20-39-59.png)

LOAD BALANCER
![](2019-06-03_20-40-28.png)

VIRTUAL SERVERS for INGRESS on LOAD BALANCER
![](2019-06-03_20-40-40.png)

FIREWALL RULEBASE
![](2019-06-03_20-43-42.png)

Notice also that CoreDNS pods are still in ContainerCreating phase, the reason for that is NSX Node Agent (which is responsible for connecting the pods to a logical switch) is still not installed on K8S Worker Nodes yet (next step)

# NSX Node Agent Installation
[Back to Table of Contents](#Table-Of-Contents)
 
"nsx-node-agent-ds.yml" will be used to deploy NSX Node Agent. This yml file is also provided in the content of the NSX Container Plugin zip file that was downloaded from My.VMware portal. 

This yml file also contains a configmap for the configuration of the ncp.ini file for the NSX Node Agent. The "nsx-node-agent-ds.yml" file can simply be edited with a text editor.  The following parameters need to be configured :

**apiserver_host_ip = 10.190.5.10** , **apiserver_host_port = 6443** : These parameters are for NSX Node Agent to access K8S API.

 **"#" is removed from the line with "serviceAccountname:..." so that role based access control can properly be applied for NSX Node Agent as well.**

The edited yml file, "nsx-node-agent-ds-custom.yml" in this case, can now be deployed from anywhere. In this environment this yml file is copied to /home/vmware folder in K8S Master Node and deployed in the "nsx-system" namespace with the following command.

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl create -f nsx-node-agent-ds-custom.yml --namespace=nsx-system</b>
</code></pre>

As NSX Node Agent is deployed as a deamonset it will be running on each worker node in the K8S cluster.

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl get pods --all-namespaces -o wide</b>
NAMESPACE     NAME                                 READY   STATUS              RESTARTS   AGE     IP            NODE         NOMINATED NODE   READINESS GATES
kube-system   coredns-fb8b8dccf-b592z              0/1     ContainerCreating   0          6h      <none>        k8s-master   <none>           <none>
kube-system   coredns-fb8b8dccf-j66fg              0/1     ContainerCreating   0          6h      <none>        k8s-master   <none>           <none>
kube-system   etcd-k8s-master                      1/1     Running             0          5h59m   10.190.5.10   k8s-master   <none>           <none>
kube-system   kube-apiserver-k8s-master            1/1     Running             0          5h59m   10.190.5.10   k8s-master   <none>           <none>
kube-system   kube-controller-manager-k8s-master   1/1     Running             0          5h59m   10.190.5.10   k8s-master   <none>           <none>
kube-system   kube-proxy-bk7rs                     1/1     Running             0          112m    10.190.5.12   k8s-node2    <none>           <none>
kube-system   kube-proxy-j4p5f                     1/1     Running             0          6h      10.190.5.10   k8s-master   <none>           <none>
kube-system   kube-proxy-mkm4w                     1/1     Running             0          137m    10.190.5.11   k8s-node1    <none>           <none>
kube-system   kube-scheduler-k8s-master            1/1     Running             0          5h59m   10.190.5.10   k8s-master   <none>           <none>
nsx-system    nsx-ncp-7f65bbf6f6-mr29b             1/1     Running             0          14m     10.190.5.12   k8s-node2    <none>           <none>
nsx-system    <b>nsx-node-agent-2tjb7</b>                 2/2     Running             0          24s     10.190.5.12   <b>k8s-node2</b>    <none>           <none>
nsx-system    <b>nsx-node-agent-nqwgx</b>                 2/2     Running             0          24s     10.190.5.11   <b>k8s-node1</b>    <none>           <none>
root@k8s-master:/home/vmware#
</code></pre>

**Note :** "-o wide" provides which Pod <=> Node mapping in the output

Notice yet again the coredns pods are still in ContainerCreating state. At this stage simply delete those two coredns pods and K8S scheduler will recreate those two pods and both of them will be successfully get attached to the respective overlay network on NSX-T side.

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl delete pod/coredns-fb8b8dccf-b592z --namespace=kube-system</b>
pod "coredns-fb8b8dccf-b592z" deleted
root@k8s-master:/home/vmware# <b>kubectl delete pod/coredns-fb8b8dccf-j66fg --namespace=kube-system</b>
pod "coredns-fb8b8dccf-j66fg" deleted
root@k8s-master:/home/vmware# <b>kubectl get pods --all-namespaces -o wide</b>
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE     IP            NODE         NOMINATED NODE   READINESS GATES
kube-system   <b>coredns-fb8b8dccf-fhn6q</b>              1/1     Running   0          3m40s   <b>172.25.4.4</b>    k8s-node1    <none>           <none>
kube-system   <b>coredns-fb8b8dccf-wqndw</b>              1/1     Running   0          88s     <b>172.25.4.3</b>    k8s-node2    <none>           <none>
kube-system   etcd-k8s-master                      1/1     Running   0          6h4m    10.190.5.10   k8s-master   <none>           <none>
kube-system   kube-apiserver-k8s-master            1/1     Running   0          6h4m    10.190.5.10   k8s-master   <none>           <none>
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          6h4m    10.190.5.10   k8s-master   <none>           <none>
kube-system   kube-proxy-bk7rs                     1/1     Running   0          117m    10.190.5.12   k8s-node2    <none>           <none>
kube-system   kube-proxy-j4p5f                     1/1     Running   0          6h5m    10.190.5.10   k8s-master   <none>           <none>
kube-system   kube-proxy-mkm4w                     1/1     Running   0          142m    10.190.5.11   k8s-node1    <none>           <none>
kube-system   kube-scheduler-k8s-master            1/1     Running   0          6h4m    10.190.5.10   k8s-master   <none>           <none>
nsx-system    nsx-ncp-7f65bbf6f6-mr29b             1/1     Running   0          20m     10.190.5.12   k8s-node2    <none>           <none>
nsx-system    nsx-node-agent-2tjb7                 2/2     Running   0          5m35s   10.190.5.12   k8s-node2    <none>           <none>
nsx-system    nsx-node-agent-nqwgx                 2/2     Running   0          5m35s   10.190.5.11   k8s-node1    <none>           <none>
root@k8s-master:/home/vmware#
</code></pre>

At this stage the topology looks like this 

![](2019-06-04_00-38-29.jpg)

# Test Workload Deployment
[Back to Table of Contents](#Table-Of-Contents)

Let' s create a new namespace 

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl create namespace demons</b>
namespace/demons created
root@k8s-master:/home/vmware#
</code></pre>

A new logical switch is created for "demons" namespace , shown below
![](2019-06-04_01-06-11.jpg)

NCP not only creates the above constructs but also tags them with the appropriate metadata, shown below 

![](2019-06-04_01-26-26.jpg)

For instance, in the above output , "project id" is the "UUID" for the "demons" K8S namespace. Which can be verified as below : 

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl get ns demons -o yaml</b>
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2019-06-03T23:54:01Z"
  name: demons
  resourceVersion: "773201"
  selfLink: /api/v1/namespaces/demons
  <b>uid: dcb423b5-865a-11e9-a2fc-005056b42e41</b>
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
</code></pre>

A new logical router is also created for "demons" namespace, shown below
![](2019-06-04_01-09-52.jpg)

A new IP Pool is also allocated from K8S-POD-IP-BLOCK, shown below
![](2019-06-04_01-22-20.jpg)

IP Pool allocated for the namespace is also tagged with metadata, shown below
![](2019-06-04_01-24-11.jpg)

An SNAT IP is allocated from the K8S-NAT-Pool (for all the Pods in demons namespace) , and the respective NAT rule is automatically configured on Tier 0 Logical Router (T0-K8S-Domain)

![](2019-06-04_02-17-44.jpg)

Deploy a sample app in the namespace (in imperative way)

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl run nsxtestapp --image=dumlutimuralp/nsx-demo --replicas=2 --namespace=demons</b>
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
<b>deployment.apps/nsxtestapp created</b>
root@k8s-master:/home/vmware# 
</code></pre>

Note : Notice the message in the output. K8S is recommending declerative way of implementing pods. 

Verify that the Pods are created and allocated IPs from the appropriate IP pool

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl get pods -o wide --namespace=demons</b>
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE        NOMINATED NODE   READINESS GATES
nsxtestapp-5bfcc97b5-n5wbz   1/1     Running   0          11m   172.25.5.3   k8s-node2   <none>           <none>
nsxtestapp-5bfcc97b5-ppkqd   1/1     Running   0          11m   172.25.5.2   k8s-node1   <none>           <none>
root@k8s-master:/home/vmware#
</code></pre>

Now the topology looks like below 

![](2019-06-04_02-11-28.jpg)

The logical port for each Pod shows up in NSX-T UI, shown below

![](2019-06-04_01-44-12.jpg)

Let' s look at the tags that are associated with that logical port as metadata

![](2019-06-04_01-44-40.jpg)

As shown above, the K8S namespace name, K8S Pod name and UUID (can be verified on K8S below) are carried over to NSX-T as metadata.

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl get pod/nsxtestapp-5bfcc97b5-n5wbz -o yaml --namespace=demons</b>
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2019-06-04T00:23:32Z"
  generateName: nsxtestapp-5bfcc97b5-
  labels:
    pod-template-hash: 5bfcc97b5
    run: nsxtestapp
  name: nsxtestapp-5bfcc97b5-n5wbz
  namespace: demons
|
|
Output Omitted
|
|
  resourceVersion: "776038"
  selfLink: /api/v1/namespaces/demons/pods/nsxtestapp-5bfcc97b5-n5wbz
  uid: <b>fc30d02f-865e-11e9-a2fc-005056b42e41</b>
spec:
  containers:
  - image: dumlutimuralp/nsx-demo
    imagePullPolicy: Always
    name: nsxtestapp
|
|
Output Omitted
|
|
</code></pre>


Finally let' s check the IP connectivity of the Pod to the resources external to NSX domain.

Perform the command below to get a shell in one of the Pods

<pre><code>
root@k8s-master:/home/vmware# <b>kubectl exec -it nsxtestapp-5bfcc97b5-n5wbz /bin/bash --namespace=demons</b>
root@nsxtestapp-5bfcc97b5-n5wbz:/app# 
</code></pre>

Check the IP address of the Pod

<pre><code>
root@nsxtestapp-5bfcc97b5-n5wbz:/app# <b>ip addr</b>
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ovs-gretap0@NONE: <BROADCAST,MULTICAST> mtu 1462 qdisc noop state DOWN group default qlen 1000
    link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
3: erspan0@NONE: <BROADCAST,MULTICAST> mtu 1450 qdisc noop state DOWN group default qlen 1000
    link/ether 36:89:34:84:cd:91 brd ff:ff:ff:ff:ff:ff
4: gre0@NONE: <NOARP> mtu 1476 qdisc noop state DOWN group default qlen 1
    link/gre 0.0.0.0 brd 0.0.0.0
5: ovs-ip6gre0@NONE: <NOARP> mtu 1448 qdisc noop state DOWN group default qlen 1
    link/gre6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
6: ovs-ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN group default qlen 1
    link/tunnel6 :: brd ::
50: eth0@if51: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 02:50:56:00:28:03 brd ff:ff:ff:ff:ff:ff
 inet <b>172.25.5.3/24</b> scope global eth0
       valid_lft forever preferred_lft forever
root@nsxtestapp-5bfcc97b5-n5wbz:/app# 
</code></pre>

Ping the external physical router from the Pod

<pre><code>
root@nsxtestapp-5bfcc97b5-n5wbz:/app# <b>ping 10.190.4.1</b>
PING 10.190.4.1 (10.190.4.1): 56 data bytes
64 bytes from 10.190.4.1: icmp_seq=0 ttl=62 time=3.047 ms
64 bytes from 10.190.4.1: icmp_seq=1 ttl=62 time=1.534 ms
64 bytes from 10.190.4.1: icmp_seq=2 ttl=62 time=1.130 ms
64 bytes from 10.190.4.1: icmp_seq=3 ttl=62 time=1.044 ms
64 bytes from 10.190.4.1: icmp_seq=4 ttl=62 time=1.957 ms
64 bytes from 10.190.4.1: icmp_seq=5 ttl=62 time=1.417 ms
^C--- 10.190.4.1 ping statistics ---
6 packets transmitted, 6 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.044/1.688/3.047/0.676 ms
root@nsxtestapp-5bfcc97b5-n5wbz:/app#
</code></pre>


To identify containers per K8S Node, the logical port that the K8S Worker Node' s vNIC2 is connected to can be investigated too. Shown below.

![](2019-06-05_22-51-43.png)

Container ports show up as below.

![](2019-06-05_22-53-16.png)















