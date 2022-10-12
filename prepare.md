# Prepare

## Reqirement

* [multipass](https://multipass.run/)
* kvm and [virsh](https://www.libvirt.org/manpages/virsh.html)
* Virtual machine with 8GB memory, 4CPU and 50GB disk space

## Create virtual machine

```sh
sudo apt install libvirt-daemon-system

# switch multipass driver to libvirt
sudo snap connect multipass:libvirt
multipass stop --all
multipass set local.passphrase
sudo multipass authenticate
sudo multipass set local.driver=libvirt
multipass get local.driver

multipass launch \
    -c 4 -m 8G -d 50G \
    -n k8s-summit \
    -vvvv \
    https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
virsh list
multipass shell k8s-summit
```

### Install microk8s & juju with snap

Inside virtual machine

```sh
sudo snap install microk8s --channel=1.24/stable --classic
sudo snap install juju --classic

snap list
```

## Insufficient permissions to access MicroK8s

```sh
# add the user ubuntu to the 'microk8s' group
sudo usermod -a -G microk8s ubuntu
sudo chown -f -R ubuntu ~/.kube
# reload the user groups
newgrp microk8s
```


## Configure microk8s

Add alias in `~/.bash_aliases`

```sh
alias kubectl='microk8s kubectl'
```

Enable addons dns and hostpath-storage

```sh
microk8s status
microk8s enable dns hostpath-storage
microk8s status --wait-ready
```

Enable addons metallb

```sh
# IPADDR=$(ip -4 -j route | jq -r '.[] | select(.dst | contains("default")) | .prefsrc')
microk8s enable metallb:$IPADDR-$IPADDR
# Or microk8s enable metallb:10.64.140.43-10.64.140.49
```

```sh
# Check addons status with kubectl
microk8s kubectl rollout status deployments/hostpath-provisioner -n kube-system -w
microk8s kubectl rollout status deployments/coredns -n kube-system -w
microk8s kubectl rollout status daemonset.apps/speaker -n metallb-system -w
```

## juju bootstrap

```sh
juju bootstrap microk8s micro --debug
microk8s kubectl get namespace | grep controller | awk '{print $1}' | xargs microk8s kubectl get all -n
juju controllers
juju clouds
```

---

## References

### Microk8s

* [MetalLB, bare metal load-balancer for Kubernetes](https://metallb.universe.tf/)
* [How to chage a IP range · Issue #276 · canonical/microk8s · GitHub](https://github.com/canonical/microk8s/issues/276#issuecomment-687663776)
