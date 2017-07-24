# Set_Up_K8S_Cluster_On_Mac

In the article, we will describe how to set up a full Kubernetes cluster on a MacPro laptop. The Mac we are using is at version 10.12.5. 

The steps are sourced from multiple articles to set up K8S cluster on a typical MacProc laptop with Vagrant & CoreOS, which has corrections to the errors/mismatches in the original artiticles and chains them together to be a full set of practical steps to 
set up a full Kubernetes cluster step by step. 

At the high level, there are three steps during the process: 

    Section A. Use Vagrant to create four CoreOS VMs on a MacPro laptop. 

    Section B. On those 4 CoreOS VMs, set up a Kubernetes cluster step by step with 
          i. one etcd server
          ii. one k8s master node
          iii. Two k8s worker nodes (more can be added afterwards)
     We choose to install K8s cluster step by step so that it can give you deeper understanding about the moving parts and we can benefit
   more in the long run. 
   
     Section C. Set up the k8s example of guestbook on the newly created k8s cluster. 

A. SET UP 4 COREOS VMs WITH VAGRANT ON Mac

The steps in this section are mainly sourced from  https://coreos.com/os/docs/latest/booting-on-vagrant.html. However some steps in this article are not consistent with the correpsonding artifects. As a consequence, we have corrected some steps according to the README.mod file clones from the corresponding GitHub repository and CoreOS common practice. 

Vagrant is a simple-to-use command line virtual machine manager. By working with VirtualBox (or VMWare which requires to purchase plug-in), it virtually turns your MacPro laptop into a datacentre.

Before we start the steps, we assume the Oracle VirtualBox and Vagrant have already been installed on the Mac laptop. VirtualBox for Mac can be downloaded from https://www.virtualbox.org/wiki/Downloads and Vagrant binary for Mac can be downloaded from https://www.vagrantup.com/downloads.html. 

Step A1. Clone the CoreOS-Vagrant repository from GitHub

On the MacPro laptop, clone the Vagrant + CoreOS repository released by CoreOS. 

*********************************************
MacBook-Pro:~ jaswang$ mkdir k8s

MacBook-Pro:~ jaswang$ cd k8s/

MacBook-Pro:k8s jaswang$ git clone https://github.com/coreos/coreos-vagrant.git
*********************************************

Step A2. Request a new etcd discovery token

In our setting up, we will have 4 VMs and 1 of them will be running etcd server and the other three will be etcd clients. So we use the URL of  https://discovery.etcd.io/new?size=1 (please note size is 1 here) to request a new etcd discovery token and the token will be applied to all 4 VMs to be created. As a result, 1 VM will be dedicated to etcd server and the other 3 VMs will be etcd clients and installed with Kubernetes components. As per k8s best practice, it's strongly recommanded to have VMs dedicated to etcd cluster separate to the VMs used in K8S cluster. In our case, the etcd cluster is composed of 1 VM only to save resources of the Mac. 

Please note in the latest CoreOS Container Linux image, it's using etcd V3 (i.e. etcd-member.service) instead of etcd v2 (i.e. etcd2.service). 

********************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:cab3eed2aa1a29c6bab1714a49b87dbb2oreos-vagrant jaswang$ curl https://discovery.etcd.io/new\?size\=1
ab3eed2aa1a29c6bab1714a49b87dbb2 (record this string for later use)
********************************************

Step A3. Update the human readiable cl.conf file and translate to CoreOS ignition file 

In the latest version of CoreOS, the traditional cloud-config to bootstrap CoreOS has been superceded by CoreOS Ignition. In particular, in the repo downloaded in Step 1, by default it's using Vagrant with VirtualBox and in particular it's expecting a Ignition file of config.ign file instead of a cloud-config file of user-data. So as per CoreOS common practice, we need to update the cl.conf in the cloned repository with the etcd discovery token retrieved in Step 2 above and use CoreOS config transpiler (i.e. ct tool) to translate cl.conf file to CoreOS ignition file of config.ign. 

**************************************************
Download Mac binary of CoreOS config transpiler ct-<version>-x86_64-apple-darwin from https://github.com/coreos/container-linux-config-transpiler/releases. Copy it to /user/local/bin, change its name to "ct" and set it to be executable. 

MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ vi cl.conf
(Replace <token> in the line of discovery with the etcd discovery token retrieved in Step 2)

MacBook-Pro:coreos-vagrant jaswang$ ct --platform=vagrant-virtualbox < cl.conf > config.ign
******************************************

Step A4. Set VM number and enable Docker Port Forwarding in config.rb file

In this step, we set up the number of VMs to be created by Vagrant as 4 and also enable Docker Port Forwarding so that later we can use the local Docker command on Mac to connect to the Docker engine inside the VMs created by Vagrant.

************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ mv config.rb.sample config.rb

MacBook-Pro:coreos-vagrant jaswang$ vi config.rb
(set $num_instances=4 and uncomment out the line of $expose_docker_tcp and change the number to 2370)
**********************************

Step A5. Enable shared directory 

In this step, we will enable shared directory so that the VMs to be created by Vagrant can have visibility to the local directory of Mac. By this way, it's easy to get codes and Docker files from local Mac directory into CoreOS VMs. 

