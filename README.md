# Kubernetes setup details.

### Contents

- [Installing Kubernetes using installer provided by romana](#installing-kubernetes-using-installer-provided-by-romana)
- [Installing Kubernetes](#installing-kubernetes)
- [Installing Romana using provided containers](#installing-romana-using-provided-containers)
- [Scheduling Pods on Kubernetes master](#scheduling-pods-on-kubernetes-master)
  - [Pre Kubernetes 1.6](#pre-kubernetes-16)
  - [Post Kubernetes 1.6](#post-kubernetes-16)
  - [Retaint master node](#retaint-master-node)
- [Changing/Adding labels to nodes/pods](#changingadding-labels-to-nodespods)
- [Testing if the install is working](#testing-if-the-install-is-working)
- [Bringup and Expose service to external world](#bringup-and-expose-service-to-external-world)
  - [Removing cirros and nginx deployments](#removing-cirros-and-nginx-deployments)
  - [Removing kubernetes install](#removing-kubernetes-install)
  - [Deleting All docker Containers and Images](#deleting-all-docker-containers-and-images)
  - [Copy file form pod to host](#copy-file-form-pod-to-host)
- [Installing Romana using installer provided by it partially and rest manually](#installing-romana-using-installer-provided-by-it-partially-and-rest-manually)
  - [Some useful commands](#some-useful-commands)

## [Installing Kubernetes using installer provided by romana](#contents)

```bash
git clone https://github.com/romana/romana
cd romana/romana-install
./romana-setup -n test1 -c 3 -s kubeadm  --verbose install

# now ssh into the machines if you want.
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 1>
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 2>
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 3>

# Done, you have working romana installation..
```

## [Installing Kubernetes](#contents)

### Ubuntu
```bash
# On Controller
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y kubeadm
# Install older version of kubernetes packagesas follows if needed:
sudo apt-get install -y kubelet=1.6.7-00 kubeadm=1.6.7-00 kubectl=1.6.7-00
sudo kubeadm init
# or use a specific version using command below
sudo kubeadm init --kubernetes-version v1.7.0
# now you would get something like this at the end:
# sudo kubeadm join --token=<token> <ip-address:port>
# example:
# sudo kubeadm join --token 0f32b7.c003ad92878711b5 192.168.99.10:6443
kubectl get nodes -a -o wide --show-labels

# On Nodes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y kubeadm
# Use the kubeadm join command and token from controller here.
sudo kubeadm join --token=<token> <ip-address:port>
```

## [Installing Romana using provided containers](#contents)

```bash
wget https://raw.githubusercontent.com/romana/romana/master/containerize/specs/romana-kubeadm.yml
# make changes to defaults like cidr's, etc in romana-kubeadm.yml if wanted.
kubectl apply -f romana-kubeadm.yml
```

## [Scheduling Pods on Kubernetes master](#contents)

Removing taint from kubernetes master allows scheduling pods on master.
This is useful when single node is present.

### [Pre Kubernetes 1.6](#contents)

```bash
kubectl taint nodes --all dedicated:NoSchedule-
```

### [Post Kubernetes 1.6](#contents)

```bash
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-
```

### [Retaint Master Node](#contents)

```bash
kubectl taint node <node-name> node-role.kubernetes.io/master=:NoSchedule
```

## [Changing/Adding labels to nodes/pods](#contents)

```bash
# Add label by patching the node yaml.
kubectl patch node <node-name> -p '{"metadata":{"labels":{"node-role.kubernetes.io/master": ""}}}'

# Add label on cli for the node or pod.
kubectl  label nodes/ubuntu com.company.com/node=cluster1
kubectl  label po/nginx-dzpsr com.company.com/app=web

# Example Output:
kubectl get nodes,pods --show-labels
NAME        STATUS    ROLES     AGE       VERSION   LABELS
no/centos   Ready     <none>    3d        v1.9.4    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=centos
no/ubuntu   Ready     master    3d        v1.9.4    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,com.company.com/node=cluster1,kubernetes.io/hostname=ubuntu,node-role.kubernetes.io/master=

NAME             READY     STATUS              RESTARTS   AGE         LABELS
po/nginx-5pntv   0/1       ContainerCreating   0          <invalid>   app=nginx
po/nginx-dwf78   0/1       ContainerCreating   0          <invalid>   app=nginx
po/nginx-dzpsr   1/1       Running             0          <invalid>   app=nginx,com.company.com/app=web
po/nginx-vj9s9   1/1       Running             0          <invalid>   app=nginx
```

## [Testing if the install is working](#contents)

```bash
# First try checking the pods and see if all of them
# come up properly and are in running state. You may
# want to wait few minutes before it settles, there
# may be some restarts but eventually it does come up.
kubectl get pods -a -o wide --all-namespaces

# once all pods are up, try installing cirros and see
# if it comes up and if dns works correctly.
kubectl run cirros --image=cirros --replicas=4

# check if the cirros pods are running
kubectl get pods -a -o wide 

# once they are running login into them using
kubectl exec -it <pod-name> /bin/sh

# once inside the pod, use nslookup to test dns
nslookup kubernetes.default
# the result would be as follows:
#Server:    10.96.0.10
#Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
#
#Name:      kubernetes.default
#Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local

nslookup cirros-4036794762-9s46o
#Server:    10.96.0.10
#Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
#
#Name:      cirros-4036794762-9s46o
#Address 1: 100.112.11.131 cirros-4036794762-9s46o

```

## [Bringup and Expose service to external world](#contents)

```bash
# bringup nginx with 2 replicas on port 80.
kubectl run nginx --image=nginx --replicas=2 --port=80

# Check if the pods are up else wait few seconds for them
# to start, when desired and current columns both show
# correct number of replicas,  it means nginx is up and
# running correctly.
kubectl get deployments --all-namespaces

# Now expose the service to outside world.
kubectl expose deployment nginx --port=80 --type=LoadBalancer --external-ip=192.168.99.10

# Check if the configuration was applied correctly, if
# it was, then external IP should be reflected using
# following command.
kubectl get services --all-namespaces -o wide

# Test if nginx is up and running
curl 192.168.99.10:80
#<!DOCTYPE html>
#<html>
#<head>
#<title>Welcome to nginx!</title>
#<snip>...

# You are all set and ready to roll.
```

### [Removing cirros and nginx deployments](#contents)

```bash
kubectl delete deployments nginx cirros
```

### [Removing kubernetes install](#contents)

```bash
# On Controller and all nodes, run following command
# to reset your cluster to what it was before installing
# kubernetes
# BEWARE, YOU WILL LOSE ALL YOUR PODS/SERVICES/DATA/etc.
sudo kubeadm reset
```

### [Deleting All docker Containers and Images](#contents)

```bash
sudo docker rm $(sudo docker ps -a -q)
sudo docker rmi $(sudo docker images -a -q)
```

### [Copy file form pod to host](#contents)

```bash
sudo docker cp `kubectl get pods -a -o wide --all-namespaces --selector=app=<name> -o   jsonpath='{.items[*].status.containerStatuses[1].containerID}'| cut -d"/" -f3 | cut -c1-30`:/usr/local/bin/<file> /usr/local/bin/<file>
```

## [Installing Romana using installer provided by it partially and rest manually](#contents)

```bash
git clone https://github.com/romana/romana
cd romana/romana-install
./romana-setup -n test1 -c 3 -s nostack  --verbose install
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 1>
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 2>
ssh -i ~/romana/romana-install/romana_id_rsa ubuntu@<IP Address of the node 3>

# On First node
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y kubeadm
sudo kubeadm init
# or use a specific version using command below
sudo kubeadm init --kubernetes-version v1.7.0
# now you would get something like this at the end:
# sudo kubeadm join --token=<token> <ip-address:port>
# example:
# sudo kubeadm join --token 0f32b7.c003ad92878711b5 192.168.99.10:6443
# this is to be used on other nodes for joining kubernetes cluster.

# Install config for kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes -a -o wide --show-labels

# Install Romana
wget https://raw.githubusercontent.com/romana/romana/master/containerize/specs/romana-kubeadm.yml
kubectl create -f romana-kubeadm.yml

# Remove taints
kubectl taint nodes --all node-role.kubernetes.io/master:NoSchedule-

# On other Nodes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y docker.io
sudo apt-get install -y kubeadm
# Use the kubeadm join command and token from controller here.
sudo kubeadm join --token=<token> <ip-address:port>
```

### [Some useful commands](#contents)

```bash
watch -d kubectl get nodes -a -o wide
watch -d kubectl get pods,svc,rc -a -o wide --all-namespaces
watch -d kubectl get pods,rc,svc,ds,jobs,deploy -a -o wide --all-namespaces

# send bridge packets to iptables for further processing, needed by kubeadm (kubernetes)
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-ip6tables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl net.bridge.bridge-nf-call-iptables=1
sudo sysctl net.bridge.bridge-nf-call-ip6tables=1
```

