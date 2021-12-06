# kubeadm_setup

***Provisioning K8s Cluster using Kubeadm****

sudo vi /etc/hosts
# add private ips of VM to hosts files across all VMs

cat << EOF | sudo tee /etc/modules-load.d/containerd.conf
> overlay
> br_netfilter
> EOF
# setup server to be able to use containerd engine...above input are kernel modules containerd requires.

sudo modprobe overlay
sudo modprobe br_netfilter
# make configuration changes take effect immediately

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
> net.bridge.bridge-nf-call-iptables	= 1
> net.ipv4.ip_forward	= 1
> net.bridge.bridge-nf-call-ip6tables	= 1
> EOF
#System level configuration required by containerd and k8s

sudo sysctl --system
# propagate and apply conf changes immediately

sudo apt-get update && sudo apt-get install -y containerd
# install containerd service

sudo mkdir -p /etc/containerd
# Dir to store containerd configurations

sudo containerd config default | sudo tee /etc/containerd/config.toml
# generate default containerd configuration 

sudo systemctl restart containerd

sudo swapoff -a

sudo sed -i '/ swap / s/^\(.*\)$/#|1/g' /etc/fstab

sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
> deb https://apt.kubernetes.io/ kubernetes-xenial main
> EOF

sudo apt-get update

sudo apt-get install -y kubelet=1.22.0-00 kubeadm=1.22.0-00 kubectl=1.22.0-00

sudo apt-mark hold kubelet kubectl kubeadm
# maintains present version by disabling auto update


++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
***Initialize cluster on the control plane/Master mach***

sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.22.0
# initalize the cluster with Kube network

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
# setup cluster using calico networking.

kubeadm token create --print-join-command
# copy token to the node VM to add then to the cluster.


++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
***Kubeadm upgrade***

kubectl drain <node-name> --ignore-damonsets
# to gracefully terminate or move pods on the specified node to another and disable scheduling

sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=1.22.2-00
# update repository and install kubeadm specified version

{
sudo kubeadm upgrade plan v1.22.2
sudo kubeadm upgrade apply v1.22.2

OR

sudo kubeadm upgrade node 
# for node 
}
sudo systemctl daemon-reload

sudo systemctl restart kubelet
# apply and immediate propagate config to system

kubectl uncordon <node name> 
# re-add the node to the cluster and make it available to pod schedulling 


***Install metric_server tool in cluster***
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands

kubectl apply -f https://raw.githubusercontent.com/ACloudGuru-Resources/content-cka-resources/master/metrics-server-components.yaml

kubectl get --raw /apis/metrics.k8s.io/
