# Deploy argocd

- copy paste kubeconfig from the cp to my (linux client)
- install kubectl
- install helm (wget https://get.helm.sh/helm-v3.16.4-linux-amd64.tar.gz; tar -xzf....;    mv linux-amd64/helm /usr/local/bin/helm)

We'll use Helm to install Argo CD with the community-maintained chart from argoproj/argo-helm. The Argo project doesn't provide an official Helm chart. (create umbrella project (/charts/argo-cd))
```

helm repo add argo-cd https://argoproj.github.io/argo-helm
helm search repo argo-cd --versions   #latest version is 7.7.12


$ helm dep update charts/argo-cd/

echo "charts/**/charts" >> .gitignore

```

Disable the dex component (integration with external auth providers).
Disable the notifications controller (notify users about changes to application state).
Disable the ApplicationSet controller (automated generation of Argo CD Applications).
We start the server with the --insecure flag to serve the Web UI over HTTP. For this tutorial, we're using a local k8s server without TLS setup.


Installing the helm chart:
```
helm install argo-cd charts/argo-cd/

#uninstall
helm uninstall argo-cd --namespace default
```

Accessing the Web UI
```
kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward svc/argo-cd-argocd-server 8080:443


#if you don't want to forward to just localhost
kubectl port-forward svc/argo-cd-argocd-server 8080:443 --address 0.0.0.0
```

TODO: add Nodeport svc















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