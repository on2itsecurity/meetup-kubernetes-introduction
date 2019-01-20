# Introduction to Kubernetes

- [Introduction to Kubernetes](#introduction-to-kubernetes)
  - [Setting up a single-node cluster](#setting-up-a-single-node-cluster)
    - [Install CRI - Docker](#install-cri---docker)
    - [Install kubeadm, kubelet and kubectl](#install-kubeadm-kubelet-and-kubectl)
    - [Deploy the node](#deploy-the-node)
    - [Install CNI (Flannel)](#install-cni-flannel)
    - [Convert to single node cluster](#convert-to-single-node-cluster)
  - [Ingress controller](#ingress-controller)
  - [Deploy 'Whack-A-Pod'](#deploy-whack-a-pod)
    - [Get the contents from github](#get-the-contents-from-github)
    - [Deploy the game](#deploy-the-game)
    - [Access the game](#access-the-game)
    - [Clean up the deployment](#clean-up-the-deployment)
  - [(EXTRA) Add an additional node](#extra-add-an-additional-node)
    - [Install CRI - Docker](#install-cri---docker-1)
    - [Install kubeadm, kubelet and kubectl](#install-kubeadm-kubelet-and-kubectl-1)
    - [Join the cluster](#join-the-cluster)
  - [Tips](#tips)

## Setting up a single-node cluster

Prerequisites:

* Clean debian install
* add-user meetup
* Swap off 'sudo swapoff -a'

### Install CRI - Docker

```bash
sudo apt install -y ca-certificates software-properties-common curl apt-transport-https
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -

sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/debian \
$(lsb_release -cs) \
stable"

sudo apt update && sudo apt install -y docker-ce=18.06.0~ce~3-0~debian
```

### Install kubeadm, kubelet and kubectl

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

sudo apt update && sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

### Deploy the node

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

You will see something like this:
```text
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  sudo kubeadm join 172.16.153.131:6443 --token bfvz2a.ie09qb8tj256t9tu --discovery-token-ca-cert-hash sha256:63572357080e3d0da5693baa7c20d19bcd804c9f639dd20338a3249793081fe5
```

*Hint: Write down the kubeadm command, including the token en hash, it will come in handy when expanding the cluster*

```bash
mkdir -p /home/meetup/.kube
sudo cp -i /etc/kubernetes/admin.conf /home/meetup/.kube/config
sudo chown meetup:users /home/meetup/.kube/config
```

Take a look at your brand new kubernetes cluster:

```bash
kubectl get all --all-namespaces
```

*Notice that coredns will not start, cause the CNI (network) is missing.*

### Install CNI (Flannel)

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

### Convert to single node cluster

To make sure pods will also be scheduled on this node.

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Ingress controller

As an ingress controller 'nginx-ingress' will be used.
More information: https://kubernetes.github.io/ingress-nginx/deploy/

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```

## Deploy 'Whack-A-Pod'

More info: https://github.com/tpryan/whack_a_pod

### Get the contents from github

```bash
cd
git clone https://github.com/tpryan/whack_a_pod.git
```

### Deploy the game

Make sure the context is set, since the make command will use it.

```bash
kubectl config set-context $(kubectl config current-context) --namespace=default
```

Copy the Sample.properties to MAkefile.properties
```bash
cd ~/whack_a_pod
cp Sample.properties Makefile.properties
```

Edit DOCKERREPO in Makefile.properties

```bash
nano Makefile.properties

DOCKERREPO=cloudowski
```

Now we need to fix a bug in Whackamole, edit the file:

```text
nano ~/whack_a_pod/apps/ingress/ingress.generic.yaml
```

Make it look like this: (Changes: add the the ```$1``` at the end of the rewrite target, and the ```(.*)``` instead of ```*``` 3 times in the paths)

```text
# Copyright 2017 Google Inc. All Rights Reserved.

# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at

#     http://www.apache.org/licenses/LICENSE-2.0

# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: wap-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - http:
      paths:
      - path: /api/(.*)
        backend:
          serviceName: api
          servicePort: 8080
      - path: /admin/(.*)
        backend:
          serviceName: admin
          servicePort: 8080
      - path: /(.*)
        backend:
          serviceName: game
          servicePort: 8080
    host: whackapod.example.com
```

Deploy the game.

```bash
make deploy.generic
```

### Access the game

As you can see from the whack-a-pod directory in 'apps/ingress/ingress.generic.yaml', the hostname that is used by default is 'whackapod.example.com'.

To access the game, we need to add name to the hosts file of your windows jump host, with the IP of the node.

On windows, edit the file:
```text
C:\Windows\System32\drivers\etc\hosts
```

To include this line (make sure to replace the IP address with your node address)

```text
10.10.10.x1 whackapod.example.com
```



Lookup the NodePort on which the ingress controller is listening.

```bash
kubectl get service --namespace=ingress-nginx
```

In the below example it is: 32602

```text
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.100.75.185   <none>        80:32602/TCP,443:31226/TCP   20h
```

Go to the browser of your windows jumphost, and go to desired url:
* Full game - http://whackapod.example.com:32602
* Less busy version - http://whackapod.example.com:32602/next.html
* Advanced view - http://whackapod.example.com:32602/advanced.html

### Clean up the deployment

*To clean up*
```bash
make clean.generic
```

## (EXTRA) Add an additional node

* Login to the new node (host)

### Install CRI - Docker

```bash
sudo apt install -y ca-certificates software-properties-common curl apt-transport-https

curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -

sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/debian \
$(lsb_release -cs) \
stable"

sudo apt update && sudo apt install -y docker-ce=18.06.0~ce~3-0~debian
```

### Install kubeadm, kubelet and kubectl

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null

sudo apt update && sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

### Join the cluster

```bash
sudo kubeadm join 172.16.153.131:6443 --token bfvz2a.ie09qb8tj256t9tu --discovery-token-ca-cert-hash sha256:63572357080e3d0da5693baa7c20d19bcd804c9f639dd20338a3249793081fe5
```

With the following commands from the master (there is kubectl configured) you can check if the node is ready and the cluster is healthy.

```bash
kubectl get nodes
```

```bash
kubectl get componentstatus
```

## Tips 

* Kubectl auto-completion (tab tab)
```bash
source <(kubectl completion bash)
```
*Add this to '.bashrc' if you want this in every (new) session.*
