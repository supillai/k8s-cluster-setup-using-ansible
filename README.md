# Building a Kubernetes Cluster with MetalLB as load-balancer

For the last decade, I worked on different programming languages and frameworks but never got a chance to experiment with Kubernetes aka k8s. Documentation on the k8s project page is excellent, but it is evolving very rapidly.  Now my immediate goal is to set up a k8s cluster and setup MetalLB as load-balancer.

There are many ways you can set up a k8s cluster for experiment and learning. You can use a cloud provider or VPS or set up locally.  In this instance, I am going to use VirtualBox to set up a k8s cluster. The reason I chose VirtualBox is, I can run the cluster on my laptop which is an Intel(R) Core(TM) i7-4810MQ CPU @ 2.80GHz (8 CPUs), ~2.8GHz running windows 10

I am going to create a Kubernetes cluster with a single control plane node and three worker nodes. Control plane and worker nodes are virtual machines running on VirtualBox with ubuntu server 20.04 focal fossa guests on Windows 10 host.  The control plane node is responsible for managing the state of the cluster whereas worker nodes are the servers where the workload is executed. The control plane node needs a minimum of 2 virtual CPUs.

I have tested this on bare-metal setup and can confirm below instructions works there too. 

## Can i automate some of the tasks?

I was exploring the possibility of automating many of the node setup tasks during the cluster creation. I found Ansible is a good framework to do the automation and am going to use the same throughout whenever possible. Cluster creation is fully automated using Ansible.

## Prerequisites
  - Downlod and install VirtualBox-6.1.16 from https://www.virtualbox.org/wiki/Downloads
  - Install Ansible - If you are in windows, you can run below powershell script to install Ansible on Cygwwin. For others please follow platform specific instruction to install Ansible.
```powershell
Invoke-Expression ((new-object net.webclient).DownloadString('https://chocolatey.org/install.ps1'))
choco install cyg-get
cyg-get rsync ncurses openssh python37 python37-certifi python37-cryptography python37-jinja2 python37-jmespath python37-passlib python37-pypsrp python37-requests python37-urllib3 python37-winrm python37-yaml sshpass ansible
```
  - Get each server ready to run Kubernetes
    - Downlod iso file from https://releases.ubuntu.com/20.04/
    - Create a new virtual machine using virtualbox using the iso downloaded in above step. Instructions for creating a VM can be found here https://oracle-base.com/articles/vm/virtualbox-creating-a-new-vm
    - Setup host-only network - Instructions to create host-only network can be found here https://carleton.ca/scs/tech-support/troubleshooting-guides/host-only-adapter-on-virtualbox/ . Please note you do not need to enable DHCP server as specified in the article as our goal is to setup static ip addresses for the host.
    - Setup static ip addess on ubuntu server - In the newer version of Ubuntu, you have to use a tool called 'neplan' to manage network settings. You can create a new yaml file 'sudo vi /etc/netplan/01-netcfg.yaml' and place the following contents into it Please note the ip address you specify has to be in same subnet as host-only adapter

  ```yaml
  network:
  ethernets:
    enp0s8:
      dhcp4: false
      addresses: [192.168.56.21/24]
  version: 2
  ```
  
  Save the file and execute the following commands
  
  ```sh
  sudo netplan generate 01-netcfg.yaml
  sudo netplan apply 01-netcfg.yaml
  ```
  
  Now if you try 'ip a', it should list the static ip addess you specified associated with enp0s8 adapter. If you want your node to have internet access, make sure you chose default NAT adapter as primary network adapter and host-only adapter as the second one.  This way your host can SSH into your VM via private ip address you specified and VM can access the internet using NAT adapter


  - Configure SSH - Configure SSH access with the keypair, place the public key into the ~/.ssh/authorized_keys file for the user in each of the nodes and controlplane node. This is needed as by default Ansible uses SSH keys to connect to remote machines. More information about setting up SSH keys can be found here https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys#generating-and-working-with-ssh-keys

You can copy the public key into the ~/.ssh/authorized_keys using below powershell command. Replace user@ipaddress with valid values.
```powershell
type $env:USERPROFILE\.ssh\id_rsa.pub | ssh user@ipaddress "cat >> .ssh/authorized_keys"
```
## Steps to create a Kubernetes cluster

Following are the steps needed to setup a kubernetes cluster. Expand the node to see the details.

<details>
  <summary>Disable swap on the control plane and worker node</summary>

  You must disable swap in order for the kubelet to work properly. Discussion about the same can be found here https://github.com/kubernetes/kubernetes/issues/53533
  ```yaml
  - hosts: "{{hostlist}}"
    remote_user: "{{ansible_user}}"
    become: yes
    become_method: sudo
    gather_facts: yes
    connection: ssh

    tasks:
      - name: Disable swap
        command: swapoff -a

      - name: Permanently disable Swap entry from /etc/fstab
        lineinfile:
          dest: /etc/fstab
          regexp: swap
          state: absent
```
</details>

