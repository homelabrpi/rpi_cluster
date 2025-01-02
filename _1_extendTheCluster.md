# Adding metric server to the cluster

[metric-server](https://github.com/kubernetes-sigs/metrics-server)
```
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

#setup --kubelet-insecure-tls on the metricsserver container

k apply -f components.yaml
```