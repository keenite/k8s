## Install kubeadm and setup a cluster
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



