# configure vault for k8s

This assumes you have completed the prerequisites and walks through the following install instructions:

https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes

### create vault namespace

```bash
kubectl create namespace vault
```

### add vault repo chart to helms

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
```

### install vault with ha config

```bash
helm install vault hashicorp/vault \
    --namespace vault \
    --set "server.ha.enabled=true" \
    --set "server.ha.replicas=3"
```

Output:

```
W0928 20:29:38.846055   11940 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
W0928 20:29:38.937063   11940 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
NAME: vault
LAST DEPLOYED: Tue Sep 28 20:29:38 2021
NAMESPACE: vault
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
```

#### Deprication Warning

Note that `PodDistruptionBudget` is moving from `v1beta1` to `v1` api with the versions used.

### initialize and unseal vault

#### verify vault containers are not running

```bash
❯ kubectl get pods --selector='app.kubernetes.io/name=vault' --namespace='vault'
NAME      READY   STATUS    RESTARTS   AGE
vault-0   0/1     Running   0          6m45s
vault-1   0/1     Pending   0          6m44s
vault-2   0/1     Pending   0          6m43s
```

### expose the gui

```
❯ kubectl port-forward vault-0 8200:8200 -n vault
Forwarding from 127.0.0.1:8200 -> 8200
Forwarding from [::1]:8200 -> 8200
```

TODO: at this point things went off the rails... will need to revisit
