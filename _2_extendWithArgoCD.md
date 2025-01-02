# Deploy argocd

- copy paste kubeconfig from the cp to my (linux client)
- install kubectl
- install helm (wget https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz; tar -xzf....;    mv linux-amd64/helm /usr/local/bin/helm)

We'll use Helm to install Argo CD with the community-maintained chart from argoproj/argo-helm. The Argo project doesn't provide an official Helm chart. (create umbrella project (/charts/argo-cd))
```
helm repo add argo https://argoproj.github.io/argo-helm
helm search repo argo --versions   #latest version is 7.7.12
```

















```
helm repo add argo https://argoproj.github.io/argo-helm
```

Create a values file.

```
server:
  serviceType: NodePort
  httpNodePort: 30080
  httpsNodePort: 30443
```

This file can be used to override any of the settings from the chart. In this case, I’m changing the Service to run as a NodePort vs. a ClusterIP. This will expose the specified ports from the cluster so that we can access it from our private network without using a reverse proxy.

Install the service.
```
helm install argocd -n argocd -f values.yaml argo/argocd
```

You’ll then want to grab the default password for the admin user.
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```