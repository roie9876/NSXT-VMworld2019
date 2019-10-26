# NSXT-VMworld2019
This repo contain the demo for vmworld 2019 and show how NSX-T SDN can work in parallel OpenShift and Native K8s  

_**This disclaimer informs readers that the views, thoughts, and opinions expressed in this series of posts belong solely to the author, and not necessarily to the author’s employer, organization, committee or other group or individual.**_

### Credits  
This guide is based on Dumlue Trump Amazning Guide:  
 [@dumlutimuralp](https://twitter.com/dumlutimuralp) / [LinkedIn](https://www.linkedin.com/in/dumlutimuralp/) 

His works can be found in this link:  
https://github.com/dumlutimuralp/nsx-t-k8s  
  


#  Software components used:  

### vSphere  
vCenter 6.7 U3 (Build 14368073)  
ESX 6.7 U3 (Build 14320388)  
### NSX-T  
NSX-T 2.5.0 (Build 14663978)  
NSX Container Plugin 2.5.0 (Build 14628220)  

### Native K8S Clusters:
Ubuntu Server Ubuntu 16.04.6 LTS  
Docker CE 18.09.6  
Kubernetes 1.15.5  

### OpenShift Cluster:
CentOS Linux release 7.6.1810 (Core)  
OpenShift 3.11


It is highly recommended to check the following resources for compatibility requirements
* VMware Product Interoperability Matrices  
https://www.vmware.com/resources/compatibility/sim/interop_matrix.php#interop&175=&2=&1=
* VMware NSX Container Plugin Release Notes
https://docs.vmware.com/en/VMware-NSX-T-Data-Center/index.html  


The goal of this series of posts is to outline and explain the steps to integrate VMware NSX-T Platform to Kubernetes control plane.

Below articles focus on the management and data plane integration principles of NSX-T and K8S.

### [Part 1](https://github.com/roie9876/NSXT-VMworld2019/tree/master/Part%201)

* Topology Information
* Fabric Preperation
* Host Transport Node
* Edge Transport Node
* Edge Cluster
* Tier 0 Logical Router
* Tier 1 Logical Router
* Tier 0 BGP Configuration
* K8S Node Dataplane Connectivity


### [Part 2](https://github.com/roie9876/NSXT-VMworld2019/tree/master/Part%202

* Container Network Interface (CNI)
* NSX-T Components in Kubernetes Integration
* NSX-T & K8S Overall Architecture

### [Part 3](https://github.com/roie9876/NSXT-VMworld2019/tree/master/Part%203)

* Ubuntu OS Installation
* Topology
* IPAM and IP Pools
* Reachability
* Tagging NSX-T Objects for K8S
* Docker Installation
* Deploy K8S

### [Part 4](https://github.com/roie9876/NSXT-VMworld2019/tree/master/Part%204)

* NCP and K8s cluster integration
* NSX Container Plugin (NCP) Installation
* Deploy sample Application

### [Part 5](https://github.com/roie9876/NSXT-VMworld2019/tree/master/Part%205)

* NCP and OpenShift cluster integration
* NSX Container Plugin (NCP) Installation
* Deploy sample Application