<details>
  <summary>Install docker on control plane and worker nodes</summary>

Below playbook installs docker and all the needed dependencies into the hosts specified as argument. This need to be done on control plane and worker nodes. Please note we customize docker config to use cgroupdriver=systemd and also set docker as system service.

  ```yaml
  - hosts: "{{hostlist}}"
    remote_user: "{{ansible_user}}"
    become: yes
    become_method: sudo
    gather_facts: yes
    connection: ssh

    tasks:
      - name: Installing Docker Dependencies
        apt:
          name:
            - apt-transport-https
            - ca-certificates
            - curl
            - software-properties-common
            - gnupg2
          state: present

      - name: Add Docker’s official GPG key
        apt_key:
          url: https://download.docker.com/linux/ubuntu/gpg
          keyring: /etc/apt/trusted.gpg.d/docker.gpg
          state: present

      - name: Add Docker Repository
        apt_repository:
          repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable
          state: present
          filename: docker
          mode: 0600

      - name: Install Docker CE
        apt:
          name:
            - docker-ce=5:19.03.11~3-0~ubuntu-focal
            - docker-ce-cli=5:19.03.11~3-0~ubuntu-focal
            - containerd.io=1.2.13-2
          state: present

      - name: Create Docker Daemon file
        copy:
          dest: "/etc/docker/daemon.json"
          content: |
            {
              "exec-opts": ["native.cgroupdriver=systemd"],
              "log-driver": "json-file",
              "log-opts": {
                "max-size": "100m"
              },
              "storage-driver": "overlay2"
            }
            EOF
  
      - name: Creates Docker Daemon directory
        file:
          path: /etc/systemd/system/docker.service.d
          state: directory
          mode: 0777

      - name: reload systemd
        command: systemctl daemon-reload

      - name: Enable service docker, and enable persistently
      service:
          name: docker
          enabled: yes
```
</details>

<details>
  <summary>Install k8s on control plane and worker nodes</summary>

Below playbook installs kubernetes and all the needed dependencies into the hosts specified as argument. This need to be done on control plane and worker nodes. Please note we set kubelet as system service.

  ```yaml
  - hosts: "{{hostlist}}"
    remote_user: "{{ansible_user}}"
    become: yes
    become_method: sudo
    gather_facts: yes
    connection: ssh

    tasks:
      - name: Add Google official GPG key
        apt_key:
          url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
          state: present

      - name: Add Kubernetes Repository
        apt_repository:
          repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
          state: present
          filename: kubernetes
          mode: 0600

      - name: Installing Kubernetes Cluster Packages.
        apt:
          name:
            - kubeadm
            - kubectl
            - kubelet
          state: present

      - name: Enable service kubelet, and enable persistently
        service:
          name: kubelet
          enabled: yes
```
</details>

<details>
  <summary>Configure the control plane node</summary>

Below playbook configure the controlplane node. You need to specify the Apiserver advertise address, which usually is the controlplane nodes ip address. You need to specify pod-network-cidr which should be in a different subnet than host. Kubernetes assigns each node a range of IP addresses, a CIDR block, so that each Pod can have a unique IP address. The size of the CIDR block corresponds to the maximum number of Pods per node. if you want to specify the pod network same as host, you can see the stackoverflow tip here https://stackoverflow.com/questions/45687310/is-it-possible-to-join-a-hardware-in-the-same-subnet-with-kubernetes-pods. 

Also you need to specify an overlay network, i use Flannel in this case. You can read more information about here https://kubernetes.io/docs/concepts/cluster-administration/networking/. 

### Note: When specifying --pod-network-cidr make sure you specify the same as in kube-flannel.yml, else MetalLB configuration doesn't work

  ```yaml
  - hosts: "{{hostlist}}"
    remote_user: "{{ansible_user}}"
    become: yes
    become_method: sudo
    gather_facts: yes
    connection: ssh

    vars_prompt:
      - name: "k8s_master_ip"
        prompt: "Enter the Apiserver advertise address, example: 192.168.0.130"
        private: no
        default: "192.168.0.130"

    tasks:
      - name: Intilizing Kubernetes Cluster
        command: kubeadm init --pod-network-cidr "192.168.57.0/24"  --apiserver-advertise-address "{{ k8s_master_ip }}" --v 5
        run_once: true
        delegate_to: "{{ k8s_master_ip }}"

      - pause: seconds=5

      - name: Create directory for kube config.
        become_method: sudo
        become: yes
        file:
          path: /home/{{ansible_user }}/.kube
          state: directory
          owner: "{{ ansible_user }}"
          group: "{{ ansible_user }}"
          mode: 0755

      - name: Copy /etc/kubernetes/admin.conf to user's home directory /home/{{ ansible_user }}/.kube/config.
        become_method: sudo
        become: yes
        copy:
          src: /etc/kubernetes/admin.conf
          dest: /home/{{ ansible_user }}/.kube/config
          remote_src: yes
          owner: "{{ ansible_user }}"
          group: "{{ ansible_user }}"
          mode: "0644"

      - pause: seconds=2

      - name: Create Pod Network & RBAC.
        become_user: "{{ansible_user}}"
        become_method: sudo
        become: yes
        command: "{{ item }}"
        with_items:
          - kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml --v 5
          - kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml --v 5
```
</details>

