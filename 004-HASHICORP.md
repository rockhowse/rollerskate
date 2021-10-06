# install consul and vault in minikube

Process outlined here:

https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube?in=vault/kubernetes

The rest of this process assumes you are using harness.io

## Add the hashicorp helm server

Setup -> Connectors -> Artifact Servers -> Add

* Type: `Helm Repository`
* Display Name: `hashicorp`
* Hosting Platform: `HTTP Repository`
* Repository URL: `https://helm.releases.hashicorp.com`
* Username:
* Password/Token:

Enable for all environments

## Create Application, add services and update pipeline

The configuration for harness is a bit more complicated than it first appears so outlining details here.

### Add New Application

Setup -> Add Application

* Name: `Hashicorp Platform`

### Add Environment

Setup -> Hashicorp Platform -> Environments -> Add Environment

* Name: `local-minikube`

#### Add Infrastructure Definition

Setup -> Hashicorp Platform -> Environments -> local-minikube

* Name: `local-minikube-cluster`
* Cloud Provider Type: `Kubernetes Cluster`
* Deployment Type: `Kubernetes`
* Use already Provisioned Infrastructure -> `Minikube Cluster`
* Namespace: `hashicorp-platform`
* Release Name: `r-${infra.kubernetes.infraId}`
  * NOTE: We reduce `release-` to `r-` due to length of k8s names limited to 63 characters =P

### Add Workflow

Setup -> Hashicorp Platform -> Workflows -> Add Workflow

* Name: `Rolling Deployment`
* Workflow Type: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Hashicorp Platform`
* Infrastructure Definition: `local-minikube-cluster`

#### Templatize Workflow

In order for us to use the same `Rolling Deployment` for more than just the hard coded environment, service and infrastructure we need to templatize it.

Setup -> Hashicorp Platform -> Workflows -> `Rolling Deployment`

* Upper right hand corner -> Edit
* Environment -> (Click the [T] upper right hand corner) -> `${Environment}`
* Service -> (Click the [T] upper right hand corner) -> `${Service}`
* Infrastructure Definition -> (Click the [T] upper right hand corner) -> `${InfraDefinition_KUBERNETES}`

Now this should be ready to use for all our services and pipelines below =)

### Add Services

Due to some changes in Helm3, we need to do a bit of manual stuff (can probably automate this).

#### Add Create Namespace Service

Setup -> Hashicorp Platform -> Services -> Add Service

* Name: `Create Namespace`
* Deployment Type: `Kubernetes`

##### Update Templates to only create a namespace

Setup -> Hashicorp Platform -> Services -> `Create Namespace`

* Under "templates" delete `deployment.yaml` and `service.yaml`
* Trim `values.yaml` to only contain the namespace values

```yaml
name: create-namespace

createNamespace: true
namespace: ${infra.kubernetes.namespace}
```

#### Create the Consul service

This will leverage the `consul` Helm3 chart to install

* `consul`

Setup -> Hashicorp Platform -> Services -> Add Service

* Name: `Consul`
* Deployment Type: `Kubernetes`

##### Add the Helm3 chart

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

Setup -> Hashicorp Platform -> Services -> `Consul`

Manifests -> (upper right hand corner) -> Link Remote Manifests -> Choose Files

* Manifest Format: `Helm Chart From Helm Repository`
* Helm Repository: `hashicorp`
* Chart Name: `consul`
* Chart Version: `0.34.1`
* Helm Version: `v3`

In order to get the latest version, you can do the following if you have Helm3 installed locally:

```bash
❯ helm repo add hashicorp https://helm.releases.hashicorp.com
"hashicorp" already exists with the same configuration, skipping

❯ helm repo update
Hang tight while we grab the latest from your chart repositories...
...
...Successfully got an update from the "hashicorp" chart repository
...
Update Complete. ⎈Happy Helming!⎈

❯ helm search repo hashicorp/consul
NAME            	CHART VERSION	APP VERSION	DESCRIPTION
hashicorp/consul	0.34.1       	1.10.2     	Official HashiCorp Consul Chart
```

##### Add Helm3 overrides

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

The default configuration creates many more resources than what we need for vault so let's do some overrides to limit the scope of the deploy.

Setup -> Hashicorp Platform -> Services -> `Consul`

Configuration -> Values YAML Override -> (upper right hand corner) -> Edit

* Store Type: `Inline`

```yaml
global:
  datacenter: vault-kubernetes-tutorial

client:
  enabled: true

server:
  replicas: 1
  bootstrapExpect: 1
  disruptionBudget:
    maxUnavailable: 0
```

### Create Pipeline using services created above

Setup -> Hashicorp Platform -> Services -> Pipelines -> Add Pipeline

Name: `Deploy Hashicorp Platform`

#### Add Create Namespace stage

Setup -> Hashicorp Platform -> Services -> Pipelines -> `Deploy Hashicorp Platform` -> Stage 1

* Step Name: `Create Namespace`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Create Namespace`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

#### Add Deploy Consul stage

Setup -> Hashicorp Platform -> Services -> Pipelines -> `Deploy Hashicorp Platform` -> Stage 2

