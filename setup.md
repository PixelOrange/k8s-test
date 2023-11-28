# Kubernetes (K8s) Setup

###### Requirements

To follow this guide exactly, you will need the following VMs:

    1 leader - 2 vCPU / 2.5 GB RAM xUbuntu 22.04
    3 workers - 2 vCPU / 2.5 GB RAM xUbuntu 22.04

You will also need some basic software.
    
    PuTTY (or similar SSH software)

###### Preface

This guide assumes that you're using untouched servers with fresh Ubuntu installs. You 
can skip steps 2-4 if you already have a non-root user as your primary account.

#### Step 1 - Update APT

We need to make sure we have the latest packages before we continue.

```Upgrade APT
sudo apt full-upgrade
```

#### Step 2 - Create an account

Log in using PuTTY as `root` to create the new user account.

Create the user

```Create User
useradd pixelorange
```

Assign a password

```Set Password
passwd pixelorange
```

Add the user to the `sudo` group so you have admin access

```Set Permissions
sudo usermod -aG sudo pixelorange
```

Close the `root` user session.

```Log out
exit
```

#### Step 2 - Remove SSH login access

**WARNING:** Failure to perform this step properly could result in loss of access to your
VM. Please follow these steps carefully.

Log in as _the user you created_ using PuTTY or your favorite SSH client. Do **not** use 
`root`.

Open the `/etc/ssh/sshd_config` write-protected file in an editor

```Disable Root Login
sudo vi /etc/ssh/sshd_config
```

Search for `PermitRootLogin` with `/PermitRootLogin`. If it is not set to `no` or if it is
commented, change it to read as `PermitRoogLogin no`. Then save the file with `:wq`

Restart the `ssh` service

```Restart SSH service
sudo systemctl restart sshd
```

You can test that this worked as expected by attempting to SSH as `root`. It should not work.
You can still `su -` if you need to use the root account, but the `sudo` group should be able
do anything that `root` could do.

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