<details>
  <summary>Save the join token to temporary file</summary>

  ```yaml
    - hosts: "{{hostlist}}"
      remote_user: "{{ansible_user}}"
      become: yes
      become_method: sudo
      gather_facts: yes
      connection: ssh

      tasks:
        - name: Copy join command to worker nodes.
          become: yes
          become_method: sudo
          copy:
            src: /tmp/kubernetes_join_command
            dest: /tmp/kubernetes_join_command
            mode: 0777

       - name: Join the Worker nodes with the master.
          become: yes
          become_method: sudo
          command: sh /tmp/kubernetes_join_command
         register: joined_or_not

        - debug:
            msg: "{{ joined_or_not.stdout }}"
  ```
</details>

<details>
  <summary>Join the worker nodes with control plane node</summary>

  ```yaml
  - hosts: "{{hostlist}}"
    remote_user: "{{ansible_user}}"
    become: yes
    become_method: sudo
    gather_facts: yes
    connection: ssh

    tasks:
      - name: Copy join command to worker nodes.
        become: yes
        become_method: sudo
        copy:
          src: /tmp/kubernetes_join_command
          dest: /tmp/kubernetes_join_command
          mode: 0777

     - name: Join the Worker nodes with the master.
        become: yes
        become_method: sudo
        command: sh /tmp/kubernetes_join_command
       register: joined_or_not

      - debug:
          msg: "{{ joined_or_not.stdout }}"
  ```
</details>

<details>
  <summary>Enable shell auto completion for kubectl</summary>

  ```yaml
  - hosts: "{{hostlist}}"
    remote_user: "{{ansible_user}}"
    become: yes
    become_method: sudo
    gather_facts: yes
    connection: ssh

    tasks:
      - name: Configure kubectl command auto-completion.
        lineinfile:
          dest: /home/{{ ansible_user }}/.bashrc
          line: "source <(kubectl completion bash)"
          insertafter: EOF
  ```
</details>

<details>
  <summary>Setup systemd as cgroupdriver for k8s</summary>

  ```yaml
  - hosts: "{{hostlist}}"
    remote_user: "{{ansible_user}}"
    become: yes
    become_method: sudo
    gather_facts: yes
    connection: ssh

    tasks:
      - name: replace line
        lineinfile:
          path: /var/lib/kubelet/config.yaml
          regexp: "^(.*)cgroupDriver(.*)$"
          line: "cgroupDriver: systemd"
          backrefs: yes
      - name: Reboot all the kubernetes nodes.
        reboot:
          post_reboot_delay: 10
          reboot_timeout: 40
          connect_timeout: 60
          test_command: whoami
  ```
</details>

## How to run the playbook

Ansible playbook can be found here [init.yaml](Ansible/init.yaml). You must specify a list of hosts and vm user as parameter to each of the playbooks below.

You can run the playbook using the below command.  -K argument let you specify the sudo password.

-  ansible-playbook -i hosts init.yaml -K
## How to check installation went ok?

Run the below commands see similar output as below.

  ```sh
k8s@k8smaster:~$ k8s@k8smaster:~$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   6m16s
k8s@k8smaster:~$ kubectl get nodes
NAME         STATUS   ROLES    AGE     VERSION
k8smaster    Ready    master   6m24s   v1.19.4
k8sworker1   Ready    <none>   4m57s   v1.19.4
k8sworker2   Ready    <none>   4m55s   v1.19.4
k8s@k8smaster:~$
  ```
## Test deploying a container

SSH to control plane node , execute the below commands. similar output can be expected. 
  ```sh
k8s@k8smaster:~$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
k8s@k8smaster:~$ kubectl expose deploy nginx --port 80 --target-port 80 --type NodePort
service/nginx exposed
k8s@k8smaster:~$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP        9m22s
nginx        NodePort    10.98.52.3   <none>        80:32013/TCP   10s
k8s@k8smaster:~$kubectl scale deployment.v1.apps/nginx --replicas=2
deployment.apps/nginx scaled
  ```
  
  Now you should be able to access nginx home page by accessing http://< worker node ip>:32013

## What if installation fail?
-  Run 'kubeadm reset --ignore-preflight-errors=all' in all the nodes
-  Run playbook again 'ansible-playbook -i hosts init.yaml -K'

