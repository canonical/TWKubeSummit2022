# E2E pipeline

> This example is based on [canonical/kubeflow-example/e2e-wine-kfp-mlflow](https://github.com/canonical/kubeflow-examples/tree/main/e2e-wine-kfp-mlflow)
> with some modifications to simplify the codebase. This example is explained
> in more details in this [blog post](https://ubuntu.com/blog/mlops-pipeline-with-mlflow-seldon-core-and-kubeflow-pipelines).


## Deploy kubeflow + mlflow

```sh
$ juju add-model kubeflow
$ juju deploy kubeflow-lite --trust --debug

# workaround for upstream issues in kubeflow:
# https://github.com/kubeflow/manifests/issues/2087#issuecomment-1139260511
$ sudo sysctl fs.inotify.max_user_instances=1280
$ sudo sysctl fs.inotify.max_user_watches=655360
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

> :warning: if you notice that you cannot access the dashboard, please check if
> `kubeflow-gateway` is present (run `microk8s kubectl get gateway -A` to verify).
> If the output is `No resources found.`, then you can create the gateway by
> `microk8s kubectl apply -f k8s-files/kubeflow-gateway.yaml`

To run this demo, we will also need mlflow.

```sh
$ juju deploy mlflow-server
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
$ juju config minio secret-key=minio123
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
$ microk8s kubectl apply -f k8s-files/allow-minio.yaml -n admin
$ microk8s kubectl apply -f k8s-files/allow-mlflow.yaml -n admin
```

### Copying the seldon deployment secret from kubeflow to admin

See github issues: https://github.com/canonical/mlflow-operator/pull/27

```sh
$ microk8s kubectl get secret localdockerreg --namespace=kubeflow -oyaml | grep -v '^\s*namespace:\s' | kubectl apply --namespace=admin -f -
```

### Running the demo

You should first navigate to the kubeflow dashboard at `$IPADDR.nip.io` and
create a jupyter notebook server under the "Notebook" tab in the sidebar. Then,
you can choose an appropriate cpus and memory for your server; the
default value is good enough for this demo. Also, check

- `Allow access to Minio`
- `Allow access to MLFlow`

so that the necessary environment variables will be imported to the notebook
server.

![Create notebook server - 1](./assets/create_server_1.png)
![Create notebook server - 2](./assets/create_server_2.png)

You can either choose to run the demo with the python script by first installing
the required package `pip install -r requirement.txt`, and then run `python3
pipeline.py` to build, train, and deploy the model. Or you can follow the
notebook `e2e-kfp-mlflow-seldon-pipeline.ipynb` that basically do the same thing
as the python script.

After running the script or notebook, you can view the result in the "Runs" tab
and view the model in mlflow dashboard. For more information about demo, you can
visit the [blog
post](https://ubuntu.com/blog/mlops-pipeline-with-mlflow-seldon-core-and-kubeflow-pipelines).

Lastly, you can try to make inference of new data using the script `sample-prediction.sh`.


### Reference

* [GitHub - canonical/bundle-kubeflow: Charmed Kubeflow](https://github.com/canonical/bundle-kubeflow)
* [Kubeflow + MLflow on Juju with Microk8s](https://github.com/canonical/mlflow-operator)
* [GitHub - canonical/bundle-kubeflow: Charmed Kubeflow](https://github.com/canonical/bundle-kubeflow)
* [Quick start guide to Kubeflow | Documentation | Charmed Kubeflow](https://charmed-kubeflow.io/docs/quickstart)

