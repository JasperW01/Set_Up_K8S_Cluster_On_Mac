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
  
      We choose to install K8s cluster step by step so that it can give you deeper understanding about the moving parts and we can benefit more in the long run. 
   
    Section C. Set up the k8s example of guestbook on the newly created k8s cluster. 

------------------------------------------------
A. SET UP 4 COREOS VMs WITH VAGRANT ON Mac
------------------------------------------------


The steps in this section are mainly sourced from  https://coreos.com/os/docs/latest/booting-on-vagrant.html. However some steps in this article are not consistent with the correpsonding artifects. As a consequence, we have corrected some steps according to the README.mod file clones from the corresponding GitHub repository and CoreOS common practice. 

Vagrant is a simple-to-use command line virtual machine manager. By working with VirtualBox (or VMWare which requires to purchase plug-in), it virtually turns your MacPro laptop into a datacentre.

Before we start the steps, we assume the Oracle VirtualBox and Vagrant have already been installed on the Mac laptop. VirtualBox for Mac can be downloaded from https://www.virtualbox.org/wiki/Downloads and Vagrant binary for Mac can be downloaded from https://www.vagrantup.com/downloads.html. 

Step A1. Clone the CoreOS-Vagrant repository from GitHub

On the MacPro laptop, clone the Vagrant + CoreOS repository released by CoreOS. 

    MacBook-Pro:~ jaswang$ mkdir k8s
    MacBook-Pro:~ jaswang$ cd k8s/
    MacBook-Pro:k8s jaswang$ git clone https://github.com/coreos/coreos-vagrant.git

Step A2. Request a new etcd discovery token

In our setting up, we will have 4 VMs and 1 of them will be running etcd server and the other three will be etcd clients. So we use the URL of  https://discovery.etcd.io/new?size=1 (please note size is 1 here) to request a new etcd discovery token and the token will be applied to all 4 VMs to be created. As a result, 1 VM will be dedicated to etcd server and the other 3 VMs will be etcd clients and installed with Kubernetes components. As per k8s best practice, it's strongly recommanded to have VMs dedicated to etcd cluster separate to the VMs used in K8S cluster. In our case, the etcd cluster is composed of 1 VM only to save resources of the Mac. 

Please note in the latest CoreOS Container Linux image, it's using etcd V3 (i.e. etcd-member.service) instead of etcd v2 (i.e. etcd2.service). 

    MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant
    MacBook-Pro:cab3eed2aa1a29c6bab1714a49b87dbb2oreos-vagrant jaswang$ curl https://discovery.etcd.io/new\?size\=1
    ab3eed2aa1a29c6bab1714a49b87dbb2 (record this string for later use)

Step A3. Update the human readiable cl.conf file and translate to CoreOS ignition file 

In the latest version of CoreOS, the traditional cloud-config to bootstrap CoreOS has been superceded by CoreOS Ignition. In particular, in the repo downloaded in Step 1, by default it's using Vagrant with VirtualBox and in particular it's expecting a Ignition file of config.ign file instead of a cloud-config file of user-data. So as per CoreOS common practice, we need to update the cl.conf in the cloned repository with the etcd discovery token retrieved in Step 2 above and use CoreOS config transpiler (i.e. ct tool) to translate cl.conf file to CoreOS ignition file of config.ign. 

    Download Mac binary of CoreOS config transpiler ct-<version>-x86_64-apple-darwin from https://github.com/coreos/container-linux-config-transpiler/releases. Copy it to /user/local/bin, change its name to "ct" and set it to be executable. 

    MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant
    MacBook-Pro:coreos-vagrant jaswang$ vi cl.conf
    (Replace <token> in the line of discovery with the etcd discovery token retrieved in Step 2)
    MacBook-Pro:coreos-vagrant jaswang$ ct --platform=vagrant-virtualbox < cl.conf > config.ign

Step A4. Set VM number and enable Docker Port Forwarding in config.rb file

In this step, we set up the number of VMs to be created by Vagrant as 4 and also enable Docker Port Forwarding so that later we can use the local Docker command on Mac to connect to the Docker engine inside the VMs created by Vagrant.

    MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant
    MacBook-Pro:coreos-vagrant jaswang$ mv config.rb.sample config.rb
    MacBook-Pro:coreos-vagrant jaswang$ vi config.rb
    (set $num_instances=4 and uncomment out the line of $expose_docker_tcp and change the number to 2370)

Step A5. Enable shared directory 

