# Remove Calico and Instead Install Flannel to Existing luster

Overview: Calico is a bit complex of a setup that requires stopping the firewalld service. Calico can be configured to act as this firewall, but I am not confident enough in my security/networking at this point. So I'm switching to Flannel. 



Flannel Requires a specific pod cidr to be set on init: 

kubeadm init --pod-network-cidr=10.244.0.0/16 

If you have already setup the cluster, we can change the pod cidr by doing the following on the master node:

```shell
cd /etc/kubernetes/manifests

sudo nano kube-controller-manager.yaml

 kubectl edit nodes nuc01k8m

# under specs:> containers: > -command: > -kub-controller-manager
--cluster-cidr=10.244.0.0/16 

kubeadm --pod-network-cidr=10.244.0.0/16 

# save and exit, then restart kubelet on master
systemctl restart kubelet
systemctl status kubelet

# verify cluster cider
ps -ef | grep "cluster-cidr"

# curl kube-flannel
curl https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml -O kube-flannel.yaml

kubectl apply -f kube-flannel.yaml

#reboot K8s child nodes

# add firewall rules for 
firewall-cmd --permanent --zone=public --add-port=8285/udp --add-port=8472/udp 
firewall-cmd --permanent --zone=public --add-port=8472/udp 
firewall-cmd --reload

# verify pods 
kubectl get pods -o wide --all-namespaces


kubectl patch node nuc01k8m nuc01k8c nuc02k8c nuc03k8c -p '{ "spec": { "podCIDR": "10.244.0.0/16" } }'

#kubectl edit 

# edit the nodes
 kubectl edit nodes nuc01k8m
 # add the following under spec and save
 '{ "spec": { "podCIDR": "10.244.0.0/16" } }'





#Clear IP routes with calico
sudo ip link list | grep cali | awk '{print $2}' | cut -c 1-15 | xargs -I {} sudo ip link delete {}

# remove calico configs
sudo rm /etc/cni/net.d/10-calico.conflist && sudo rm /etc/cni/net.d/calico-kubeconfig