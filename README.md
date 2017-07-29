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

A. SET UP 4 COREOS VMs WITH VAGRANT ON Mac

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

Step B1 Verify etcd Service status on CoreOS VMs

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
    core@core-02 ~ $ systemctl status docker
    ● docker.service - Docker Application Container Engine
       Loaded: loaded (/run/torcx/unpack/docker/lib/systemd/system/docker.service; linked; vendor preset: disabled)
       Active: active (running) since Sat 2017-07-29 02:11:50 UTC; 30min ago
         Docs: http://docs.docker.com
     Main PID: 2480 (dockerd)
        Tasks: 8
       Memory: 14.2M
          CPU: 1.463s
       CGroup: /system.slice/docker.service
               └─2480 /run/torcx/bin/dockerd --host=fd:// --containerd=/var/run/docker/libcontainerd/docker-containerd.sock --selinux-    enabled=true --bip=10.1.4.1/24 --mtu=1472 --ip-masq=false
    
    Jul 29 02:11:50 core-02 systemd[1]: Starting Docker Application Container Engine...
    Jul 29 02:11:50 core-02 env[2480]: time="2017-07-29T02:11:50.079921481Z" level=error msg="Failed to built-in GetDriver graph aufs  /var/lib/docker"
    Jul 29 02:11:50 core-02 env[2480]: time="2017-07-29T02:11:50.173229979Z" level=info msg="Graph migration to content-addressability     took 0.00 seconds"
    Jul 29 02:11:50 core-02 env[2480]: time="2017-07-29T02:11:50.173660858Z" level=info msg="Loading containers: start."
    Jul 29 02:11:50 core-02 env[2480]: time="2017-07-29T02:11:50.447055401Z" level=info msg="Loading containers: done."
    Jul 29 02:11:50 core-02 env[2480]: time="2017-07-29T02:11:50.468144470Z" level=info msg="Daemon has completed initialization"
    Jul 29 02:11:50 core-02 env[2480]: time="2017-07-29T02:11:50.469396879Z" level=info msg="Docker daemon" commit=89658be    graphdriver=overlay2 version=17.05.0-ce
    Jul 29 02:11:50 core-02 systemd[1]: Started Docker Application Container Engine.
    Jul 29 02:11:50 core-02 env[2480]: time="2017-07-29T02:11:50.481145428Z" level=info msg="API listen on [::]:2375"
    Jul 29 02:11:50 core-02 env[2480]: time="2017-07-29T02:11:50.481352682Z" level=info msg="API listen on /var/run/docker.sock"
    Warning: docker.service changed on disk. Run 'systemctl daemon-reload' to reload units.

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



    






 




    



  

    