In this step, we will enable shared directory so that the VMs to be created by Vagrant can have visibility to the local directory of Mac. By this way, it's easy to get codes and Docker files from local Mac directory into CoreOS VMs. 

    MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant
    MacBook-Pro:coreos-vagrant jaswang$ vi Vagrantfile
    (Uncomment out the line of config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp'])

Step A6. Start CoreOS VMs using Vagrant's default VirtualBox provider

In this step, we will actually create and start CoreOS VMs using Vagrant's default VirtualBox provider. 

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

Step A7 Verify CoreOS VMs created by Vagrant

In this step, we verify the newly-created CoreOS VMs by ssh onto each VMs.

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

B. SET UP KUBERNETES CLUSTER ON MAC

Now we have 4 CoreOS VMs created on Mac with Vagrant & VirtualBox. In this section, we will set up Kubernetes cluster on those VMs step by step, in which core-02 is the master node and core-03 & core-04 are the worker nodes. 

The steps in this section are mainly sourced from https://coreos.com/kubernetes/docs/latest/getting-started.html with practical adjustment. Before start, we list the actual values of the following variables which will be used throughout this section. 

      MASTER_HOST=172.17.8.102 - IP address of the K8S master node core-02 which can be accessed by worker nodes and kubectl clien on Mac.
      ETCD_ENDPOINTS=http://172.17.8.101:2379 - List of etcd machines, comma separated. As only core-01 runs etcd so only 1 URL
      POD_NETWORK=10.2.0.0/16 - The CIDR network to use for pod IPs. the flannel overlay network will provide routing to this network.
      SERVICE_IP_RANGE=10.3.0.0/24 - The CIDR network to use for service cluster VIPs, routing is handled by local kube-proxy
      K8S_SERVICE_IP=10.3.0.1 - The VIP (Virtual IP) address of the Kubernetes API Service.
      DNS_SERVICE_IP=10.3.0.10 - The VIP (Virtual IP) address of the cluster DNS service. 

Step B1 Verify etcd Service status on CoreOS VMs & Troubleshooting when restart failing upon VM reboot

Kubernetes uses etcd service, which is a distributed key-value database, to store all kinds of configurations and status information. When the 4 CoreOS VMs were created in Step A7, etcd service (etcd v3) has been set up as 1 VM running etcd server and 3 VMs as etcd proxy/client. 

So in this step, we verify the etcd service is working well before setting up K8S components. 

First we log onto core-01 VM first to verfiy the etcd server is running on core-01 VM

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

Then we log onto each of the rest 3 VMs to verify the etcd proxy is working by the steps below: 

    MacBook-Pro:~ jaswang$ cd ~/k8s/coreos-vagrant
    MacBook-Pro:coreos-vagrant jaswang$ vagrant ssh core-02 -- -A
    core@core-02 ~ $ systemctl status etcd-member
    (Verify etcd-member service is running as etcd proxy/client instead of etcd server)
    
    ● etcd-member.service - etcd (System Application Container)
    ...
    Active: active (running) since Sun 2017-07-23 03:53:56 UTC; 9h ago
    ...
    Jul 23 03:53:56 core-02 etcd-wrapper[767]: 2017-07-23 03:53:56.464032 I | etcdmain: proxy: listening for client requests on     http://0.0.0.0:2379
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

Please note it's possible that the etcd-member service of proxy style on core-02 & 03 & 04 fails to start when VM reboot, even though the etcd-member of server style on core-01 starts successfully when VM reboot. In this case execute the following steps to trouble shoot: 

    core@core-02 ~ $ systemctl status etcd-member
    ● etcd-member.service - etcd (System Application Container)
       Loaded: loaded (/usr/lib/systemd/system/etcd-member.service; enabled; vendor preset: enabled)
      Drop-In: /etc/systemd/system/etcd-member.service.d
               └─20-clct-etcd-member.conf
       Active: activating (auto-restart) (Result: exit-code) since Wed 2017-08-02 13:12:54 UTC; 6s ago
         Docs: https://github.com/coreos/etcd
      Process: 13263 ExecStart=/usr/lib/coreos/etcd-wrapper $ETCD_OPTS --name=${COREOS_VAGRANT_VIRTUALBOX_HOSTNAME} --listen-peer-    urls=http://${COREOS_VAGRANT_VIRTUALBOX_PRIVATE_IPV4}:2380 --listen-client-url
      Process: 13252 ExecStartPre=/usr/bin/rkt rm --uuid-file=/var/lib/coreos/etcd-member-wrapper.uuid (code=exited, status=0/SUCCESS)
      Process: 13249 ExecStartPre=/usr/bin/mkdir --parents /var/lib/coreos (code=exited, status=0/SUCCESS)
     Main PID: 13263 (code=exited, status=1/FAILURE)
        Tasks: 0 (limit: 32768)
       Memory: 0B
          CPU: 0
       CGroup: /system.slice/etcd-member.service

    Aug 02 13:12:54 core-02 systemd[1]: Failed to start etcd (System Application Container).
    Aug 02 13:12:54 core-02 systemd[1]: etcd-member.service: Unit entered failed state.
    Aug 02 13:12:54 core-02 systemd[1]: etcd-member.service: Failed with result 'exit-code'.
    
    core@core-02 ~ $ journalctl -u etcd-member -f
    (The following error repeats)
    ...
    Aug 02 13:15:32 core-02 systemd[1]: etcd-member.service: Service hold-off time over, scheduling restart.
    Aug 02 13:15:32 core-02 systemd[1]: Stopped etcd (System Application Container).
    Aug 02 13:15:32 core-02 systemd[1]: Starting etcd (System Application Container)...
    Aug 02 13:15:32 core-02 rkt[14013]: "666f8d6f-3915-4b49-b658-0ba3dc3151d2"
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: ++ id -u etcd
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: + exec /usr/bin/rkt run --uuid-file-save=/var/lib/coreos/etcd-member-wrapper.uuid --trust-keys-from-https --mount volume=coreos-systemd-dir,target=/run/systemd/system --volume coreos-systemd-dir,kind=host,source=/run/systemd/system,readOnly=true --mount volume=coreos-notify,target=/run/systemd/notify --volume coreos-notify,kind=host,source=/run/systemd/notify --set-env=NOTIFY_SOCKET=/run/systemd/notify --volume coreos-data-dir,kind=host,source=/var/lib/etcd,readOnly=false --volume coreos-etc-ssl-certs,kind=host,source=/etc/ssl/certs,readOnly=true --volume coreos-usr-share-certs,kind=host,source=/usr/share/ca-certificates,readOnly=true --volume coreos-etc-hosts,kind=host,source=/etc/hosts,readOnly=true --volume coreos-etc-resolv,kind=host,source=/etc/resolv.conf,readOnly=true --mount volume=coreos-data-dir,target=/var/lib/etcd --mount volume=coreos-etc-ssl-certs,target=/etc/ssl/certs --mount volume=coreos-usr-share-certs,target=/usr/share/ca-certificates --mount volume=coreos-etc-hosts,target=/etc/hosts --mount volume=coreos-etc-resolv,target=/etc/resolv.conf --inherit-env --stage1-from-dir=stage1-fly.aci quay.io/coreos/etcd:v3.1.8 --user=232 -- --name=core-02 --listen-peer-urls=http://172.17.8.102:2380 --listen-client-urls=http://0.0.0.0:2379 --initial-advertise-peer-urls=http://172.17.8.102:2380 --advertise-client-urls=http://172.17.8.102:2379 --discovery=https://discovery.etcd.io/ab3eed2aa1a29c6bab1714a49b87dbb2
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.934767 I | pkg/flags: recognized and used environment variable ETCD_DATA_DIR=/var/lib/etcd
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.935060 I | pkg/flags: recognized environment variable ETCD_NAME, but unused: shadowed by corresponding flag
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.935228 W | pkg/flags: unrecognized environment variable ETCD_USER=etcd
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.935369 W | pkg/flags: unrecognized environment variable ETCD_IMAGE_TAG=v3.1.8
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.938642 I | etcdmain: etcd Version: 3.1.8
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.938818 I | etcdmain: Git SHA: d267ca9
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.938965 I | etcdmain: Go Version: go1.7.5
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.939095 I | etcdmain: Go OS/Arch: linux/amd64
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.939222 I | etcdmain: setting maximum number of CPUs to 1, total number of available CPUs is 1
    Aug 02 13:15:32 core-02 etcd-wrapper[14022]: 2017-08-02 13:15:32.939373 C | etcdmain: invalid datadir. Both member and proxy directories exist.
    Aug 02 13:15:32 core-02 systemd[1]: etcd-member.service: Main process exited, code=exited, status=1/FAILURE
    Aug 02 13:15:32 core-02 systemd[1]: Failed to start etcd (System Application Container).
    Aug 02 13:15:32 core-02 systemd[1]: etcd-member.service: Unit entered failed state.
    Aug 02 13:15:32 core-02 systemd[1]: etcd-member.service: Failed with result 'exit-code'.
    ...
    
    In this case, remove the sub directories of member then etcd will be started successfully. 
    
    core@core-02 ~ $ sudo rm -rf /var/lib/etcd/*
    core@core-02 ~ $ etcdctl member list
    f3c0b70e84d56c98: name=core-01 peerURLs=http://172.17.8.101:2380 clientURLs=http://172.17.8.101:2379 isLeader=true
    
(The permenant fix below needs to be re-visited????)
    To make a permenant fix, add one line in the etcd-member servcie drop-in file as shown below (on etcd proxy VMs only). 
    
    core@core-02 ~ $ cd /etc/systemd/system/etcd-member.service.d/
    core@core-02 /etc/systemd/system/etcd-member.service.d $ vi 20-clct-etcd-member.conf 
    core@core-02 /etc/systemd/system/etcd-member.service.d $ sudo vi 20-clct-etcd-member.conf 
    Add the line of "ExecStartPre=-/usr/bin/rkt rm -rf /var/lib/etcd/member" before the line of "ExecStart="

Step B2 - Generate Kubernetes TLS Assets

The Kubernetes API has various methods for validating clients. In this practice, we will configure the API server to use client certificate authentication. If we are in an enterprise which has an exising PKI infrastructure, we should follow the normal enterprise PKI procedure to create certificate requests and sign them with enterprise root certificate. In this practice, however, we will use openssl tool to create our own certificates as below: 
        
        Root CA Public & Private keys - ca.pem & ca-key.pam
        API Server Public & Private Keys - apiserver.pem & apiserver-key.pem
        Worker Node Public & Private Keys - ${WORKER_FQDN}-worker.pem & ${WORKER_FQDN}-worker-key.pem
        Cluster Admin Public & Private Keys - admin.pem & admin-key.pem

First we create the cluster root CA keys on one of the CoreOS VMs by the steps below: 

    core@core-01 ~ $ cd share/
    core@core-01 ~/share $ mkdir certificates
    core@core-01 ~/share $ cd certificates/
    core@core-01 ~/share $ openssl genrsa -out ca-key.pem 2048
    Generating RSA private key, 2048 bit long modulus       
    ...+++
    e is 65537 (0x10001)
    core@core-02 ~/share/certificates $ openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"

Then we create Kubernetes API Server Keypair

    core@core-02 ~/share/certificates $ vi openssl.cnf
    [req]
    req_extensions = v3_req
    distinguished_name = req_distinguished_name
    [req_distinguished_name]
    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = kubernetes
    DNS.2 = kubernetes.default
    DNS.3 = kubernetes.default.svc
    DNS.4 = kubernetes.default.svc.cluster.local
    IP.1 = 10.3.0.1
    IP.2 = 172.17.8.102
    core@core-02 ~/share/certificates $ openssl genrsa -out apiserver-key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    .......................................................................................................+++
    .+++
    e is 65537 (0x10001)
    core@core-02 ~/share/certificates $ openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-apiserver" -config openssl.cnf
    core@core-02 ~/share/certificates $ openssl x509 -req -in apiserver.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out apiserver.pem -days 365 -extensions v3_req -extfile openssl.cnf
    Signature ok
    subject=/CN=kube-apiserver
    Getting CA Private Key
    
Then we generates a unique TLS certificate for every Kubernetes worker node, i.e core-03 & core-04. 

    core@core-02 ~/share/certificates $ vi worker-openssl.cnf
    [req]
    req_extensions = v3_req
    distinguished_name = req_distinguished_name
    [req_distinguished_name]
    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    subjectAltName = @alt_names
    [alt_names]
    IP.1 = $ENV::WORKER_IP
    core@core-02 ~/share/certificates $ openssl genrsa -out core-03-worker-key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    ......................................................................+++
    .....................................+++
    e is 65537 (0x10001)
    core@core-02 ~/share/certificates $ WORKER_IP=172.17.8.103 openssl req -new -key core-03-worker-key.pem -out core-03-worker.csr -subj "/CN=core-03" -config worker-openssl.cnf
    core@core-02 ~/share/certificates $ WORKER_IP=172.17.8.103 openssl x509 -req -in core-03-worker.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out core-03-worker.pem -days 365 -extensions v3_req -extfile worker-openssl.cnf
    Signature ok
    subject=/CN=core-03
    Getting CA Private Key
    
    Repeat the above steps for core-04
    
Then we generate the Cluster Administrator Keypair, which will be used by the kubectl client to be set up in local MacPro host. 

    core@core-02 ~/share/certificates $ openssl genrsa -out admin-key.pem 2048
    Generating RSA private key, 2048 bit long modulus
    ............................+++
    ..................................................................................+++
    e is 65537 (0x10001)
    core@core-02 ~/share/certificates $ openssl req -new -key admin-key.pem -out admin.csr -subj "/CN=kube-admin"
    core@core-02 ~/share/certificates $ openssl x509 -req -in admin.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out admin.pem -days     365
    Signature ok
    subject=/CN=kube-admin
    Getting CA Private Key

Step B3 - Deploy K8S Master Node Components

In this step, we deploy K8S master node on core-02. 

First we prepare TSL certificates/assets on core-02

    core@core-02 ~ $ sudo mkdir -p /etc/kubernetes/ssl
    core@core-02 ~ $ cd /etc/kubernetes/ssl
    core@core-02 /etc/kubernetes/ssl $ sudo cp /home/core/share/certificates/ca.pem .
    core@core-02 /etc/kubernetes/ssl $ sudo cp /home/core/share/certificates/apiserver.pem .
    core@core-02 /etc/kubernetes/ssl $ sudo cp /home/core/share/certificates/apiserver-key.pem .
    core@core-02 /etc/kubernetes/ssl $ sudo chmod 600 *-key.pem

Then we configure flannel to source its local configuration in /etc/flannel/options.env and cluster-level configuration in etcd. 

    core@core-02 ~ $ sudo mkdir /etc/flannel
    core@core-02 ~ $ sudo vi /etc/flannel/options.env
    (Add the following two lines)
    FLANNELD_IFACE=172.17.8.102
    FLANNELD_ETCD_ENDPOINTS=http://172.17.8.101:2379

Next create the systemd drop-in for changing flanneld service setting. 

    core@core-03 ~ $ sudo netstat -nap|grep flanneld
    tcp        0      0 127.0.0.1:47748         127.0.0.1:2379          ESTABLISHED 1036/flanneld       
    tcp        0      0 127.0.0.1:47744         127.0.0.1:2379          ESTABLISHED 1036/flanneld       
    udp        0      0 10.0.2.14:8285          0.0.0.0:*                           1036/flanneld         
    core@core-02 /etc/kubernetes/ssl $ cd /etc/systemd/system/flanneld.service.d/
    core@core-02 /etc/systemd/system/flanneld.service.d $ sudo vi 40-ExecStartPre-symlink.conf
    (Add the following two lines)
    [Service]
    ExecStartPre=/usr/bin/ln -sf /etc/flannel/options.env /run/flannel/options.env
    core@core-02 /etc/systemd/system/flanneld.service.d $ sudo systemctl daemon-reload
    core@core-02 /etc/systemd/system/flanneld.service.d $ sudo systemctl restart flanneld
    core@core-02 ~ $ sudo netstat -nap|grep flanneld
    (After the change, flanneld now is connecting to etcd server on core-01)
    tcp        0      0 172.17.8.102:49880      172.17.8.101:2379       ESTABLISHED 2232/flanneld       
    tcp        0      0 172.17.8.102:49876      172.17.8.101:2379       ESTABLISHED 2232/flanneld       
    tcp        0      0 172.17.8.102:49878      172.17.8.101:2379       ESTABLISHED 2232/flanneld
    udp        0      0 172.17.8.102:8285       0.0.0.0:*                           2232/flanneld

In order for flannel to manage the pod network in the cluster, Docker needs to be configured to use flannel. So we need to do three things here: 

    a. use systend drop-in to configure flanneld running prior to Docker starting
    b. create Docker CNI (Containter Network Interface) options file
    c. set up flannel CNI configuration file (Please note, we choose to use Flannel instead of Calico for container networking)

    core@core-03 ~ $ systemctl status docker
    (Before change, docker service is not active)
    ● docker.service - Docker Application Container Engine
       Loaded: loaded (/run/torcx/unpack/docker/lib/systemd/system/docker.service; linked; vendor preset: disabled)
       Active: inactive (dead)
         Docs: http://docs.docker.com
    core@core-02 ~ $ sudo mkdir -p /etc/systemd/system/docker.service.d
    core@core-02 ~ $ cd /etc/systemd/system/docker.service.d
    core@core-02 /etc/systemd/system/docker.service.d $ sudo vi 40-flannel.conf
    (Add the following lines)
        [Unit]
        Requires=flanneld.service
        After=flanneld.service
        [Service]
        EnvironmentFile=/etc/kubernetes/cni/docker_opts_cni.env
    core@core-02 ~ $ sudo mkdir /etc/kubernetes/cni
    core@core-02 ~ $ cd /etc/kubernetes/cni/
    core@core-02 /etc/kubernetes/cni $ sudo vi docker_opts_cni.env
    (Add the following lines)
        DOCKER_OPT_BIP=""
        DOCKER_OPT_IPMASQ=""
    core@core-02 /etc/kubernetes/cni $ sudo mkdir net.d
    core@core-02 /etc/kubernetes/cni $ cd net.d/
    core@core-02 /etc/kubernetes/cni/net.d $ sudo vi 10-flannel.conf
    (Add the following lines)
        {
            "name": "podnet",
            "type": "flannel",
            "delegate": {
                "isDefaultGateway": true
            }
        }

Now we create kubelet unit on K8S master. The kubelet is the agent on each machine that starts and stops Pods and other machine-level tasks. The kubelet communicates with the API server (also running on the master nodes) with the TLS certificates we placed on disk earlier.

On the master node, the kubelet is configured to communicate with the API server, but not register for cluster work, as shown in the --register-schedulable=false line in the YAML excerpt below. This prevents user pods being scheduled on the master nodes, and ensures cluster work is routed only to task-specific worker nodes.Note that the kubelet running on a master node may log repeated attempts to post its status to the API server. These warnings are expected behavior and can be ignored. 

The following kubelet service unit file uses the following environment variables: 

    ${ADVERTISE_IP} = 172.17.8.102
    ${DNS_SERVICE_IP} = 10.3.0.10
    ${K8S_VER} =  v1.7.2_coreos.0
    
    core@core-02 ~ $ cd /etc/systemd/system
    core@core-02 /etc/systemd/system $ sudo vi kubelet.service
    (Add the following lines)
    [Service]
    Environment=KUBELET_IMAGE_TAG=v1.7.2_coreos.0
    Environment="RKT_RUN_ARGS=--uuid-file-save=/var/run/kubelet-pod.uuid \
      --volume var-log,kind=host,source=/var/log \
      --mount volume=var-log,target=/var/log \
      --volume dns,kind=host,source=/etc/resolv.conf \
      --mount volume=dns,target=/etc/resolv.conf"
    ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests
    ExecStartPre=/usr/bin/mkdir -p /var/log/containers
    ExecStartPre=-/usr/bin/rkt rm --uuid-file=/var/run/kubelet-pod.uuid
    ExecStart=/usr/lib/coreos/kubelet-wrapper \
      --api-servers=http://127.0.0.1:8080 \
      --register-schedulable=false \
      --cni-conf-dir=/etc/kubernetes/cni/net.d \
      --network-plugin=${NETWORK_PLUGIN} \
      --container-runtime=docker \
      --allow-privileged=true \
      --pod-manifest-path=/etc/kubernetes/manifests \
      --hostname-override=172.17.8.102 \
      --cluster_dns=10.3.0.10 \
      --cluster_domain=cluster.local
    ExecStop=-/usr/bin/rkt stop --uuid-file=/var/run/kubelet-pod.uuid
    Restart=always
    RestartSec=10

    [Install]
    WantedBy=multi-user.target

Now we set up the kube-apiserver Pod. The API server is where most of the magic happens. It is stateless by design and takes in API requests, processes them and stores the result in etcd if needed, and then returns the result of the request.

We're going to use a unique feature of the kubelet to launch a Pod that runs the API server. Above we configured the kubelet to watch a local directory for pods to run with the --pod-manifest-path=/etc/kubernetes/manifests flag. All we need to do is place our Pod manifest in that location, and the kubelet will make sure it stays running, just as if the Pod was submitted via the API. The cool trick here is that we don't have an API running yet, but the Pod will function the exact same way, which simplifies troubleshooting later on.

The following YAML file for api-service POD uses the following environment variables;

    ${ETCD_ENDPOINTS} = http://172.17.8.101:2379 
    ${SERVICE_IP_RANGE} = 10.3.0.0/24
    ${ADVERTISE_IP} = 172.17.8.102
    ${K8S_VER} =  v1.7.2_coreos.0
    
    core@core-02 ~ $ sudo mkdir /etc/kubernetes/manifests
    core@core-02 ~ $ cd /etc/kubernetes/manifests
    core@core-02 /etc/kubernetes/manifests $ sudo vi kube-apiserver.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: kube-apiserver
      namespace: kube-system
    spec:
      hostNetwork: true
      containers:
      - name: kube-apiserver
        image: quay.io/coreos/hyperkube:v1.7.2_coreos.0
        command:
        - /hyperkube
        - apiserver
        - --bind-address=0.0.0.0
        - --etcd-servers=http://172.17.8.101:2379
        - --allow-privileged=true
        - --service-cluster-ip-range=10.3.0.0/24
        - --secure-port=443
        - --advertise-address=172.17.8.102
        - --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
        - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
        - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
        - --client-ca-file=/etc/kubernetes/ssl/ca.pem
        - --service-account-key-file=/etc/kubernetes/ssl/apiserver-key.pem
        - --runtime-config=extensions/v1beta1/networkpolicies=true
        - --anonymous-auth=false
        livenessProbe:
          httpGet:
            host: 127.0.0.1
            port: 8080
            path: /healthz
          initialDelaySeconds: 15
          timeoutSeconds: 15
        ports:
        - containerPort: 443
          hostPort: 443
          name: https
        - containerPort: 8080
          hostPort: 8080
          name: local
        volumeMounts:
        - mountPath: /etc/kubernetes/ssl
          name: ssl-certs-kubernetes
          readOnly: true
        - mountPath: /etc/ssl/certs
          name: ssl-certs-host
          readOnly: true
      volumes:
      - hostPath:
          path: /etc/kubernetes/ssl
        name: ssl-certs-kubernetes
      - hostPath:
          path: /usr/share/ca-certificates
        name: ssl-certs-host

Similarly now we set up kube-proxy Pod. The proxy is responsible for directing traffic destined for specific services and pods to the correct location. The proxy communicates with the API server periodically to keep up to date. Both the master and worker nodes in K8S cluster will run the proxy. 

The following YAML file for kub-proxy POD uses the following environment variable;

    ${K8S_VER} =  v1.7.2_coreos.0
    
    core@core-02 ~ $ cd /etc/kubernetes/manifests
    core@core-02 /etc/kubernetes/manifests $ sudo vi kube-proxy.yaml
    core@core-02 /etc/kubernetes/manifests $ cat kube-proxy.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: kube-proxy
      namespace: kube-system
    spec:
      hostNetwork: true
      containers:
      - name: kube-proxy
        image: quay.io/coreos/hyperkube:v1.7.2_coreos.0
        command:
        - /hyperkube
        - proxy
        - --master=http://127.0.0.1:8080
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: ssl-certs-host
          readOnly: true
      volumes:
      - hostPath:
          path: /usr/share/ca-certificates
        name: ssl-certs-host

Next we set up POD YMAL file for kube-controller-manager. The controller manager is responsible for reconciling any required actions based on changes to Replication Controllers.

    ${K8S_VER} =  v1.7.2_coreos.0
    
    core@core-02 ~ $ cd /etc/kubernetes/manifests
    core@core-02 /etc/kubernetes/manifests $ sudo vi kube-controller-manager.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: kube-controller-manager
      namespace: kube-system
    spec:
      hostNetwork: true
      containers:
      - name: kube-controller-manager
        image: quay.io/coreos/hyperkube:v1.7.2_coreos.0
        command:
        - /hyperkube
        - controller-manager
        - --master=http://127.0.0.1:8080
        - --leader-elect=true
        - --service-account-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
        - --root-ca-file=/etc/kubernetes/ssl/ca.pem
        resources:
          requests:
            cpu: 200m
        livenessProbe:
          httpGet:
            host: 127.0.0.1
            path: /healthz
            port: 10252
          initialDelaySeconds: 15
          timeoutSeconds: 15
        volumeMounts:
        - mountPath: /etc/kubernetes/ssl
          name: ssl-certs-kubernetes
          readOnly: true
        - mountPath: /etc/ssl/certs
          name: ssl-certs-host
          readOnly: true
      volumes:
      - hostPath:
          path: /etc/kubernetes/ssl
        name: ssl-certs-kubernetes
      - hostPath:
          path: /usr/share/ca-certificates
        name: ssl-certs-host

Now we set up POD YAML file for kube-scheduler. The scheduler monitors the API for unscheduled pods, finds them a machine to run on, and communicates the decision back to the API.

    ${K8S_VER} =  v1.7.2_coreos.0
    
    core@core-02 ~ $ cd /etc/kubernetes/manifests
    core@core-02 /etc/kubernetes/manifests $ sudo vi kube-scheduler.yaml

Now that we've defined all of our units and written our TLS certificates to disk, we're ready to start the master components.

First, we need to tell systemd that we've changed units on disk and it needs to rescan & reload everything:

    core@core-02 /etc/kubernetes/manifests $ sudo systemctl daemon-reload

Earlier it was mentioned that flannel stores cluster-level configuration in etcd. Since we already started etcd in previuos steps, so now we need to configure our Pod network IP range and store it in etcd now. Since etcd was started earlier, 

    Environement variable values used here are: 
    ${POD_NETWORK}=10.2.0.0/16
    ${ETCD_ENDPOINTS} = http://172.17.8.101:2379 
     
    core@core-02 /etc/kubernetes/manifests $ etcdctl ls / --recursive
    /flannel
    /flannel/network
    /flannel/network/config
    /flannel/network/subnets
    /flannel/network/subnets/10.1.14.0-24
    /flannel/network/subnets/10.1.3.0-24
    core@core-02 /etc/kubernetes/manifests $ curl -X PUT -d "value={\"Network\":\"10.2.0.0/16\",\"Backend\":{\"Type\":\"vxlan\"}}" "http://172.17.8.101:2379/v2/keys/coreos.com/network/config"
    {"action":"set","node":{"key":"/coreos.com/network/config","value":"{\"Network\":\"10.2.0.0/16\",\"Backend\":{\"Type\":\"vxlan\"}}","modifiedIndex":56,"createdIndex":56}}
    core@core-02 /etc/kubernetes/manifests $ etcdctl ls / --recursive
    /flannel
    /flannel/network
    /flannel/network/config
    /flannel/network/subnets
    /flannel/network/subnets/10.1.14.0-24
    /flannel/network/subnets/10.1.3.0-24
    /coreos.com
    /coreos.com/network
    /coreos.com/network/config

After configuring flannel, we should restart it for our changes to take effect. Note that this will also restart the docker daemon and could impact running containers.

    core@core-02 ~ $ sudo systemctl stop flanneld
    core@core-02 ~ $ sudo systemctl start flanneld
    core@core-02 ~ $ sudo systemctl enable flanneld
    core@core-02 /etc/systemd/system/flanneld.service.d $ sudo systemctl start docker
    
Now that everything is configured, we can start the kubelet, which will also start the Pod manifests for the API server, the controller manager, proxy and scheduler.

    core@core-02 ~ $ sudo systemctl start kubelet
    core@core-02 ~ $ sudo systemctl enable kubelet
    Created symlink /etc/systemd/system/multi-user.target.wants/kubelet.service → /etc/systemd/system/kubelet.service.

Now we can do the following basic health check about the K8S master components we set. Later we will set up kubectl client natively on the local Mac laptop which will connect to the API service running on core-02 to manage the K8S cluster. 

    First we check the kubelet service is started and running properly. (this could take a few minutes after starting the kubelet.service)
    
    core@core-02 ~ $ systemctl status kubelet
    ● kubelet.service
       Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: disabled)
       Active: active (running) since Wed 2017-08-02 06:22:28 UTC; 9min ago
     Main PID: 3633 (kubelet)
        Tasks: 13 (limit: 32768)
       Memory: 93.8M
          CPU: 36.489s
       CGroup: /system.slice/kubelet.service
               ├─3633 /kubelet --api-servers=http://127.0.0.1:8080 --register-schedulable=false --cni-conf-dir=/etc/kubernetes/cni/net.d     --network-plugin= --container-runtime=docker --allow-privileged=true --
               └─3741 journalctl -k -f

    Aug 02 06:30:52 core-02 kubelet-wrapper[3633]: W0802 06:30:52.213861    3633 helpers.go:771] eviction manager: no observation found for
    ...
    
    Then we check Kube API service.
    
    core@core-02 ~ $ curl http://127.0.0.1:8080/version
    {
      "major": "1",
      "minor": "7",
      "gitVersion": "v1.7.2+coreos.0",
      "gitCommit": "c6574824e296e68a20d36f00e71fa01a81132b66",
      "gitTreeState": "clean",
      "buildDate": "2017-07-24T23:28:22Z",
      "goVersion": "go1.8.3",
      "compiler": "gc",
      "platform": "linux/amd64"
    }
    
    Next we check our Pods should be starting up and downloading their containers. Once the kubelet has started, you can check it's creating its pods via the metadata api. There should be four PODs, kube-apiserver, kube-controller-manager, kube-proxy and kube-scheduler. 
    
    core@core-02 ~ $ curl -s localhost:10255/pods | jq -r '.items[].metadata.name'
    kube-apiserver-172.17.8.102
    kube-controller-manager-172.17.8.102
    kube-proxy-172.17.8.102
    kube-scheduler-172.17.8.102
    
    core@core-02 /etc/systemd/system $ docker ps
    CONTAINER ID        IMAGE                                      COMMAND                  CREATED             STATUS              PORTS               NAMES
    d20f65d5eaac        quay.io/coreos/hyperkube                   "/hyperkube proxy ..."   6 minutes ago       Up 6 minutes                            k8s_kube-proxy_kube-proxy-172.17.8.102_kube-system_b746724e282bfb90131d57df30375f98_4
    fef5416eed3f        quay.io/coreos/hyperkube                   "/hyperkube apiser..."   6 minutes ago       Up 6 minutes                            k8s_kube-apiserver_kube-apiserver-172.17.8.102_kube-system_12327280fb3060d12bef125b7eec7345_4
    56649b7bcb32        quay.io/coreos/hyperkube                   "/hyperkube contro..."   6 minutes ago       Up 6 minutes                            k8s_kube-controller-manager_kube-controller-manager-172.17.8.102_kube-system_dd33222f8ef11e61dc945aea3f1da733_5
    c5c3a5ae943f        gcr.io/google_containers/pause-amd64:3.0   "/pause"                 6 minutes ago       Up 6 minutes                            k8s_POD_kube-controller-manager-172.17.8.102_kube-system_dd33222f8ef11e61dc945aea3f1da733_4
    a72a06d7589b        gcr.io/google_containers/pause-amd64:3.0   "/pause"                 6 minutes ago       Up 6 minutes                            k8s_POD_kube-proxy-172.17.8.102_kube-system_b746724e282bfb90131d57df30375f98_4
    a23b03cabb9a        gcr.io/google_containers/pause-amd64:3.0   "/pause"                 6 minutes ago       Up 6 minutes                            k8s_POD_kube-apiserver-172.17.8.102_kube-system_12327280fb3060d12bef125b7eec7345_4
    be440c4c9bda        quay.io/coreos/hyperkube                   "/hyperkube schedu..."   6 minutes ago       Up 6 minutes                            k8s_kube-scheduler_kube-scheduler-172.17.8.102_kube-system_fa3811c223367ac4d37eb181f83a8aac_5
    bcb25af8bd49        gcr.io/google_containers/pause-amd64:3.0   "/pause"                 6 minutes ago       Up 6 minutes                            k8s_POD_kube-scheduler-172.17.8.102_kube-system_fa3811c223367ac4d37eb181f83a8aac_4

At this point, we have successfully set up the K8S Master Node on core-02. Next we will set up Worker Node on core-03 & core-04. 

Step B4 - Deploy K8S Worker Node Components

In this step, we deploy K8S Worker Node on both core-03 & core-4. The detailed steps are shown as be executed on core-03 and just need to logically repeat the same on core-04. 

First we prepare TSL certificates/assets on core-03 for the Worker Node component

    core@core-03 ~ $ sudo mkdir -p /etc/kubernetes/ssl
    core@core-03 ~ $ cd /etc/kubernetes/ssl
    core@core-03 /etc/kubernetes/ssl $ sudo cp /home/core/share/certificates/ca.pem .
    core@core-03 /etc/kubernetes/ssl $ sudo cp /home/core/share/certificates/core-03-worker.pem .
    core@core-03 /etc/kubernetes/ssl $ sudo cp /home/core/share/certificates/core-03-worker-key.pem .
    core@core-03 /etc/kubernetes/ssl $ sudo chmod 600 *-key.pem

Then create symlinks to the worker-specific certificate and key so that the remaining configurations on the workers do not have to be unique per worker.

    core@core-03 ~ $ cd /etc/kubernetes/ssl/
    core@core-03 /etc/kubernetes/ssl $ sudo ln -s core-03-worker.pem worker.pem
    core@core-03 /etc/kubernetes/ssl $ sudo ln -s core-03-worker-key.pem worker-key.pem

Then we configure flannel to source its local configuration in /etc/flannel/options.env and cluster-level configuration in etcd. 

    core@core-03 ~ $ sudo mkdir /etc/flannel
    core@core-03 ~ $ sudo vi /etc/flannel/options.env
    (Add the following two lines)
    FLANNELD_IFACE=172.17.8.103
    FLANNELD_ETCD_ENDPOINTS=http://172.17.8.101:2379

Next create the systemd drop-in for changing flanneld service setting. 

    core@core-03 ~ $ sudo netstat -nap|grep flanneld
    tcp        0      0 127.0.0.1:47748         127.0.0.1:2379          ESTABLISHED 1036/flanneld       
    tcp        0      0 127.0.0.1:47744         127.0.0.1:2379          ESTABLISHED 1036/flanneld       
    udp        0      0 10.0.2.14:8285          0.0.0.0:*                           1036/flanneld         
    core@core-02 /etc/kubernetes/ssl $ cd /etc/systemd/system/flanneld.service.d/
    core@core-02 /etc/systemd/system/flanneld.service.d $ sudo vi 40-ExecStartPre-symlink.conf
    (Add the following two lines)
    [Service]
    ExecStartPre=/usr/bin/ln -sf /etc/flannel/options.env /run/flannel/options.env
    core@core-02 /etc/systemd/system/flanneld.service.d $ sudo systemctl daemon-reload
    core@core-02 /etc/systemd/system/flanneld.service.d $ sudo systemctl restart flanneld
    core@core-02 ~ $ sudo netstat -nap|grep flanneld
    (After the change, flanneld now is connecting to etcd server on core-01)
    tcp        0      0 172.17.8.102:49880      172.17.8.101:2379       ESTABLISHED 2232/flanneld       
    tcp        0      0 172.17.8.102:49876      172.17.8.101:2379       ESTABLISHED 2232/flanneld       
    tcp        0      0 172.17.8.102:49878      172.17.8.101:2379       ESTABLISHED 2232/flanneld
    udp        0      0 172.17.8.102:8285       0.0.0.0:*                           2232/flanneld

In order for flannel to manage the pod network in the cluster, Docker needs to be configured to use flannel. So we need to do three things here: 

    a. use systend drop-in to configure flanneld running prior to Docker starting
    b. create Docker CNI (Containter Network Interface) options file
    c. set up flannel CNI configuration file (Please note, we choose to use Flannel instead of Calico for container networking)

    core@core-03 ~ $ systemctl status docker
    (Before change, docker service is not active)
    ● docker.service - Docker Application Container Engine
       Loaded: loaded (/run/torcx/unpack/docker/lib/systemd/system/docker.service; linked; vendor preset: disabled)
       Active: inactive (dead)
         Docs: http://docs.docker.com
    core@core-03 ~ $ sudo mkdir -p /etc/systemd/system/docker.service.d
    core@core-03 ~ $ cd /etc/systemd/system/docker.service.d
    core@core-03 /etc/systemd/system/docker.service.d $ sudo vi 40-flannel.conf
    (Add the following lines)
        [Unit]
        Requires=flanneld.service
        After=flanneld.service
        [Service]
        EnvironmentFile=/etc/kubernetes/cni/docker_opts_cni.env
    core@core-03 ~ $ sudo mkdir /etc/kubernetes/cni
    core@core-03 ~ $ cd /etc/kubernetes/cni/
    core@core-03 /etc/kubernetes/cni $ sudo vi docker_opts_cni.env
    (Add the following lines)
        DOCKER_OPT_BIP=""
        DOCKER_OPT_IPMASQ=""
    core@core-03 /etc/kubernetes/cni $ sudo mkdir net.d
    core@core-03 /etc/kubernetes/cni $ cd net.d/
    core@core-03 /etc/kubernetes/cni/net.d $ sudo vi 10-flannel.conf
    (Add the following lines)
        {
            "name": "podnet",
            "type": "flannel",
            "delegate": {
                "isDefaultGateway": true
            }
        }

Now we create kubelet unit on workder node. The following kubelet service unit file uses the following environment variables: 

    ${MASTER_HOST} = 172.17.8.102
    ${ADVERTISE_IP} = 172.17.8.103
    ${DNS_SERVICE_IP} = 10.3.0.10
    ${K8S_VER} =  v1.7.2_coreos.0
    
    core@core-03 ~ $ cd /etc/systemd/system
    core@core-03 /etc/systemd/system $ sudo vi kubelet.service
    (Add the following lines)
    [Service]
    Environment=KUBELET_IMAGE_TAG=v1.7.2_coreos.0
    Environment="RKT_RUN_ARGS=--uuid-file-save=/var/run/kubelet-pod.uuid \
      --volume dns,kind=host,source=/etc/resolv.conf \
      --mount volume=dns,target=/etc/resolv.conf \
      --volume var-log,kind=host,source=/var/log \
      --mount volume=var-log,target=/var/log"
    ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests
    ExecStartPre=/usr/bin/mkdir -p /var/log/containers
    ExecStartPre=-/usr/bin/rkt rm --uuid-file=/var/run/kubelet-pod.uuid
    ExecStart=/usr/lib/coreos/kubelet-wrapper \
      --api-servers=https://172.17.8.102 \
      --cni-conf-dir=/etc/kubernetes/cni/net.d \
      --network-plugin=${NETWORK_PLUGIN} \
      --container-runtime=docker \
      --register-node=true \
      --allow-privileged=true \
      --pod-manifest-path=/etc/kubernetes/manifests \
      --hostname-override=172.17.8.103 \
      --cluster_dns=10.3.0.10 \
      --cluster_domain=cluster.local \
      --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml \
      --tls-cert-file=/etc/kubernetes/ssl/worker.pem \
      --tls-private-key-file=/etc/kubernetes/ssl/worker-key.pem
    ExecStop=-/usr/bin/rkt stop --uuid-file=/var/run/kubelet-pod.uuid
    Restart=always
    RestartSec=10
    
    [Install]
    WantedBy=multi-user.target

Now we set up kube-proxy Pod YAML file. 

The following YAML file for kub-proxy POD uses the following environment variable;

    ${MASTER_HOST} = 172.17.8.102
    ${K8S_VER} =  v1.7.2_coreos.0
    
    core@core-03 ~ $ sudo mkdir /etc/kubernetes/manifests
    core@core-02 ~ $ cd /etc/kubernetes/manifests
    core@core-02 /etc/kubernetes/manifests $ sudo vi kube-proxy.yaml
    core@core-02 /etc/kubernetes/manifests $ cat kube-proxy.yaml 
    apiVersion: v1
    kind: Pod
    metadata:
      name: kube-proxy
      namespace: kube-system
    spec:
      hostNetwork: true
      containers:
      - name: kube-proxy
        image: quay.io/coreos/hyperkube:v1.7.2_coreos.0
        command:
        - /hyperkube
        - proxy
        - --master=172.17.8.102
        - --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /etc/ssl/certs
          name: "ssl-certs"
        - mountPath: /etc/kubernetes/worker-kubeconfig.yaml
          name: "kubeconfig"
          readOnly: true
        - mountPath: /etc/kubernetes/ssl
          name: "etc-kube-ssl"
          readOnly: true
      volumes:
      - name: "ssl-certs"
        hostPath:
          path: "/usr/share/ca-certificates"
      - name: "kubeconfig"
        hostPath:
          path: "/etc/kubernetes/worker-kubeconfig.yaml"
      - name: "etc-kube-ssl"
        hostPath:
          path: "/etc/kubernetes/ssl"

In order to facilitate secure communication between Kubernetes components, kubeconfig can be used to define authentication settings. In this case, the kubelet and proxy are reading this configuration to communicate with the API.

    core@core-03 ~ $ cd /etc/kubernetes/
    core@core-03 /etc/kubernetes $ sudo vi worker-kubeconfig.yaml
    core@core-03 /etc/kubernetes $ cat worker-kubeconfig.yaml 
    apiVersion: v1
    kind: Config
    clusters:
    - name: local
      cluster:
        certificate-authority: /etc/kubernetes/ssl/ca.pem
    users:
    - name: kubelet
      user:
        client-certificate: /etc/kubernetes/ssl/worker.pem
        client-key: /etc/kubernetes/ssl/worker-key.pem
    contexts:
    - context:
        cluster: local
        user: kubelet
      name: kubelet-context
    current-context: kubelet-context

Now we can start the Worker services.

    core@core-03 ~ $ sudo systemctl daemon-reload
    core@core-03 ~ $ sudo systemctl start flanneld
    core@core-03 ~ $ sudo systemctl start kubelet
    core@core-03 ~ $ sudo systemctl start docker

Verify kubelet started and kube proxy also started. 

    core@core-03 ~ $ systemctl status kubelet
    ● kubelet.service
       Loaded: loaded (/etc/systemd/system/kubelet.service; disabled; vendor preset: disabled)
       Active: active (running) since Fri 2017-08-04 10:11:23 UTC; 8min ago
      Process: 1461 ExecStartPre=/usr/bin/rkt rm --uuid-file=/var/run/kubelet-pod.uuid (code=exited, status=254)
      Process: 1459 ExecStartPre=/usr/bin/mkdir -p /var/log/containers (code=exited, status=0/SUCCESS)
      Process: 1456 ExecStartPre=/usr/bin/mkdir -p /etc/kubernetes/manifests (code=exited, status=0/SUCCESS)
     Main PID: 1466 (kubelet)
        Tasks: 13 (limit: 32768)
       Memory: 157.0M
    ...
    core@core-03 ~ $ curl -s localhost:10255/pods | jq -r '.items[].metadata.name'
    kube-proxy-172.17.8.103
    core@core-03 ~ $ docker ps
    CONTAINER ID        IMAGE                                      COMMAND                  CREATED             STATUS              PORTS               NAMES
    3a6b49755f98        quay.io/coreos/hyperkube                   "/hyperkube proxy ..."   9 minutes ago       Up 9 minutes                            k8s_kube-proxy_kube-proxy-172.17.8.103_kube-system_16f5df290df73a44cb4049674da09067_0
    bfa49796eac8        gcr.io/google_containers/pause-amd64:3.0   "/pause"                 10 minutes ago      Up 10 minutes                           k8s_POD_kube-proxy-172.17.8.103_kube-system_16f5df290df73a44cb4049674da09067_0

Repeat the above on another Worker Node -core-04. 

Step B5 - Set Native Kubectl Client on MacPro

In this step, we will set up native Kubectl client on MacPro which connects to the API Server running on K8S Master Node core-01 to manage K8S cluster. 

In the terminal of the local MacPro, execute the following steps to download kubectl binary for MacOS and set it up. 

    $ curl -O https://storage.googleapis.com/kubernetes-release/release/v1.7.2/bin/darwin/amd64/kubectl
    $ chmod +x kubectl
    $ mv kubectl /usr/local/bin/kubectl

Now we configure kubectl to connect to the target cluster using the following commands. The following environment variables are used in those commands.

    ${MASTER_HOST}=172.17.8.102
    ${CA_CERT}=/Users/jaswang/k8s/coreos-vagrant/certificates/ca.pem
    ${ADMIN_KEY}=/Users/jaswang/k8s/coreos-vagrant/certificates/admin-key.pem
    ${ADMIN_CERT}=/Users/jaswang/k8s/coreos-vagrant/certificates/admin.pem

    MacBook-Pro:~ jaswang$ kubectl config set-cluster default-cluster --server=https://172.17.8.102 --certificate-authority=/Users/jaswang/k8s/coreos-vagrant/certificates/ca.pem
    Cluster "default-cluster" set.
    MacBook-Pro:bin jaswang$ kubectl config set-credentials default-admin --certificate-authority=/Users/jaswang/k8s/coreos-vagrant/certificates/ca.pem --client-key=/Users/jaswang/k8s/coreos-vagrant/certificates/admin-key.pem --client-certificate=/Users/jaswang/k8s/coreos-vagrant/certificates/admin.pem 
    User "default-admin" set.
    MacBook-Pro:bin jaswang$ kubectl config set-context default-system --cluster=default-cluster --user=default-admin
    Context "default-system" created.
    MacBook-Pro:bin jaswang$ kubectl config use-context default-system
    Switched to context "default-system".

Now check that the client is configured properly by using kubectl to inspect the cluster:
    
    MacBook-Pro:bin jaswang$ kubectl get nodes
    NAME           STATUS                     AGE       VERSION
    172.17.8.102   Ready,SchedulingDisabled   2d        v1.7.2+coreos.0
    172.17.8.103   Ready                      16h       v1.7.2+coreos.0




 
 



    
    


    



