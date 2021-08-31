# Ansible Setup Raspberry Pi OS

Overview: This will cover the installation of Ansible on the Raspberry Pi OS. It also will contain playbooks to install/setup Fedora with Kubernetes & Weave.

**Hardware Overview**
1. Raspberry Pi 
2. 5 Intell Nucs
   - Build Agent
   - Master Node
   - (3) Child Nodes 


> **Before Starting: Ensure each that each machine has a static ip and unique hostname** 
> Use `hostnamectl set-hostname 'foo'` to set a hostname

---

## Installation Steps for Ansible on Raspberry Pi

[Follow the full instructions here](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-specific-operating-systems)

```shell 
# add trusted repository from Ansible Documentation 
 echo "deb http://ppa.launchpad.net/ansible/ansible/ubuntu trusty main" | sudo tee -a /etc/apt/sources.list

# Add GPG Signing Key from Ubuntu
 sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 93C4A3FD7BB9C367

# update packages 
sudo apt update

# verify the latest version of ansible can be pull (2.9.XX)
sudo apt list -a ansible

# install ansible
sudo apt install ansible

# Check the ansible version
ansible --version
```

---

## Configure the Ansible Hosts File

This step lets Ansible know which machines we want it to manage.

```shell
# Navigate to Ansible Host file
cd /etc/ansible

# Edit host file & paste in the contents from the host file. 
sudo nano hosts
```

Generate an SSH Key that will be installed on the other machines

```shell
cd ~/.ssh
touch authorized_keys

#generate a ssh key (just press enter on each question)
ssh-keygen

# Set permissions of ssh folder and set permissions of authorized_keys to just be Pi user
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Add the SSH Public Key to authorized_keys file
cat id_rsa.pub >> authorized_keys

# Copy that authorized key to each nuc (from the Ansible server)
sudo ssh-copy-id user@hostname.example.com # or @ipaddress

# try pinging all the servers with 
ansible all -m -u username ping

# Once everything is working add the username to the /etc/ansible.cfg file. This will 
# prevent us from having to type it in manually everytime
remote_user = <SSH_USERNAME>

# rerun ping without the username flag
ansible all -m ping

```

---

## Using Ansible 

> Live Commands use "-a": Ex: `ansible all -a "/bin/echo hello"`
- Playbooks - Written using Yaml - ran via `ansible-playbook mytask.yaml`

```shell
# Create a dir for playbooks
cd ~
mkdir ansible-playbooks
cd ansible-playbooks

# copy/paste the updateFedora.yaml from this repo
sudo nano updateFedora.yaml

# Run the playbook. updateFedora.yaml has a value "become: yes" which tells it to run as sudo. --ask-become-pass will require you to enter a password after entering that command
ansible-playbook updateFedora.yaml --ask-become-pass

# Adhoc get fedora version. The updateFedora command should also patch linux and change it's kernal version
ansible nucs -a "cat /etc/fedora-release"
# Adhoc get Fedora Linux Kernal Version
ansible nucs -a "uname -mrs"

# TODO setup ansible lint to help verify playbooks (requires additional steps and pip or pipx)

```


---

## Preconfigure for Kubernetes

If all of the steps above worked correctly, Ansible should have updated the DNF packages and rebooted each machine. We are now ready to prep for the Kubernetes installation

```shell

# In the PXE Anaconda Install I skipped configuring the the network interface, I need to add the interface to the public zone. See https://github.com/natecowen/PiPXEServer for Pi PXE Server instructions

# Setup network interface in public zone
ansible nucs -a "firewall-cmd --permanent --zone=public --add-interface=enp2s0" --become --ask-become-pass
ansible nucs -a "firewall-cmd --list-ports --zone=public" --become --ask-become-pass

```

> [Helpful firewalld list](https://www.cyberithub.com/firewalld-examples-open-port/).


---

## Prepare for K8s Install. 

Run each item in order in the playbooks/k8s directory
```shell 
ansible-playbook 01disableswap.yaml --ask-become-pass
ansible-playbook 02prepForK8s.yaml --ask-become-pass
ansible-playbook 03firewallMaster.yaml --ask-become-pass
ansible-playbook 04firewallChildren.yaml --ask-become-pass
ansible-playbook 05containerdInstall.yaml --ask-become-pass

# test containerd
ansible all -a "ctr --version"

# Set Cgroup Driver to SystemD
# kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
# SSH into each machine and edit the CGroup Driver in config.toml
sudo nano -l /etc/containerd/config.toml

# Add under [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options] (around line 96 )
 SystemdCgroup = true

# On the Ansible machine run 
ansible all -a "systemctl restart containerd" --become --ask-become-pass
```

---

## Install K8s

```shell
# Install Kubelet, Kubectl, KubeAdmin
ansible-playbook 06K8sSetup.yaml --ask-become-pass

# Initialize the Master
ansible-playbook 07initMaster.yaml --ask-become-pass

# Did not fully test this playbook, but it's what i manually ran on master
# https://www.weave.works/docs/net/latest/kubernetes/kube-addon/
ansible-playbook 08weave.yaml --ask-become-pass

# : Join Nodes together
ansible-playbook 09joinMaster.yaml --ask-become-pass

# Connect to the Master node and verify nodes health
kubectl get nodes

#NOTE IF WEAVE WON'T START - Follow the instructions in weave.md

```

---


## Quickly Reset K8s

```shell

ansible k8children -a "kubeadm reset" --become --ask-become-pass

ansible k8children -a "rm -rf /etc/cni/net.d" --become --ask-become-pass


```

---

### Useful Links

- [Ansible Kubernetes Tutorial](https://buildvirtual.net/deploy-a-kubernetes-cluster-using-ansible/)