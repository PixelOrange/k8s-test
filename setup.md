# Kubernetes (K8s) Setup

## Requirements

To follow this guide exactly, you will need the following VMs:

* 1 leader - 2 vCPU / 2.5 GB RAM xUbuntu 22.04
* 3 workers - 2 vCPU / 2.5 GB RAM xUbuntu 22.04

You will also need some basic software.

**Installed on local machine:**
* PuTTY (or similar SSH software)

**Installed on server:**
```
apt-transport-https
ca-certificates
curl
net-tools
```

## Preface

This guide assumes that you're using untouched servers with fresh Ubuntu installs. You 
can skip steps 2-4 if you already have a non-root user as your primary account.

## Configuration
#### Step 1 - Update APT

We need to make sure we have the latest packages.

```Upgrade APT
sudo apt update
sudo apt full-upgrade
```

#### Step 2 - Create an account

Create the user.

```Create User
adduser pixelorange
```

Add the user to the `sudo` group so you have admin access.

```Set Permissions
usermod -aG sudo pixelorange
```

**Optional:** Ensure that your default shell is `/bin/bash` in `/etc/passwd`. This will
allow you to use autocomplete, reverse search, and scroll up through commands.

Open the file.

```Change shell
vi /etc/passwd
```

Search for the username with `/pixelorange` and change `/bin/sh` to `/bin/bash`. 

If the server told you to reboot for changes, do that now. Otherwise, close 
the `root` user session.

#### Step 3 - Remove SSH login access

**WARNING:** Failure to perform this step properly could result in loss of access to your
VM. Please follow these steps carefully.

Log in as _the user you created_ using PuTTY or your favorite SSH client. Do **not** use 
`root`.

Open the `/etc/ssh/sshd_config` write-protected file in an editor.

```Disable Root Login
sudo vi /etc/ssh/sshd_config
```

Search for `PermitRootLogin` with `/PermitRootLogin`. If it is not set to `no` or if it is
commented, change it to read as `PermitRoogLogin no`. Then save the file with `:wq`.

Restart the `ssh` service.

```Restart SSH service
sudo systemctl restart sshd
```

You can test that this worked as expected by attempting to SSH as `root`. It should not work.
You can still `su -` if you need to use the root account, but the `sudo` group should be able
do anything that `root` could do.

#### Step 4 - Disabling Swap

While kubeadm 1.28 has beta support for using swap, we're still going to 
disable it.

First check if it's on.

```Check Swap Status
swapon -s
```

Turn it off.

```Disable Swap
sudo swapoff -a
```
Remove any swap lines from `/etc/fstab`.

```Persist
sudo sed -i '/swap/d' /etc/fstab
```

On some flavors of Linux, you'll also need to disable it with `systemctl`. Find the swap
to disable.

```Find the swap
sudo systemctl --type swap --all
```

Mask the unit(s) listed.

```Mask Swap
sudo systemctl mask dev-vda3.swap
```

Reboot so the changes to swap can take effect.

```Restart
sudo reboot
```

#### Step 5 - Configure iptables

Add the `overlay` and `br_netfilter` modules to `/etc/modules-load.d/crio.conf` so
they are loaded for CRI-O.

```crio.conf
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF
```

Enable the modules.

```Enable modules
sudo modprobe overlay
sudo modprobe br_netfilter
```

Enable `net.bridge` and `ip_forward` so the pods can talk to each other.

