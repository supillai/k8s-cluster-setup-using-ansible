# Kubernetes setup using Ansible

For the last decade, I worked on different programming languages and frameworks but never got a chance to play around with Kubernetes aka k8s.  Now its the time :) 

Documentation on the k8s project page is excellent, but it is evolving very rapidly.  Now my immediate goal is to set up a k8s cluster and play around with it. One sure-fire way to learn something new is to play around with it on your playground.  

## How to setup a cluster?

There are many ways you can set up a k8s cluster for experiment and learning. You can use a cloud provider or VPS or set up locally.  In this instance, I am going to use VirtualBox to set up a k8s cluster. The reason I chose VirtualBox is, I can run the cluster on my laptop which is an Intel(R) Core(TM) i7-4810MQ CPU @ 2.80GHz (8 CPUs), ~2.8GHz running windows 10

I am going to create a Kubernetes cluster with a single control plane node and three worker nodes. Control plane and worker nodes are virtual machines running on VirtualBox with ubuntu server 20.04 focal fossa guests on Windows 10 host.  The control plane node is responsible for managing the state of the cluster whereas worker nodes are the servers where the workload is executed. The control plane node needs a minimum of 2 virtual CPUs.

## Can i automate some of the tasks?

I am a big fan of automating trivial tasks. I was exploring the possibility of automating many of the node setup tasks during the cluster creation. I found Ansible is a good framework to do the automation and am going to use the same throughout whenever possible. Cluster creation is fully automated using Ansible ( Virtual machine creation and setup host-only network is not part of Ansible script yet)

## What are the steps?
So on a high level, the following are the steps I am going to do. Expand the section to see details

<details>
  <summary>Install VirtualBox</summary>

  - Downlod and  VirtualBox-6.1.16 from https://www.virtualbox.org/wiki/Downloads
</details>
<details>
  <summary>Create virtual machines with ubuntu server 20.04 focal fossa</summary>
  
  - Downlod iso file from https://releases.ubuntu.com/20.04/
  - Create a new virtual machine using virtualbox using the iso downloaded in above step. Instructions for creating a VM can be found here https://oracle-base.com/articles/vm/virtualbox-creating-a-new-vm. 
    - Make sure to install openvpn server using operating system setup during vm creation
</details>
<details>
  <summary>Setup host-only network</summary>
  
</details>
<details>
  <summary>Disable swap on the control plane and worker nodes</summary>
  
</details>
<details>
  <summary>Install docker</summary>
  
</details>
<details>
  <summary>Install k8s</summary>
  
</details>
<details>
  <summary>Configure the control plane node</summary>
  
</details>
<details>
  <summary>Join the worker nodes with control plane node</summary>
  
</details>
<details>
  <summary>Enable shell auto completion</summary>
  
</details>
<details>
  <summary>Configure to use systemd as cgroupdriver for k8s</summary>
  
</details>








