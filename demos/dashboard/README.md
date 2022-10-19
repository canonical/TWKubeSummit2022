
# Enable K8s dashboard

https://microk8s.io/docs/addon-dashboard

Enable Dashboard and Shared Nginx Ingress

```bash
microk8s enable dashboard ingress

```

# Port-Forward

Export the kube-dashboard in VM

```bash
sudo microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard --address 0.0.0.0 443:443
```
# Login Token

Get login token  
```
token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
kubectl -n kube-system describe secret $token
```

Open browser e.g. `https://<vm ip>.nip.io/`

Note: type `thisisunsafe` to bypass the self-signed certificate or import it into chrome browser

