
## Complete DevOps Project
#### Go Application 


Aiming to save this entire project in the below structure.

### Manual Installation

### Cert manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
```
### Install Kube prometheus stack
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace

```
### Getting Grafana login secret for admin user

```
kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl port-forward svc/grafana 3000:80 -n monitoring
```

### CI - Jenkins
