# Set_Up_K8S_Cluster_On_Mac

In the article, we will describe how to set up a full Kubernetes cluster on a MacPro laptop. 

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