* Step Name: `Deploy Consul`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Consul`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

TODO: there is an issue with k8s version detection here... Need to figure it out

TEMP: we are temporarily going to install via CLI

```bash
❯ helm install consul hashicorp/consul --values helm-consul-values.yml -n hashicorp-platform
NAME: consul
LAST DEPLOYED: Wed Oct  6 11:52:04 2021
NAMESPACE: hashicorp-platform
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Consul!
```

### Verify consul is running successfully in your cluster

You can make use of the `kubectl` command to verify you see services running in the`hashicorp-platform` namespace:

```bash
❯ kubectl get pod -n hashicorp-platform
NAME                     READY   STATUS    RESTARTS   AGE
consul-consul-hnv2j      1/1     Running   0          56s
consul-consul-server-0   1/1     Running   0          56s
```

#### Create the Vault service

This will leverage the `vault` Helm3 chart to install

* `vault`

Setup -> Hashicorp Platform -> Services -> Add Service

* Name: `Vault`
* Deployment Type: `Kubernetes`

##### Add the Helm3 chart

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

Setup -> Hashicorp Platform -> Services -> `Vault`

Manifests -> (upper right hand corner) -> Link Remote Manifests -> Choose Files

* Manifest Format: `Helm Chart From Helm Repository`
* Helm Repository: `hashicorp`
* Chart Name: `vault`
* Chart Version: `0.16.1`
* Helm Version: `v3`

In order to get the latest version, you can do the following if you have Helm3 installed locally:

```bash
❯ helm repo add hashicorp https://helm.releases.hashicorp.com
"hashicorp" already exists with the same configuration, skipping

❯ helm repo update
Hang tight while we grab the latest from your chart repositories...
...
...Successfully got an update from the "hashicorp" chart repository
...
Update Complete. ⎈Happy Helming!⎈

❯ helm search repo hashicorp/vault
NAME           	CHART VERSION	APP VERSION	DESCRIPTION
hashicorp/vault	0.16.1       	1.8.3      	Official HashiCorp Vault Chart
```

##### Add Helm3 overrides

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

The default configuration creates many more resources than what we need for vault so let's do some overrides to limit the scope of the deploy.

Setup -> Hashicorp Platform -> Services -> `Vault`

Configuration -> Values YAML Override -> (upper right hand corner) -> Edit

* Store Type: `Inline`

```yaml
server:
  affinity: ""
  ha:
    enabled: true
```

#### Add Deploy Consul stage to pipeline

Setup -> Hashicorp Platform -> Services -> Pipelines -> `Deploy Hashicorp Platform` -> Stage 2

* Step Name: `Deploy Vault`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Vault`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

TODO: there is an issue with rollup and Rolling Deploy... Need to figure it out

TEMP: we are temporarily going to install via CLI

```bash
❯ helm install vault hashicorp/vault --values helm-vault-values.yml -n hashicorp-platform
W1006 12:08:18.351541   93381 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
W1006 12:08:18.460034   93381 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
NAME: vault
LAST DEPLOYED: Wed Oct  6 12:08:18 2021
NAMESPACE: hashicorp-platform
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!
```

### Verify vault is running successfully in your cluster

You can make use of the `kubectl` command to verify you see services running in the`hashicorp-platform` namespace:

```bash
❯ kubectl get pod -n hashicorp-platform
NAME                                    READY   STATUS    RESTARTS   AGE
consul-consul-52259                     1/1     Running   0          54s
consul-consul-server-0                  1/1     Running   0          54s
vault-0                                 0/1     Running   0          41s
vault-1                                 0/1     Running   0          41s
vault-2                                 0/1     Running   0          41s
vault-agent-injector-77cd49cdd5-8wfbz   1/1     Running   0          41s
```

You can double check to see the unseal progress for the 3 HA Vault instances by checking `vault-0`:

```bash
❯ kubectl exec -n hashicorp-platform vault-0 -- vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.8.3
Storage Type       consul
HA Enabled         true
command terminated with exit code 2
```

##### Initialize and unseal the vault

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

Before we can make use of vault in k8s, we need to unseal it.

```bash
kubectl exec -n hashicorp-platform vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
```

Display the unseal information:

```bash
❯ cat cluster-keys.json | jq -r ".unseal_keys_b64[]"
<RANDOM_UNSEAL_INFORMATION>
```

```bash
Insecure operation: Do not run an unsealed Vault in production with a single key share and a single key threshold. This approach is only used here to simplify the unsealing process for this demonstration.
```

Create an environment variable to hold the unsealed key:

```bash
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```

Unseal vault running on the `vault-0` pod:

```bash
❯ kubectl exec -n hashicorp-platform vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.8.3
Storage Type    consul
Cluster Name    vault-cluster-806c29d7
Cluster ID      01382dbe-e15f-80b5-798a-ad63b5681257
HA Enabled      true
HA Cluster      https://vault-0.vault-internal:8201
HA Mode         active
Active Since    2021-10-06T17:41:41.291583191Z
```

```bash
Insecure operation: Providing the unseal key with the command writes the key to your shell's history. This approach is only used here to simplify the unsealing process for this demonstration.
```

Unseal vault running on the `vault-1` pod:

```bash
❯ kubectl exec -n hashicorp-platform vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY
...
```

Unseal vault running on the `vault-2` pod:

```bash
❯ kubectl exec -n hashicorp-platform vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY
...
```

##### Verify All Consul and Vault instances are running

Now that `consul` and `vault` charts have been applied and the vault ha pods have been unsealed, all pods should be running:

```bash
❯ kubectl get pod -n hashicorp-platform
NAME                                    READY   STATUS    RESTARTS   AGE
consul-consul-52259                     1/1     Running   0          31m
consul-consul-server-0                  1/1     Running   0          31m
vault-0                                 1/1     Running   0          31m
vault-1                                 1/1     Running   0          31m
vault-2                                 1/1     Running   0          31m
vault-agent-injector-77cd49cdd5-8wfbz   1/1     Running   0          31m
```

You should now be able to interact with vault using one of the pods. Fore more information look here:

https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube?in=vault/kubernetes#set-a-secret-in-vault
