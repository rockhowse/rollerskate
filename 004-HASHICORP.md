# install consul

Process outlined here:

https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube?in=vault/kubernetes#install-the-consul-helm-chart

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

* Name: `Hashicorp Platform`
* Deployment Type: `Kubernetes`

##### Add the Helm3 chart

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

Setup -> Hashicorp Platform -> Services -> `JFrog Platform`

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

#### Add Hashicorp Platform stage

Setup -> Hashicorp Platform -> Services -> Pipelines -> `Deploy Hashicorp Platform` -> Stage 2

* Step Name: `Deploy Consul`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Consul`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

TODO: there is an issue with k8s version detection here... Need to figure it out

### Verify consul is running successfully in your cluster

You can make use of the `kubectl` command to verify you see services running in the`hashicorp-platform` namespace:

```bash
❯ kubectl get pods -n jfrog-platform
NAME                                                              READY   STATUS    RESTARTS   AGE
r-c1826f2c-7c7b-362f-9ebc-e1d164910986-artifactory-0              1/1     Running   0          45m
r-c1826f2c-7c7b-362f-9ebc-e1d164910986-artifactory-nginx-7kgzrl   1/1     Running   0          45m
r-c1826f2c-7c7b-362f-9ebc-e1d164910986-postgresql-0               1/1     Running   0          45m
```
