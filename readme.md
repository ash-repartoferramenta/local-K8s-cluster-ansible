# Deploy a simple Kubernetes cluster using Kubeadm, vagrant and Ansible

## Before we begin
**Requirements**
- HashiCorp Vagrant for building and deploying virtual inftractructures
>  https://developer.hashicorp.com/vagrant/downloads
- Virtualbox as virtualization engine  (the selected vagrant box supports also paralles, libvirt, hyperv and VMware workstation but you need to edit vagrant file accordingly)
> https://www.virtualbox.org/wiki/Downloads
- Ansible for provisioning the VMs
>  https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

**Disclaimer** 
The following project has been realised for training and learning purpose and the material here provided  is only suitable for testing and learning enviroments;
please review the code carefully before utilizing it for any other scope.

## Implementation description


![ansible-k8s](https://github.com/ash-repartoferramenta/local-K8s-cluster-ansible/assets/135543207/36a42bac-8406-4a1d-a74f-6c04e112b3ae)


The following playbook deploy a kubernetes cluster using Kubeadm, Containerd and Flannel as CNI


## Vagrant

The  vagrant file available in the Vagrant folder deploy and builds recursivelty the number of VMs specified by the NODESNUMBER variable plus the master node. 
The Vm's will be running the generic/centos9s box from Vagrant Cloud; workernodes will be created with 2gb of ram and a vcpu assigned, while controlplane 
2gb and 2 vCpu (minimum suggested resources for running the cluster).
Subsequently vagrant will set up a private network the cluster  will use for internal communication.
For running the vagrant file you must travel to the Vagrant folder and use 
```
vagrant up
```
Vagrant will auto-generate an inventory for ansible with the informations provided in its Vagrantfile.
It's therefore important to set up correctly some variables.


**Security concerns:**
For the sake of ease of use, this Vagrantfile is set to utilize the default insecure ssh key provided by Vagrant

**Variables**
- NODESNUMBER : number of nodes (and VM'd) vagrant will build
- IPSTART: host part of the ipv4 adress from which vagrant will start assigning addresses to galera nodes (see example)
- VNETWORK: Network part of the ipv4 address vagrant will use to assign addresses to galera nodes (see example)

> Example:
> if NODESNUMBER is 3, VNETWORK is 192.168.1 and IPSTART is 5, 3 VMs will be created with the following addresses assigned:
> 192.168.1.6 , 192.168.1.7 , 192.168.1.8

**Current architectural concerns:**
Ips assignment  previously described is only based on string interpolation between the reported variables,
no further check is done so please review them carefully.


## Ansible

In the ansible folder, the following roles are described:
- containerd
- kubernetes
- crictl
- controlplane
- workernode

### containerd

This role installs the containerd container runtime, configuring it and its system modules and settings using the files present in the file folder.
After that sysctl variables and containerd service are reloaded and enabled.


### kubernetes

Install the required components on each node required to run and administer kubernetes cluster and disable swap on the controlplane and nodes.

### crictl

install a cli tool for inspect and debug container runtimes.

**Variables**
Latest version as of now is declared in the variable _clicrtllastv_ ,
please check out for new versions at https://github.com/kubernetes-sigs/cri-tools  

### Controlplane

Assign to the machine with the first address of the previously declared address block the role of controlplane. In addition flannel CNI is configured on this node and a join token is generated and passed to the workernodes.
The flannel configuration has been customised starting from the one provided from the original repository (https://github.com/flannel-io/flannel) due to the fact that Vagrant and its multiple network interfaces mess up the default flannel deploy. The edited config can be found in the files folder of this role.

**Variables**
- cpnetcidr: range of IP addresses for the pod network, used as --pod-network-cidr string parameter during cluster init

**Security concerns and notes:**
- As previously said right now the flannel install and config file is static so must be updated manually when newer versions comes out.
- The kubeadm init command as of now doesn't check if the cluster has already been initialised so if the playbook is run twice an error will be reported.
- This role also opens kube-control-plane related ports for controlplane communication with nodes.

### workernode

Join all the nodes to the cluster using the previously generated token
