# integrate jira into k8s

(WORK IN PROGRES)

This assumes you have completed the prerequisites and walks through the following install instructions:

https://atlassian.github.io/data-center-helm-charts/userguide/INSTALLATION/

## provision an nginx ingress controller

https://atlassian.github.io/data-center-helm-charts/examples/ingress/INGRESS_NGINX/

### add ngnix chart to helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

### update helm

```bash
helm repo update
```

### create ingress namespace

```bash
kubectl create namespace ingress
```

### install the nginx ingress

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress
```

### watch the ingress until you see 1/1 pods

```bash
>watch -n1 "kubectl get pods --namespace ingress"

Every 1.0s: kubectl get pods --namespace ingress
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-748d8ff6c7-9x5mr   1/1     Running   0          98s
```

### get the ingress configuration

```bash
❯ kubectl get service -n ingress | grep ingress-nginx
ingress-nginx-controller             LoadBalancer   10.109.8.132   <pending>     80:30639/TCP,443:30981/TCP   2m34s
ingress-nginx-controller-admission   ClusterIP      10.98.22.132   <none>        443/TCP                      2m34s
```

TODO: get a DNS or other entry routed through

## create the atlassian namespace

``bash
kubectl create namespace atlassian
```

## Add the atlassian repo into helm

```bash
helm repo add atlassian-data-center \
 https://atlassian.github.io/data-center-helm-charts
```

## install the artifactory helm chart using the config above

```bash
helm install atlassian-jira \
             atlassian-data-center/jira \
             --namespace atlassian
```

## validate jira pod is initialized

```bash
>watch -n1 "kubectl get pods -n atlassian"

Every 1.0s: kubectl get pods -n atlassian
NAME               READY   STATUS    RESTARTS   AGE
atlassian-jira-0   1/1     Running   0          2m48s
```

## port forward 8080 from the jira pod

```bash
❯ kubectl --namespace atlassian port-forward atlassian-jira-0 8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

## ran into error getting jira license via HTTPS

TODO: get license exposed via public TLS
