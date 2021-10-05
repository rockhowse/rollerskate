# configure drone for k8s

(WORK IN PROGRES)

This assumes you have completed the prerequisites and walks through the following install instructions:

https://github.com/drone/charts/blob/master/charts/drone/docs/install.md

## add done repo for helm

```bash
helm repo add drone https://charts.drone.io
helm repo update
```

## create drone namespace

```bash
kubectl create ns drone
```

## install drone helm chart

```bash
helm install --namespace drone drone drone/drone -f drone-values.yml
```

## authorize using OATH callback

```bash
  export POD_NAME=$(kubectl get pods --namespace drone -l "app.kubernetes.io/name=drone,app.kubernetes.io/instance=drone" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace drone port-forward $POD_NAME 8080:80
```

TODO: find IP/hostname and re-do OATH configuration... needs internet ingress.
