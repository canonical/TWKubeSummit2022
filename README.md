# Kubeflow on microk8s

## Summary

In this tutorial, we will deploy kubeflow-lite & cos-lite on microk8s in a local virtual machine.

## Reqirement

* [multipass](https://multipass.run/)
* kvm and [virsh](https://www.libvirt.org/manpages/virsh.html)
* Virtual machine with 8GB memory, 4CPU and 50GB disk space

## Notes


### Create virtual machine

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


### Insufficient permissions to access MicroK8s

```sh
# add the user ubuntu to the 'microk8s' group
sudo usermod -a -G microk8s ubuntu
sudo chown -f -R ubuntu ~/.kube
# reload the user groups
newgrp microk8s
```


### Configure microk8s

Add alias in `~/.bash_aliases`

```sh
alias kubectl='microk8s kubectl'
```

```sh
microk8s status
microk8s enable dns hostpath-storage
microk8s status --wait-ready
```

### juju bootstrap

```sh
juju bootstrap microk8s micro --debug
microk8s kubectl get namespace | grep controller | awk '{print $1}' | xargs microk8s kubectl get all -n
juju controllers
juju clouds
```


### Deploy cos-lite

Enable metallb

```sh
IPADDR=$(ip -4 -j route | jq -r '.[] | select(.dst | contains("default")) | .prefsrc')
microk8s enable metallb:$IPADDR-$IPADDR
microk8s kubectl rollout status deployments/hostpath-provisioner -n kube-system -w
microk8s kubectl rollout status deployments/coredns -n kube-system -w
microk8s kubectl rollout status daemonset.apps/speaker -n metallb-system -w
```

```sh
juju add-model cos

curl -L https://raw.githubusercontent.com/canonical/cos-lite-bundle/main/overlays/offers-overlay.yaml -O
curl -L https://raw.githubusercontent.com/canonical/cos-lite-bundle/main/overlays/storage-small-overlay.yaml -O

juju deploy cos-lite \
  --trust \
  --channel=edge \
  --overlay ./offers-overlay.yaml \
  --overlay ./storage-small-overlay.yaml

juju status
```

```sh
# Get ip address for instance
multipass list

# 10.152.183.1/24 is the default ip range for microk8s cluster
sudo sshuttle -r ubuntu@$INSTANCE_IP_ADDRESS  10.1.123.0/24 --ssh-cmd "ssh -i /var/snap/multipass/common/data/multipassd/ssh-keys/id_rsa
```

To access cos-lite dashboard, :

http://10.1.24.70:9093, for alertmanager
http://10.1.24.71:3000, for grafana
http://10.1.24.73:9090, for prometheus

> Note: the IP addresses are almost certainly going to be different in your case.


```sh
# grafana password, user is admin
juju actions grafana

juju run-action --wait grafana/0 get-admin-password
```


## Create cross model relation between kubeflow-lite and cos-lite

```sh
juju list-offers
juju find-offers grafana-dashboards
juju find-offers prometheus-scrape
```

`create-cmr.sh`

```bash
#!/bin/bash
declare -a arr=(
        "argo-controller"
        "dex-auth"
        "jupyter-controller"
        "kfp-api"
        "metacontroller-operator"
        "minio"
        "seldon-controller-manager"
        "training-operator"
)

for i in "${arr[@]}"
do
        echo "create cross model relation for $i"
        juju relate $i:metrics-endpoint admin/cos.prometheus-scrape
        juju relate $i:grafana-dashboard admin/cos.grafana-dashboards
done
```

---

## Issues

* Promethes connect failure: `dex-auth`, `metacontroller-operator`, `minio`
* If can't create notebook on kubeflow, please check pod status in kubernetes.

---
## References

### juju

* [Juju | How to manage a cross-model relation](https://juju.is/docs/olm/manage-cross-model-relations)

### Microk8s

* [MetalLB, bare metal load-balancer for Kubernetes](https://metallb.universe.tf/)
* [How to chage a IP range · Issue #276 · canonical/microk8s · GitHub](https://github.com/canonical/microk8s/issues/276#issuecomment-687663776)

### kubeflow

* [GitHub - canonical/bundle-kubeflow: Charmed Kubeflow](https://github.com/canonical/bundle-kubeflow)
* [Quick start guide to Kubeflow | Documentation | Charmed Kubeflow](https://charmed-kubeflow.io/docs/quickstart)

### COS-lite

* [Juju | Deploy the COS Lite observability stack on MicroK8s](https://juju.is/docs/olm/lma-light)
* [GitHub - canonical/cos-lite-bundle: Canonical Observability Stack Lite, or COS Lite, is a light-weight, highly-integrated, Juju-based observability suite running on Kubernetes.](https://github.com/canonical/cos-lite-bundle)
