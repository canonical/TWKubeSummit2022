## Deploy kubeflow + mlflow


### Upstream too many open files issue

```sh
# workaround for upstream issues in kubeflow:
# https://github.com/kubeflow/manifests/issues/2087#issuecomment-1139260511
$ sudo sysctl fs.inotify.max_user_instances=1280
$ sudo sysctl fs.inotify.max_user_watches=655360
$ microk8s stop
$ microk8s start
```

Or change in `/etc/sysctl.conf`, then reboot.

```sh
$ sudo vim /etc/sysctl.conf
```

```yaml
fs.inotify.max_user_instances=1280
fs.inotify.max_user_watches=655360
```

### Enable addons metallb

```sh
microk8s enable metallb:10.64.140.43-10.64.140.49


# Or
# IPADDR=$(ip -4 -j route | jq -r '.[] | select(.dst | contains("default")) | .prefsrc')
# microk8s enable metallb:$IPADDR-$IPADDR
```

check status

```sh
microk8s kubectl rollout status daemonset.apps/speaker -n metallb-system -w
```

### Deploy

```sh
$ juju add-model kubeflow
$ juju deploy kubeflow-lite --trust --debug

# Check events
$ microk8s kubectl get events -n kubeflow -w
```

```sh
# check the status of the deployment
$ watch -c juju status --color --relations
$ microk8s kubectl get all -n kubeflow
```

> :warning: you should wait for the pods to be "Running"

```sh
$ IPADDR=$(microk8s kubectl -nkubeflow get svc istio-ingressgateway-workload -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ juju config oidc-gatekeeper public-url=http://$IPADDR.nip.io
$ juju config dex-auth public-url=http://$IPADDR.nip.io
$ juju config dex-auth static-username=admin
$ juju config dex-auth static-password=admin
```

The kubeflow dashboard will available at `$IPADDR.nip.io`

```sh
sudo sshuttle -r ubuntu@$INSTANCE_IP_ADDRESS  10.1.123.0/24 --ssh-cmd "ssh -i /var/snap/multipass/common/data/multipassd/ssh-keys/id_rsa
```

> :warning: if you notice that you cannot access the dashboard, please check if
> `kubeflow-gateway` is present (run `microk8s kubectl get gateway -A` to verify).
> If the output is `No resources found.`, then you can create the gateway by
> `microk8s kubectl apply -f k8s-files/kubeflow-gateway.yaml`

To run this demo, we will also need mlflow.

```sh
$ juju deploy mlflow-server --debug
$ juju deploy charmed-osm-mariadb-k8s mlflow-db
$ juju relate minio mlflow-server
$ juju relate istio-pilot mlflow-server
$ juju relate mlflow-db mlflow-server
$ juju relate mlflow-server admission-webhook
```

You can go to the mlflow dashboard using the IP and port given by the following
commands.

```sh
$ microk8s kubectl get svc mlflow-server -nkubeflow
```


## Validate Minio Access

Validate access to Minio by looking up the values for "access-key" and "secret-key" in the
charm configuration.

```sh
$ juju config minio access-key
$ juju config minio secret-key
```
If the values are not defined, apply a new user/password with the juju commands.

```sh
$ juju config minio access-key=minio
$ kubectl get secret -n kubeflow minio-secret -o yaml | grep MINIO_SECRET_KEY | awk '{ print $2}' | base64 -d | xargs -I {} juju config minio secret-key={}

$ juju config minio secret-key
```

Test your access to Minio by accessing the dashboard. You can change the service
to be of LoadBalancer type or use port-forwarding to access it. To access the
minio dashboard, you find the IP and port using:

```sh
$ microk8s kubectl get svc minio -nkubeflow
```

The pipeline expects buckets `mlflow` and `mlpipeline` to exist in Minio. You
can create those manually in the Minio UI if it's not created for you.


### Allow the user namespace access to Minio and MLFLow

With Kubeflow 1.4, each user namespace needs to be given access to Minio and
MLFlow manually.  Apply the files `k8s-files/allow-minio.yaml` and
`k8s-files/allow-mlflow.yaml` to your user namespace.

For example, if the user is the "admin" namespace:

```sh
wget -q -O - https://raw.githubusercontent.com/canonical/kubeflow-on-microk8s-example/main/demos/kubeflow/k8s-files/allow-minio.yaml | microk8s kubectl apply -n admin -f -
wget -q -O - https://raw.githubusercontent.com/canonical/kubeflow-on-microk8s-example/main/demos/kubeflow/k8s-files/allow-mlflow.yaml | microk8s kubectl apply -n admin -f -
```

### Copying the seldon deployment secret from kubeflow to admin

See github issues: https://github.com/canonical/mlflow-operator/pull/27

```sh
$ microk8s kubectl get secret -n kubeflow mlflow-server-seldon-init-container-s3-credentials -o yaml > seldon-init-container-secret.yaml

```

Update yaml:
* RCLONE_CONFIG_S3_ENDPOINT to `echo -n http://minio.kubeflow.svc.cluster.local:9000 | base64 -w 0`
* namespace=`admin`
* metadata.name=`seldon-init-container-secret`

```yaml
apiVersion: v1
data:
  RCLONE_CONFIG_S3_ACCESS_KEY_ID: bWluaW8=
  RCLONE_CONFIG_S3_ENDPOINT: aHR0cDovL21pbmlvLmt1YmVmbG93LnN2Yy5jbHVzdGVyLmxvY2FsOjkwMDA=
  RCLONE_CONFIG_S3_ENV_AUTH: ZmFsc2U=
  RCLONE_CONFIG_S3_PROVIDER: bWluaW8=
  RCLONE_CONFIG_S3_SECRET_ACCESS_KEY: M05aSFRSQzhOVTcxSFJJVFZWTUJEVDJVTjJBMk5L
  RCLONE_CONFIG_S3_TYPE: czM=
kind: Secret
metadata:
  name: seldon-init-container-secret
  namespace: admin
```

```sh
microk8s kubectl apply -f ./seldon-init-container-secret.yaml
```
