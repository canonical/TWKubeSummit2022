
# Enable Observability dashboard (v1.25 NEW!)

https://microk8s.io/docs/addons

Enable Dashboard

```bash
microk8s enable observability

```

# Port-Forward

Export the Grafana Dashboard in VM

```bash
sudo microk8s kubectl port-forward -n observability service/kube-prom-stack-grafana --address 0.0.0.0 80:80
```
# Login Password

You can get the password after deployment
```
Observability has been enabled (user/pass: admin/prom-operator) 
```

Open browser e.g. `http://<vm ip>/`

Note: type `thisisunsafe` to bypass the self-signed certificate or import it into chrome browser

