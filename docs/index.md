- [Install kubeadm and setup a cluster](#install-kubeadm-and-setup-a-cluster)
  * [Install Docker as Container Runtime](#install-docker-as-container-runtime)
    + [Rootless mode](#rootless-mode)
  * [Install Kubenetes](#install-kubenetes)
  * [Add nodes to the cluster](#add-nodes-to-the-cluster)
  * [Install DashBoard](#install-dashboard)
  * [Install OpenFaas](#install-openfaas)
  * [Preinstall steps](#preinstall-steps)
- [OpenFass](#openfass)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# Install kubeadm and setup a cluster

## Install Docker as Container Runtime
[Install docker on ubuntu](https://docs.docker.com/engine/install/ubuntu/)
Install tool:
```
 sudo apt-get update
 sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
Add Docker’s official GPG key:
```
 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Add stable repo:
```bash
 echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Install Docker Engine
```
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
 sudo docker run hello-world
```

### Rootless mode
Must install uidmap and has at least 65536 subordinate UIDs/GIDs for the user.
```
sudo apt-get install uidmap
grep ^$(whoami): /etc/subuid
grep ^$(whoami): /etc/subgid
```
Then run the script after docker installed:
```
dockerd-rootless-setuptool.sh install
```

```
sudo sh -eux <<EOF
# Add subuid entry for wenji
echo "wenji:100000:65536" >> /etc/subuid
# Add subgid entry for wenji
echo "wenji:100000:65536" >> /etc/subgid
EOF
```

Add the installed output env to ~/.bashrc

Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups.
```
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Restart Docker and enable on boot:
```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Install Kubenetes
[link](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
Turn the swap partition off
```bash
sudo swapoff -a
sudo vim /etc/fstab # remove swap mount
```

Load br_netfilter driver
```bash
sudo modprobe br_netfilter
lsmod | grep br_netfilter
```
As a requirement for your Linux Node's iptables to correctly see bridged traffic, you should ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config, e.g.
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```
Download the Google Cloud public signing key:
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
Add the Kubernetes apt repository:
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Configure the Docker daemon, in particular to use systemd for the management of the container’s cgroups.

sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
Restart Docker and enable on boot:

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```
kubeadm reset
kubeadm init --pod-network-cidr=10.244.0.0/16
```

Control Plan need to make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Apply kubectl kube-flannel network
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## Add nodes to the cluster
Control plan will show the command used to join the cluster. e.g.
```
kubeadm join 10.211.55.76:6443 --token m5jelv.em1up9uyjv23jgrp \
        --discovery-token-ca-cert-hash sha256:dca9f19584c9af036edc7d6256d221b8b7c3a1332d69c2d6227e833f566b461a
```

kubectl show the existing nodes:
```
kubectl get nodes
```
Show dashboard
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
```

## Install DashBoard
Edit kubernetes-dashboard service.
```
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
```
You should see yaml representation of the service. Change type: ClusterIP to type: NodePort and save file. If it's already changed go to next step.
Next we need to check port on which Dashboard was exposed.
```
kubectl -n kubernetes-dashboard get service kubernetes-dashboard
```
The output is similar to this:
```
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   NodePort   10.100.124.90   <nodes>       443:31707/TCP   21h
```
Dashboard has been exposed on port 31707 (HTTPS). Now you can access it from your browser at: ```https://<master-ip>:31707```. master-ip can be found by executing kubectl cluster-info. Usually it is either 127.0.0.1 or IP of your machine, assuming that your cluster is running directly on the machine, on which these commands are executed.

In case you are trying to expose Dashboard using NodePort on a multi-node cluster, then you have to find out IP of the node on which Dashboard is running to access it. Instead of accessing ```https://<master-ip>:<nodePort>``` you should access ```https://<node-ip>:<nodePort>.```

Create a user as https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/README.md and paste token in authentication.
 
## Install OpenFaas
## Preinstall steps

Install faas-cli
```
curl -sLSf https://cli.openfaas.com | sudo sh
```
Login Docker
```
docker login
```
Install arkada and openfaas
```
curl -SLsf https://dl.get-arkade.dev/ | sudo sh
arkade install openfaas --load-balancer (arkade install openfaas)
```
Rollout gateway
```
kubectl rollout status -n openfaas deploy/gateway
kubectl port-forward svc/gateway -n openfaas 8080:8080
```
Get external-ip
```
kubectl get svc -o wide gateway-external -n openfaas
```
Login openfaas
```
export OPENFAAS_URL="" # Populate as above

# This command retrieves your password
PASSWORD=$(kubectl get secret -n openfaas basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode; echo)

# This command logs in and saves a file to ~/.openfaas/config.yml
echo -n $PASSWORD | faas-cli login --username admin --password-stdin
faas-cli list
```
Deploy functions and OpenFaas Web UI
```
faas-cli deploy -f https://raw.githubusercontent.com/openfaas/faas/master/stack.yml
http://10.211.55.76:31112/ui/
```
Show installed functions and invoke:
```
faas-cli list --verbose
echo Hi | faas-cli invoke markdown
```

# OpenFass
Build normal yml file but got error **Temporary failure resolving 'deb.debian.org'**
Need to change the mode for resolv file.
```
sudo service docker restart
(May need to run)
sudo chmod o+r /etc/resolv.conf
```
