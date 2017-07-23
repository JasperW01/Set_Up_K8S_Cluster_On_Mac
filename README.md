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

Step 1. Clone the CoreOS-Vagrant repository from GitHub

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

MacBook-Pro:cab3eed2aa1a29c6bab1714a49b87dbb2oreos-vagrant jaswang$ curl https://discovery.etcd.io/new\?size\=1
ab3eed2aa1a29c6bab1714a49b87dbb2 (record this string for later use)
********************************************

Step 3. Update the human readiable cl.conf file and translate to CoreOS ignition file 

In the latest version of CoreOS, the traditional cloud-config to bootstrap CoreOS has been superceded by CoreOS Ignition. In particular, in the repo downloaded in Step 1, by default it's using Vagrant with VirtualBox and in particular it's expecting a Ignition file of config.ign file instead of a cloud-config file of user-data. So as per CoreOS common practice, we need to update the cl.conf in the cloned repository with the etcd discovery token retrieved in Step 2 above and use CoreOS config transpiler (i.e. ct tool) to translate cl.conf file to CoreOS ignition file of config.ign. 

**************************************************
Download Mac binary of CoreOS config transpiler ct-<version>-x86_64-apple-darwin from https://github.com/coreos/container-linux-config-transpiler/releases. Copy it to /user/local/bin, change its name to "ct" and set it to be executable. 

MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ vi cl.conf
(Replace <token> in the line of discovery with the etcd discovery token retrieved in Step 2)

MacBook-Pro:coreos-vagrant jaswang$ ct --platform=vagrant-virtualbox < cl.conf > config.ign
******************************************

Step 4. Set VM number and enable Docker Port Forwarding in config.rb file

In this step, we set up the number of VMs to be created by Vagrant as 4 and also enable Docker Port Forwarding so that later we can use the local Docker command on Mac to connect to the Docker engine inside the VMs created by Vagrant.

************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ mv config.rb.sample config.rb

MacBook-Pro:coreos-vagrant jaswang$ vi config.rb
(set $num_instances=4 and uncomment out the line of $expose_docker_tcp and change the number to 2370)
**********************************

Step 5. Enable shared directory 

In this step, we will enable shared directory so that the VMs to be created by Vagrant can have visibility to the local directory of Mac. By this way, it's easy to get codes and Docker files from local Mac directory into CoreOS VMs. 

************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ vi Vagrantfile
(Uncomment out the line of config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp'])
*************************************

Step 6. Start CoreOS VMs using Vagrant's default VirtualBox provider

In this step, we will actually create and start CoreOS VMs using Vagrant's default VirtualBox provider. 

************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ vagrant up
(During the process, it will prompt for the Mac local admin passport when setting up shared directory in VMs. Just key in the password as requested)

MacBook-Pro:coreos-vagrant jaswang$ vagrant status
Current machine states:

core-01                   running (virtualbox)
core-02                   running (virtualbox)
core-03                   running (virtualbox)
core-04                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
****************************************