```Enable network capabilities
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Restart the system without reboot.

```Restart no reboot
sudo sysctl --system
```

#### Step 6 - Enable UFW

Enable the firewall to harden the nodes.

```Enable UFW
sudo ufw enable
```

**IMPORTANT:** Make sure you add port 22 for SSH.

```SSH
sudo ufw allow 22/tcp
```

#### Step 7 - Calico CNI

Add the ports necessary for the Calico CNI. All the servers need these ports.

```Calico Ports
sudo ufw allow 179/tcp
sudo ufw allow 2379/tcp
sudo ufw allow 4789/udp
sudo ufw allow 4789/tcp
```

* Port 179/tcp  is used by the K8s API server for communication with the etcd datastore
* Port 2379/tcp is used for the etcd datastore for communication between the cluster nodes
* Port 4789/udp is used by the K8s networking plugin for overlay networking
* Port 4789/tcp is used by the K8s networking plugin for overlay networking

#### Step 8 - LEADER ONLY - Enable ports

Allow the ports that K8s needs for the control plane. Do not add these to the worker nodes.

```Allow ports
sudo ufw allow 2379:2380/tcp
sudo ufw allow 6443/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 10259/tcp
```

* Port 2379:2380/tcp is for the ETCD server client API.
* Port 6443/tcp is for the Kubernetes API server.
* Port 10250/tcp is for the Kubelet API.
* Port 10257/tcp is for the Kube controller manager.
* Port 10259/tcp is for the kube scheduler.

#### Step 9 - WORKER NODES - Enable Ports

```Allow ports
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
```

* Port 10250/tcp is for the Kubelet API.
* Port 30000:32767/tcp is for Node Port Services.

#### Step 10 - Install CRI-O

CRI-O is the container runtime this guide will use.

First, configure some environment variables.

```Variables
OS="xUbuntu_22.04"
CRIO_VERSION="1.28"
```

Then add the repositories to the package manager.

```Package Manager
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /"|sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$CRIO_VERSION/$OS/ /"|sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$CRIO_VERSION.list
```

Add the GPG key for CRI-O.

```GPG Key
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$CRI_VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
```

Update the package manager.

```Update APT
sudo apt update
```

Install the CRI-O packages.

```CRI-O packages
sudo apt install cri-o cri-o-runc cri-tools -y
```

Enable the systemd configurations and CRI-O.

```Enable packages
sudo systemctl daemon-reload
sudo systemctl enable crio --now
```

Check to make sure that CRI-O is configured properly.

```crictl
sudo crictl info
sudo crictl version
```

#### Step 11 - Install Kubeadm

Download dependencies for K8s. Some may already be installed.

```Dependencies
sudo apt install -y apt-transport-https ca-certificates curl net-tools
```

Download the GPG key for the K8s APT repository.

```K8s GPG key
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://dl.k8s.io/apt/doc/apt-key.gpg
```

Add the K8s APT repository.

```K8s APT repository
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update the repository.

```Update
sudo apt update
sudo apt full-upgrade -y
```

Get the latest version of K8s.

```Latest Version
apt-cache madison kubeadm | tac
```

Install either a specific version or the latest.

```Latest
sudo apt install -y kubelet kubeadm kubectl
```

Prevent the packages from receiving updates.

```Add holds
sudo apt-mark hold kubelet kubeadm kubectl
```

Enable the kubelet

```Enable kubelet
sudo systemctl enable kubelet
```

Pull the images with kubeadm

```kubeadm image pull
sudo kubeadm config images pull
```

#### Step 12 - LEADER ONLY - Initialize Kubernetes

Set some basic variables

```Variables
IPADDR=$(curl ifconfig.me && echo "")
NODENAME=$(hostname -s)
POD_CIDR="192.168.0.0/16"
```

Initialize the cluster

```kubeadm init
sudo kubeadm init --control-plane-endpoint=$IPADDR  \
--apiserver-cert-extra-sans=$IPADDR  --pod-network-cidr=$POD_CIDR \
--cri-socket unix:///var/run/crio/crio.sock --node-name $NODENAME \
--ignore-preflight-errors Swap
```

Configure .kube/config

```kube/config
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

If you have K8s installed on your local machine, run this in CLI

```SCP config
sudo scp pixelorange@198.46.203.200:~/.kube/config ~/.kube/config
sudo chown pixelorange ~/.kube/config
sudo chgrp pixelorange ~/.kube/config
sudo chmod 770 ~/.kube/config
```

Confirm pods were created. Two should be in `ContainerCreating` status.

```Check pods
kubectl get po -n kube-system
```

Verify the component health.

```Component Health
kubectl get --raw='/readyz?verbose'
```

Get cluster info.

```Cluster Info
kubectl cluster-info
```

#### Step 13 - LEADER ONLY - Install Calico

Get the latest Calico YAML from the manifest tab of
https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises

```Calico YAML
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.4/manifests/calico.yaml -O
```

Apply it!

```Apply Calico
kubectl apply -f calico.yaml
```

Check on the Calico pod status

```Pod Status
watch kubectl get pods -A
```

Get the join command

```K8s join
sudo kubeadm token create --print-join-command
```

#### Step 14 - WORKERS - Join cluster

Join the worker nodes to the cluster

```Join command
kubeadm join 198.46.203.200:6443 --token REDACTED --discovery-token-ca-cert-hash REDACTED
```

#### Step 15 - Test with NGINX

Deploy NGINX

```NGINX Deploy
kubectl create deploy nginx-web-server --image nginx
```

Expose the service

```Expose NGINX
kubectl expose deploy nginx-web-server --port 80 --type NodePort
```