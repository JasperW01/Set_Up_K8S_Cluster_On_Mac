# Set_Up_K8S_Cluster_On_Mac

In the article, we will describe how to set up a full Kubernetes cluster on a MacPro laptop. The Mac we are using is at version 10.12.5. 

The steps are sourced from multiple articles to set up K8S cluster on a typical MacProc laptop with Vagrant & CoreOS, which has corrections to the errors/mismatches in the original artiticles and chains them together to be a full set of practical steps to 
set up a full Kubernetes cluster step by step. 

At the high level, there are three steps during the process: 

1. Use Vagrant to create four CoreOS VMs on a MacPro laptop. 

2. On those 4 CoreOS VMs, set up a Kubernetes cluster step by step with 
          i. one etcd server
          ii. one k8s master node
          iii. Two k8s worker nodes (more can be added afterwards)
   We choose to install K8s cluster step by step so that it can give you deeper understanding about the moving parts and we can benefit
   more in the long run. 
   
3. Set up the k8s example of guestbook on the newly created k8s cluster. 

A. SET UP 4 COREOS VMs WITH VAGRANT ON Mac

The steps in this section are mainly sourced from  https://coreos.com/os/docs/latest/booting-on-vagrant.html. However some steps in this article are not consistent with the correpsonding artifects. As a consequence, we have corrected some steps according to the README.mod file clones from the corresponding GitHub repository and CoreOS common practice. 

Vagrant is a simple-to-use command line virtual machine manager. By working with VirtualBox (or VMWare which requires to purchase plug-in), it virtually turns your MacPro laptop into a datacentre.

Before we start the steps, we assume the Oracle VirtualBox and Vagrant have already been installed on the Mac laptop. VirtualBox for Mac can be downloaded from https://www.virtualbox.org/wiki/Downloads and Vagrant binary for Mac can be downloaded from https://www.vagrantup.com/downloads.html. 

Step 1. Clone the Vagrant + CoreOS repository from GitHub

On the MacPro laptop, clone the Vagrant + CoreOS repository released by CoreOS. 

*********************************************
MacBook-Pro:~ jaswang$ mkdir k8s

MacBook-Pro:~ jaswang$ cd k8s/

MacBook-Pro:k8s jaswang$ git clone https://github.com/coreos/coreos-vagrant.git
*********************************************

Step 2. Request a new etcd discovery token

In our setting up, we will have 4 VMs and 1 of them will be running etcd server and the other three will be etcd clients. So we use the URL of  https://discovery.etcd.io/new?size=1 (please note size is 1 here) to request a new etcd discovery token and the token will be applied to all 4 VMs to be created. As a result, 1 VM will be dedicated to etcd server and the other 3 VMs will be etcd clients and installed with Kubernetes components. As per k8s best practice, it's strongly recommanded to have VMs dedicated to etcd cluster separate to the VMs used in K8S cluster. In our case, the etcd cluster is composed of 1 VM only to save resources of the Mac. 

Please note in the latest CoreOS Container Linux image, it's using etcd V3 (i.e. etcd-member.service) instead of etcd v2 (i.e. etcd2.service). 

********************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ curl https://discovery.etcd.io/new\?size\=1
********************************************

Step 2. Update the human readiable cl.conf file and translate to CoreOS ignition file 

In the latest version of CoreOS, the traditional cloud-config has been superceded by ignition. In particular, in the repo downloaded in Step 1, by default it's using Vagrant with VirtualBox and it's expecting a config.ign file instead of a cloud-config file (i.e. user-data). So as CoreOS common practice, we need to request a new etcd discovery token to update the cl.conf in the cloned repository and use CoreOS config transpiler (i.e. ct tool) to translate cl.conf file to CoreOS ignition file of config.ign. 

**************************************************
MacBook-Pro:k8s jaswang$ cd coreos-vagrant
MacBook-Pro:coreos-vagrant jaswang$


