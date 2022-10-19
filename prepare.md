# Prepare

## Reqirement

* [multipass](https://multipass.run/)
* kvm and [virsh](https://www.libvirt.org/manpages/virsh.html) (Optional)
    * Virtual machine with 10GB memory, 6CPU and 50GB disk space

## Platform for virtual machine

Multipass: https://multipass.run/docs

* MacOS
  * Homebrew Installation if need
  ``` bash
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
  ```
* Windows
* Linux

## Create virtual machine

This will create a virtual machien which use 22.04 latest/stable image from offical website.

```sh
# create vm
multipass launch \
    -c 6 -m 10G -d 50G \
    -n k8s-summit \
    jammy

# list virtual machine
multipass list

# exec into vm
multipass shell k8s-summit
```

### Install microk8s & juju with snap

Inside virtual machine

```sh
# List package information and channels
snap info microk8s

sudo snap install microk8s --channel=1.24/stable --classic

# List snap packages & channel
snap list
```

## Insufficient permissions to access MicroK8s

```sh
# add the user ubuntu to the 'microk8s' group
sudo usermod -a -G microk8s ubuntu
sudo chown -f -R ubuntu ~/.kube
# reload the user groups or reboot
newgrp microk8s
```

check status 

```sh
microk8s status
```

## (Optional) microk8s alias

Add alias in `~/.bash_aliases`

```sh
alias kubectl='microk8s kubectl'
```

## Configure microk8s

Enable addons dns and hostpath-storage

```sh
microk8s status
microk8s enable dns hostpath-storage

# Check
microk8s status --wait-ready
```

> https://microk8s.io/docs/addon-dns
> https://microk8s.io/docs/addon-hostpath-storage

Check addons status with kubectl

```sh
microk8s kubectl rollout status deployments/hostpath-provisioner -n kube-system -w
microk8s kubectl rollout status deployments/coredns -n kube-system -w
```

## Install juju 

```sh
sudo snap install juju --classic
```

## juju bootstrap controller

In Juju, a controller is the initial cloud instance that is created by the juju client during bootstrapping. It is a central piece of Juju, being responsible for implementing all the changes defined by a Juju user.


```sh
juju bootstrap microk8s micro

# Check status
microk8s kubectl get namespace | grep controller | awk '{print $1}' | xargs microk8s kubectl get all -n
juju controllers
juju clouds
```

---

## References

### Microk8s

* [MetalLB, bare metal load-balancer for Kubernetes](https://metallb.universe.tf/)
* [How to chage a IP range · Issue #276 · canonical/microk8s · GitHub](https://github.com/canonical/microk8s/issues/276#issuecomment-687663776)
