# NSX-T & OpenShift - PART 5
[Home Page](https://github.com/roie9876/NSXT-VMworld2019)

# Table Of Contents

[Deploy OpenShift Cluster](#Current-State)   
[NSX Container Plugin (NCP) Installation with OpenShift](#NSX-Container-Plugin-Installation)  
[Deploy sample Application](#Test-Workload-Deployment)



## Deploy OpenShift Cluster

We deploy new OpenShift Cluster in version 3.11.  
You can read very good blog post created by Yasen Simeonov:
https://blogs.vmware.com/networkvirtualization/2019/02/nsx-t-integration-with-openshift.html/ 

In my lab i deploy 7 VMs as fllows:  

![](2019-10-26-07-36-26.png)


**In NCP version 2.5 and OpenShift integration we change the way the installation works. in the OpenShift host file we are not providing NSX configuration**

In The SDN section we specify that we are Not Installaing OpenShift SDN and Not Installiong NSX SDN


**openshift_use_openshift_sdn=false**  
**openshift_use_nsx=false**

Here is my complite OpenShift Host file:

<pre><code>
[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
ansible_ssh_user=root
openshift_deployment_type=origin

openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
openshift_master_htpasswd_users={'demo':'$apr1$Y.qago1d$kq387DpWGC.NrbCyLG8xf.'}
openshift_disable_check=package_version,memory_availability,disk_availability,docker_storage,docker_image_availability
openshift_cluster_monitoring_operator_install=false
openshift_logging_install_logging=false
openshift_web_console_install=false

openshift_master_default_subdomain=demo.lab.local
openshift_use_nsx=false
os_sdn_network_plugin_name=cni
openshift_use_openshift_sdn=false
openshift_node_sdn_mtu=1500
openshift_master_cluster_method=native
openshift_master_cluster_hostname=master01.lab.local
openshift_master_cluster_public_hostname=master01.lab.local

[masters]
master01.lab.local
master02.lab.local
master03.lab.local

[etcd]
master01.lab.local
master02.lab.local
master03.lab.local

[nodes]
master01.lab.local ansible_ssh_host=192.168.112.71 openshift_node_group_name='node-config-master'
master02.lab.local ansible_ssh_host=192.168.112.72 openshift_node_group_name='node-config-master'
master03.lab.local ansible_ssh_host=192.168.112.73 openshift_node_group_name='node-config-master'
infra01.lab.local ansible_ssh_host=192.168.112.74 openshift_node_group_name='node-config-infra'
infra02.lab.local ansible_ssh_host=192.168.112.75 openshift_node_group_name='node-config-infra'
node01.lab.local ansible_ssh_host=192.168.112.76 openshift_node_group_name='node-config-compute'
node02.lab.local ansible_ssh_host=192.168.112.77 openshift_node_group_name='node-config-compute'

</code></pre>  

Run on all node VMs:
<pre><code>
yum -y install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct
yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo
</code></pre>

Run on the master nodes:


<pre><code>
ssh-copy-id -i ~/.ssh/id_rsa.pub master01
ssh-copy-id -i ~/.ssh/id_rsa.pub master02
ssh-copy-id -i ~/.ssh/id_rsa.pub master03
ssh-copy-id -i ~/.ssh/id_rsa.pub node01
ssh-copy-id -i ~/.ssh/id_rsa.pub node02
ssh-copy-id -i ~/.ssh/id_rsa.pub infra01
ssh-copy-id -i ~/.ssh/id_rsa.pub infra02
 
yum -y --enablerepo=epel install ansible pyOpenSSL
git clone https://github.com/openshift/openshift-ansible
cd openshift-ansible/
git checkout release-3.11
cd
ansible-playbook -i hosts openshift-ansible/playbooks/prerequisites.yml
</code></pre>


Last step is to deploy the Openshift cluster:
<pre><code>
ansible-playbook -i hosts openshift-ansible/playbooks/deploy_cluster.yml
</code></pre>


<pre><code>
[root@master01 kubernetes-manifests]# oc get node
NAME                 STATUS    ROLES            AGE       VERSION
infra01.lab.local    Ready     compute,infra    6d        v1.11.0+d4cacc0
infra02.lab.local    Ready     compute,infra    6d        v1.11.0+d4cacc0
master01.lab.local   Ready     compute,master   6d        v1.11.0+d4cacc0
master02.lab.local   Ready     compute,master   6d        v1.11.0+d4cacc0
master03.lab.local   Ready     compute,master   6d        v1.11.0+d4cacc0
node01.lab.local     Ready     compute          6d        v1.11.0+d4cacc0
node02.lab.local     Ready     compute          6d        v1.11.0+d4cacc0
</code></pre>


When the deploy cluster ansible script finish there is no function SDN in the cluster.  
We need to deploy the ncp-openshift.yaml

The original unmodified ncp-openshift can be found here:
https://github.com/roie9876/NSXT-VMworld2019/blob/master/Part%205/ncp-openshift.yaml



# NSX Container Plugin Installation
[Back to Table of Contents](#Table-Of-Contents)

NSX Container Plugin (NCP) image file need to be copy to all masters and workers.  

## Load The Docker Image for NSX NCP on all masters and workers:

For the commands below, "sudo" can be used with each command or privilege can be escalated to root by using "sudo -H bash" in advance.



_**NSX Container Plugin (NCP) and NSX Node Agent Pods use the same container image.**_

## Install NCP image on all masters and Workers 

CNI is a Cloud Native Computing Foundation (CNCF) project. It is a set of specifications and libraries to configure network interfaces in Linux containers. It has a pluggable architecture hence third party plugins are supported.

NSX-T CNI Plugin comes within the "NSX-T Container" package. The package can be downloaded (in _.zip_ format) from Downloads page for NSX-T, shown below.

![](2.png)

In the current NSX-T version (2.5.0) , the zip file is named as "**nsx-container-2.5.0.14628220**" . 

* Extract the zip file to a folder.

![](2019-24-10-19-06-21.png)

Use the NCP image for RHEL:  
![](2019-10-26-07-46-31.png)

Once the above playbook finish do the following on all nodes:
<pre><code>
docker load -i nsx-ncp-rhel-2.5.0.14628220.tar
# Get the image name 
docker images
docker image tag registry.local/nsx-ncp-rhel-2.5.0.14628220.tar nsx-ncp
</code></pre>


##  (NCP) Configuration 

The file, "ncp-opensift.yml" will be used to deploy NSX Container Plugin. Node-Agent, Kube-Proxy , OVS and other related NCP configurations. 
However, before moving forward, NSX-T specific environmental parameters need to be configured. The yml file contains a configmap for the configuration of the ncp.ini file for the NCP.  Basically most of the parameters are commented out with a "#" character. The definitions of each parameter are in the yml file itself. 

The "ncp-opensift.yml" file can simply be edited with a text editor. The parameters in the file that are used in this environment has "#" removed. Below is a list and explanation of each :

staring with NCP 2.5 we can work with the policy API. with Policy API we need to defined the UUIDs of the objects.
#### Note: The UUIDs of the policy API are differences then the UUIDs of the manager API.

### how we can find the Policy UUID ? 
With the Chrome browser, we need to use the  Developer Tools:  
![](2019-10-25-12-21-36.png) 

Click on the Ctrl + R to start record:  

![](2019-10-25-12-22-31.png)  

We can clear the screen by the small icon ![](2019-10-25-12-23-01.png)  
for example lets find the UUID of the K8s-LB-Pool: 
We need to click on the NSX-T object that we would like to find his UUID:  


There is a small blue circle refresh button:  ![](2019-10-25-12-29-01.png) , Click on it. New date will show up. as you can see in the image bellow  
the **"id" : "k8s-LB-Poo"** this is the object UUID.  
  
  ![](2019-10-25-12-25-42.png)  
    

let's start to explain the different parameters:

**policy_nsxapi = True** : Specify that NCP will work with the Policy API.  

**single_tier_topology = True** : configure single tier1 per K8s/OpenShift Cluster. starting with NCP 2.5 we have two options for the tier1. we can manually create this tier1 and specify his name in the ncp config file, or we can let ncp automatically create this tier1. in this demo we want NCP to create this tier1 gateway. the name of tier1 taken from the **cluster =** parameters, in our demo its k8scluster.

**cluster = k8scluster** : Used to identify the NSX-T objects that are provisioned for this K8S cluster. Notice that K8S Node logical ports in "K8s-Contaainers" are configured with the "k8scluster" tag and the "ncp/cluster" scope also with the hostname of Ubuntu node as the tag and "ncp/node_name" scope on NSX-T side.

**enable_snat = True** : This parameter basically defines that all the K8S Pods in each K8S namespace in this K8S cluster will be SNATed (to be able to access the other resources in the datacenter external to NSX domain) . The SNAT rules will be autoatically provisioned on Tier 0 Router in this lab. The SNAT IP will be allocated from IP Pool named "K8S-NAT-Pool" that was configured back in Part 3.

**apiserver_host_port = 6443** : These parameters are for NCP to access K8S API. **Note this parnter need to configured multiple times in this yaml file, so every time you this parmter you need to configure the same value**
  
**apiserver_host_ip = 192.168.113.2**  
This is the IP address of the master node, if we have clusters of k8s masters we need to use the LB VIP.  
**Note this paramter need to configured multiple times in this yaml file, so every time you this parmter you need to configure the same value**

**ingress_mode = nat** : This parameter basically defines that NSX will use SNAT/DNAT rules for K8S ingress (L7 HTTPS/HTTP load balancing) to access the K8S service at the backend.

**nsx_api_managers = 192.168.110.34,192.168.110.35,192.168.110.36** , **nsx_api_user = admin** ,  **nsx_api_password = VMware1!VMware1!**  : These parameters are for NCP to access/consume the NSX Manager. in my lab i have 3 NSX managers, we need to provide the IPs of all managers (not the VIP of the managers). we can work with static username and passowrd (clear text) or we can work with certificate. in this demo i'm working with username and password.

**insecure = True** : NSX Manager server certificate is not verified. in lab enviroment where the NSX manager is self sigeded its better to use this way.

**ttier0_gateway = tier0** : The UUID of the Logical Gateway that will be used for connection top the phisical router (tier0).

**overlay_tz = a8811e7a-4d09-4076-94b3-7a9082750c64** : The UUID of the existing overlay transport zone that will be used for creating new logical switches/segments for K8S namespaces and container networking.

**subnet_prefix = 24** : The size of the IP Pools for the namespaces that will be carved out from the main "K8S-POD-IP-BLOCK" configured in Part 3 (10.19.0.0 /16). Whenever a new K8S namespace is created a /24 IP pool will be allocated from thatthat IP block.

**use_native_loadbalancer = True** : This setting is to use NSX-T load balancer for K8S Service Type : Load Balancer. Whenever a new K8S service is exposed with the Type : Load Balancer then a VIP will be provisioned on NSX-T LB attached to a Tier 1 Logical Router dedicated for LB function. The VIP will be allocated from the IP Pool named "K8S-LB-Pool" that was configured back in Part 3.

**default_ingress_class_nsx = True** : This is to use NSX-T load balancer for K8S ingress (L7 HTTP/HTTPS load balancing) , instead of other solutions such as NGINX, HAProxy etc. Whenever a K8S ingress object is created, a Layer 7 rule will be configured on the NSX-T load balancer.

**service_size = 'SMALL'** : This setting configures a small sized NSX-T Load Balancer for the K8S cluster. Options are Small/Medium/Large. This is the Load Balancer instance which is attached to a dedicated Tier 1 Logical Router in the topology.

**container_ip_blocks = K8S-POD-IP-BLOCK** : This setting defines from which IP block each K8S namespace will carve its IP Pool/IP address space from. (172.25.0.0 /16 in this case) Size of each K8S namespace pool was defined with subnet_prefix parameter above)

**external_ip_pools = K8S-NAT-Pool** : This setting defines from which IP pool each SNAT IP will be allocated from. Whenever a new K8S namespace is created, then a NAT IP will be allocated from this pool. (10.190.7.100 to 10.190.7.150 in this case)

**external_ip_pools_lb = K8S-LB-Pool** : This setting defines from which IP pool each K8S service, which is configured with Type : Load Balancer, will allocate its IP from. (10.190.6.100 to 10.190.6.150 in this case)

**top_firewall_section_marker = Section1** and **bottom_firewall_section_marker = Section2** : This is to specify between which sections the K8S orchestrated firewall rules will fall in between. 





I modified this file to make it works according to my environment:
https://github.com/roie9876/NSXT-VMworld2019/blob/master/Part%205/ncp-openshift-VMworkd2019.yml

<pre><code>
oc create -f ncp-opensift-VMworkd2019.yml
</code></pre>


The namespaces that are provisioned by default can be seen using the following kubectl command.
<pre><code>
[root@master01 home]# oc get ns
NAME                STATUS    AGE
default             Active    6d
kube-public         Active    6d
kube-system         Active    6d
management-infra    Active    6d
nsx-system          Active    5d
openshift           Active    6d
openshift-infra     Active    6d
openshift-logging   Active    6d
openshift-node      Active    6d
</code></pre>


<pre><code>
[root@master01 home]# oc get pod -n nsx-system
NAME                       READY     STATUS    RESTARTS   AGE
nsx-ncp-6857f8cd4c-ps96g   1/1       Running   0          51s
nsx-ncp-bootstrap-7qssw    1/1       Running   0          5d
nsx-ncp-bootstrap-967v2    1/1       Running   0          5d
nsx-ncp-bootstrap-98gxf    1/1       Running   0          5d
nsx-ncp-bootstrap-9zppw    1/1       Running   0          5d
nsx-ncp-bootstrap-c9ph8    1/1       Running   0          5d
nsx-ncp-bootstrap-lvs7l    1/1       Running   0          5d
nsx-ncp-bootstrap-rrtzf    1/1       Running   1          5d
nsx-node-agent-22zms       3/3       Running   2          5d
nsx-node-agent-5tznp       3/3       Running   3          5d
nsx-node-agent-fbwtj       3/3       Running   0          5d
nsx-node-agent-g9j7n       3/3       Running   0          5d
nsx-node-agent-qgnb9       3/3       Running   0          5d
nsx-node-agent-srr27       3/3       Running   0          5d
nsx-node-agent-w9rkc       3/3       Running   0          5d

</code></pre>


**Notice the changes to the existing logical switches/segments, Tier 1 Logical Routers, Load Balancer below . All these newly created objects have been provisioned by NCP (as soon as NCP Pod has been successfully deployed) by identifying the  the K8S desired state and mapping the K8S resources in etcd to the NSX-T Logical Networking constructs.**

We can view the tier1 gateway created by NCP:  

![](2019-10-26-09-01-04.png)


This tieer1 have 9 Seegments, each segemnt its equal to OpenShift project:  
![](2019-10-26-09-02-22.png)


For each OpenShift project we have NAT etry created on the tier1 Gateway:

![](2019-10-26-09-03-52.png)

NCP automaticly created Load Balancer with two VIPs:
![](2019-10-26-09-06-32.png)

In The Virtual Server (VIP) we have two L7 etnry:
![](2019-10-26-09-07-21.png)







LOGICAL Segments  
![](2019-10-25-13-05-38.png)

New Tier1 LOGICAL Gateway:
![](2019-10-25-13-28-37.png)


Connected to this Tier1 logical router we have the following Segnets: 
![](2019-10-25-13-32-51.png)

SNAT Rules
![](2019-10-25-13-35-26.png)


LOAD BALANCER runing on the same Tier1 created to the k8scluster
![](2019-10-25-13-36-48.png)

VIRTUAL SERVERS for INGRESS on LOAD BALANCER
![](2019-10-25-13-38-26.png)

FIREWALL RULEBASE
![](2019-10-25-13-38-57.png)

Notice also that CoreDNS pods are changed thier status to Running state.  
<pre><code>
root@master:/home/localadmin# kubectl get pod -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-584795fc57-5wvll         1/1     Running   1          3d20h
coredns-584795fc57-pmgl5         1/1     Running   1          3d20h
etcd-master                      1/1     Running   0          4d5h
kube-apiserver-master            1/1     Running   0          4d5h
kube-controller-manager-master   1/1     Running   0          4d5h
kube-proxy-vpgfq                 1/1     Running   0          4d5h
kube-proxy-zlc9f                 1/1     Running   0          4d5h
kube-scheduler-master            1/1     Running   0          4d5h
</code></pre> 



# Deploy the acme application
Thie application contains a Polyglot demo application comprised of (presently) 6 microservices and 4 datastores:  

![](2019-10-25-15-41-51.png)

The contents here are the necessary YAML files to deploy the ACMEFIT application in a kubernetes cluster.

This app is developed by team behind www.cloudjourney.io

https://github.com/roie9876/acme_fitness_demo

<pre><code>
git clone https://github.com/roie9876/acme_fitness_demo
</code></pre> 



create acme  namespace, we can work with NAT namespace or No-NAT. lets create No-Nat. to do this we need to anotated the namespace.

<pre><code>
cat acme-no-nat.yaml
apiVersion: v1
kind: Namespace
metadata:
   name: acme
   annotations:
      ncp/no_snat: "True"
</code></pre> 

<pre><code>
root > cat acme-no-nat.yaml
apiVersion: v1
kind: Namespace
metadata:
   name: acme
   annotations:
      ncp/no_snat: "True"
root > kubectl create -f acme-no-nat.yaml
namespace/acme created  
  

root > kubectl describe ns acme
Name:         acme
Labels:       <none>
Annotations:  ncp/no_snat: True
Status:       Active

No resource quota.

No resource limits.

</code></pre>   

We can see the results of the new namespace in NSX Segments, we have new Segment with the name: seg-k8cluster-acme-0.  
In the PORTS we have 0 Connected PODs (we didn't deploy any PODs yet...)

![](2019-10-25-16-13-36.png)

NSXT curve new CIDR 10.19.2.0/24 where the 10.19.2.1 is the default Gateway for this namespce 
![](2019-10-25-16-16-03.png)

We can chekc if we have NCP create NAT rule for this namespace:  
As you can see there is no any ANT entry for 10.19.2.0/42
![](2019-10-25-16-17-39.png)


Lets change the default namespace to acme:
<pre><code>
root > kubectl config set-context --current --namespace=acme
Context "kubernetes-admin@kubernetes" modified.
 </code></pre>   


# Deploy the acme application:  
<pre><code>
root > kubectl create -f acme_fitness.yaml
secret/redis-pass created
secret/catalog-mongo-pass created
secret/order-mongo-pass created
secret/users-mongo-pass created
service/cart-redis created
deployment.apps/cart-redis created
service/cart created
deployment.apps/cart created
configmap/catalog-initdb-config created
service/catalog-mongo created
deployment.apps/catalog-mongo created
service/catalog created
deployment.apps/catalog created
deployment.apps/frontend created
service/order-mongo created
deployment.apps/order-mongo created
service/order created
deployment.apps/order created
service/payment created
deployment.apps/payment created
configmap/users-initdb-config created
service/users-mongo created
deployment.apps/users-mongo created
service/users created
deployment.apps/users created
 </code></pre>    


<pre><code>
root > kubectl get pod -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
cart-6d84f4cf64-ftlbp            1/1     Running   0          42s   10.19.2.4    node02   <none>           <none>
cart-redis-7fcdbcd8d8-848rx      1/1     Running   0          42s   10.19.2.5    node02   <none>           <none>
catalog-587f7df8c8-7bxst         1/1     Running   0          41s   10.19.2.2    node02   <none>           <none>
catalog-mongo-5c8db99954-l2plp   1/1     Running   0          41s   10.19.2.3    node02   <none>           <none>
frontend-7557468545-9gzb2        1/1     Running   0          41s   10.19.2.6    node02   <none>           <none>
order-c8885777b-kjdf7            1/1     Running   0          41s   10.19.2.8    node02   <none>           <none>
order-mongo-7cc8f784f8-tl29b     1/1     Running   0          41s   10.19.2.7    node02   <none>           <none>
payment-79bc456ff-cxprm          1/1     Running   0          40s   10.19.2.9    node02   <none>           <none>
users-6d7c5dd7c8-t6jmc           1/1     Running   0          40s   10.19.2.11   node02   <none>           <none>
users-mongo-7cfb97b5bb-mxhkc     1/1     Running   0          40s   10.19.2.10   node02   <none>           <none>
  
</code></pre>   
As you can see, all the PODs get IPs from range 10.19.2.0/24

We can see the reflections of the PODs in NSXT
![](2019-10-25-16-20-59.png)  

Now we have 10 PORTs connected to this segments
we can see the PODs name from NSX UI:  
![](2019-10-25-16-24-24.png)

We can see all the K8s tags from the NSX UI:  
![](2019-10-25-16-25-44.png)


The Load Balancer status before deploy new LB VIPs:
![](2019-10-25-16-38-20.png)
This is the default VIP created with the K8s cluster.

# L4 Load Balancer
Create new L4 Service Type LB with persistence IP
<pre><code>
root > cat frontend-svc.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: frontend-total
    service: frontend
  name: frontend
  namespace: acme
spec:
  ports:
  - name: http-frontend
    nodePort: 30807
    port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: frontend-total
    service: frontend
  type: LoadBalancer
  loadBalancerIP: 10.15.100.100
</code></pre>  

We expting the VIP IP address will be 10.15.100.100
<pre><code>
root > kubectl create -f frontend-svc.yaml
service/frontend created
root > kubectl get svc
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)          AGE
cart            ClusterIP      10.100.119.49    <none>          5000/TCP         22m
cart-redis      ClusterIP      10.105.208.2     <none>          6379/TCP         22m
catalog         ClusterIP      10.101.251.48    <none>          8082/TCP         22m
catalog-mongo   ClusterIP      10.111.14.3      <none>          27017/TCP        22m
frontend        LoadBalancer   10.105.235.195   10.15.100.100   3000:30807/TCP   9s
order           ClusterIP      10.111.73.197    <none>          6000/TCP         22m
order-mongo     ClusterIP      10.97.151.38     <none>          27017/TCP        22m
payment         ClusterIP      10.103.128.200   <none>          9000/TCP         22m
users           ClusterIP      10.106.44.10     <none>          8081/TCP         22m
users-mongo     ClusterIP      10.104.246.186   <none>          27017/TCP        22m

</code></pre>   

The as you can see the svc of the frontend has 10.15.100.100 with port 3000 as L4 entry.
We can see the results of the new LB VIP in NSX UI:

![](2019-10-25-16-43-50.png)



We can see the server pool member from NSX UI:
![](2019-10-25-16-44-44.png)


Scale more frontend PODs to the acme app:

<pre><code>
root > kubectl scale --replicas=2 deployment frontend
deployment.extensions/frontend scaled
  
</code></pre>   

We can see that we now have 2 endpoint in the service pool: 
<pre><code>  
root > kubectl describe svc frontend
Name:                     frontend
Namespace:                acme
Labels:                   app=frontend-total
                          service=frontend
Annotations:              ncp/internal_ip_for_policy: 100.64.128.5
Selector:                 app=frontend-total,service=frontend
Type:                     LoadBalancer
IP:                       10.105.235.195
IP:                       10.15.100.100
LoadBalancer Ingress:     10.15.100.100
Port:                     http-frontend  3000/TCP
TargetPort:               3000/TCP
NodePort:                 http-frontend  30807/TCP
Endpoints:                10.19.2.12:3000,10.19.2.6:3000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
 
</code></pre> 

From NSX UI we can see we have new member in the server pool:
![](2019-10-25-16-48-27.png)

Test the acme application with the broswer:
![](2019-10-25-16-49-19.png)


# L7 Ingress
We can create L7 ingress for the frontend service:  

root > cat ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: frontend
spec:
  rules:
  - host: frontend.lab.local
    http:
      paths:
      - path: /*
        backend:
          serviceName: frontend
          servicePort: 3000
root > kubectl create -f ingress.yaml
ingress.extensions/frontend created
root > kubectl describe ingress frontend
Name:             frontend
Namespace:        acme
Address:          10.20.1.1
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host                Path  Backends
  ----                ----  --------
  frontend.lab.local
                      /*   frontend:3000 (10.19.2.12:3000,10.19.2.6:3000)
Annotations:
  ncp/internal_ip_for_policy:  100.64.128.5
Events:                        <none>

In the NSX-T LB we can can see the L7 rule:  
The rule created in the **Request Forwarding Phase**
![](2019-10-25-17-16-21.png)

We can view the re-write rule from the UI:  

![](2019-10-25-17-17-48.png)

The user sends traffic to URL: frontend.lab.local which hit the L7 Ingress in NSX LB. Then the NSX LB re-write this request and sends it to the K8s frontend pool.