************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ vi Vagrantfile
(Uncomment out the line of config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp'])
*************************************

Step A6. Start CoreOS VMs using Vagrant's default VirtualBox provider

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

Step A7 Verify CoreOS VMs created by Vagrant

In this step, we verify the newly-created CoreOS VMs by ssh onto each VMs.

************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ vagrant ssh core-01 -- -A

Last login: Sun Jul 23 12:19:31 UTC 2017 from 10.0.2.2 on pts/0
Container Linux by CoreOS alpha (1478.0.0)

core@core-01 ~ $ journalctl -f --lines=50
(Verify there is no error as shown below)
...
Jul 23 13:19:36 core-01 systemd[1258]: Reached target Basic System.

Jul 23 13:19:36 core-01 systemd[1258]: Reached target Default.

Jul 23 13:19:36 core-01 systemd[1258]: Startup finished in 21ms.

Jul 23 13:19:36 core-01 systemd[1]: Started User Manager for UID 500.

************************************

B. SET UP KUBERNETES CLUSTER ON MAC

Now we have 4 CoreOS VMs created on Mac with Vagrant & VirtualBox. In this section, we will set up Kubernetes cluster on those VMs step by step, in which core-02 is the master node and core-03 & core-04 are the worker nodes. 

The steps in this section are mainly sourced from https://coreos.com/kubernetes/docs/latest/getting-started.html with practical adjustment. Before start, we list the actual values of the following variables which will be used throughout this section. 

      MASTER_HOST=172.17.8.102 - IP address of the K8S master node core-02 which can be accessed by worker nodes and kubectl clien on Mac.
      ETCD_ENDPOINTS=http://172.17.8.101:2379 - List of etcd machines, comma separated. As only core-01 runs etcd so only 1 URL
      POD_NETWORK=10.2.0.0/16 - The CIDR network to use for pod IPs. the flannel overlay network will provide routing to this network.
      SERVICE_IP_RANGE=10.3.0.0/24 - The CIDR network to use for service cluster VIPs, routing is handled by local kube-proxy
      K8S_SERVICE_IP=10.3.0.1 - The VIP (Virtual IP) address of the Kubernetes API Service.
      DNS_SERVICE_IP=10.3.0.10 - The VIP (Virtual IP) address of the cluster DNS service. 

Step B1 Verify etcd Service status on CoreOS VMs

Kubernetes uses etcd service, which is a distributed key-value database, to store all kinds of configurations and status information. When the 4 CoreOS VMs were created in Step A7, etcd service (etcd v3) has been set up as 1 VM running etcd server and 3 VMs as etcd proxy/client. 

So in this step, we verify the etcd service is working well before setting up K8S components. 

First we log onto core-01 VM first to verfiy the etcd server is running on core-01 VM

****************************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ vagrant ssh core-01 -- -A

core@core-01 ~ $ etcdctl member list
(it should shown one etcd server is running on core-01 VM as shown below)

f3c0b70e84d56c98: name=core-01 peerURLs=http://172.17.8.101:2380 clientURLs=http://172.17.8.101:2379 isLeader=true

core@core-01 ~ $ systemctl status etcd-member.service
(Verify etcd-member service is running as etcdserver instead of etcd proxy, i.e. client)

● etcd-member.service - etcd (System Application Container)
   Loaded: loaded (/usr/lib/systemd/system/etcd-member.service; enabled; vendor preset: enabled)
  Drop-In: /etc/systemd/system/etcd-member.service.d
           └─20-clct-etcd-member.conf
           
   Active: active (running) since Sun 2017-07-23 03:53:37 UTC; 9h ago
   
 ...
 
Jul 23 12:40:28 core-01 etcd-wrapper[773]: 2017-07-23 12:40:28.726727 I | etcdserver: compacted raft log at 25003
*******************************************

Then we log onto each of the rest 3 VMs to verify the etcd proxy is working by the steps below: 

***********************************
MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant

MacBook-Pro:coreos-vagrant jaswang$ vagrant ssh core-02 -- -A

core@core-02 ~ $ systemctl status etcd-member

(Verify etcd-member service is running as etcd proxy/client instead of etcd server)

● etcd-member.service - etcd (System Application Container)
...
   Active: active (running) since Sun 2017-07-23 03:53:56 UTC; 9h ago
...
Jul 23 03:53:56 core-02 etcd-wrapper[767]: 2017-07-23 03:53:56.464032 I | etcdmain: proxy: listening for client requests on http://0.0.0.0:2379

core@core-02 ~ $ etcdctl ls / --recursive

(Verify the etcd tree can be displayed as below)

/flannel

/flannel/network

/flannel/network/config

/flannel/network/subnets

/flannel/network/subnets/10.1.55.0-24

core@core-02 ~ $ etcdctl set /message Hello

(Verify on K8S VM we can set etcd key and value pairs)

Hello

core@core-02 ~ $ etcdctl get /message

(Verify on K8S VM we can get etcd key and value pairs)

Hello

core@core-02 ~ $ curl http://172.17.8.101:2379/v2/keys/message

(Verify on K8S VM we can get etcd key and value pairs via etcd server URL)

{"action":"get","node":{"key":"/message","value":"Hello","modifiedIndex":16,"createdIndex":16}}

core@core-02 ~ $ etcdctl rm /message

(Verify on K8S VM we can remove etcd key and value pairs)

PrevNode.Value: Hello

core@core-04 /etc/systemd/system/etcd-member.service.d $ etcdctl cluster-health

member f3c0b70e84d56c98 is healthy: got healthy result from http://172.17.8.101:2379

cluster is healthy
****************************************

Step B2 - Generate Kubernetes TLS Assets

The Kubernetes API has various methods for validating clients. In this practice, we will configure the API server to use client certificate authentication. If we are in an enterprise which has an exising PKI infrastructure, we should follow the normal enterprise PKI procedure to create certificate requests and sign them with enterprise root certificate. In this practice, however, we will use openssl tool to create our own certificates as below: 
        
        Root CA Public & Private keys - ca.pem & ca-key.pam
        API Server Public & Private Keys - apiserver.pem & apiserver-key.pem
        Worker Node Public & Private Keys - ${WORKER_FQDN}-worker.pem & ${WORKER_FQDN}-worker-key.pem
        Cluster Admin Public & Private Keys - admin.pem & admin-key.pem




