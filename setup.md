# Kubernetes (K8s) Setup

###### Requirements

To follow this guide exactly, you will need the following VMs:

* 1 leader - 2 vCPU / 2.5 GB RAM xUbuntu 22.04
* 3 workers - 2 vCPU / 2.5 GB RAM xUbuntu 22.04

###### Preface

This guide assumes that you're using untouched servers with fresh Ubuntu installs. You 
can skip the first few steps if you already have a non-root user as your primary account.

#### Step 1 - Create an account

```Create User
useradd pixelorange
```

```Set Password
passwd pixelorange
```

```Set Permissions
sudo usermod -aG sudo pixelorange
```

#### Step - Disabling swap

While kubeadm 1.28 has beta support for using swap, we're still going to 
disable it.

Turn it off:
```Disable Swap
sudo swapoff -a
```

Persist the change through reboots by removing the swap line from fstab
```Persist
sed -i '/swap/d' /etc/fstab
```

