# install prometheus, graphana, and alertmanager

Process outlined here:

https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

The rest of this process assumes you are using harness.io

## Add the prometheus-community helm server

Setup -> Connectors -> Artifact Servers -> Add

* Type: `Helm Repository`
* Display Name: `prometheus-community`
* Hosting Platform: `HTTP Repository`
* Repository URL: `https://prometheus-community.github.io/helm-charts`
* Username:
* Password/Token:

Enable for all environments

## Create Application, add services and update pipeline

The configuration for harness is a bit more complicated than it first appears so outlining details here.

### Add New Application

Setup -> Add Application

* Name: `Kube Prometheus Stack`

### Add Environment

Setup -> Kube Prometheus Stack -> Environments -> Add Environment

* Name: `local-minikube`

#### Add Infrastructure Definition

Setup -> Kube Prometheus Stack -> Environments -> local-minikube

* Name: `local-minikube-cluster`
* Cloud Provider Type: `Kubernetes Cluster`
* Deployment Type: `Kubernetes`
* Use already Provisioned Infrastructure -> `Minikube Cluster`
* Namespace: `monitoring`
* Release Name: `release-${infra.kubernetes.infraId}`

### Add Workflow

Setup -> Kube Prometheus Stack -> Workflows -> Add Workflow

* Name: `Rolling Deployment`
* Workflow Type: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Kube Prometheus Stack`
* Infrastructure Definition: `local-minikube-cluster`

#### Templatize Workflow

In order for us to use the same `Rolling Deployment` for more than just the hard coded environment, service and infrastructure we need to templatize it.

Setup -> Kube Prometheus Stack -> Workflows -> `Rolling Deployment`

* Upper right hand corner -> Edit
* Environment -> (Click the [T] upper right hand corner) -> `${Environment}`
* Service -> (Click the [T] upper right hand corner) -> `${Service}`
* Infrastructure Definition -> (Click the [T] upper right hand corner) -> `${InfraDefinition_KUBERNETES}`

Now this should be ready to use for all our services and pipelines below =)

### Add Services

Due to some changes in Helm3, we need to do a bit of manual stuff (can probably automate this).

#### Add Create Namespace Service

Setup -> Kube Prometheus Stack -> Services -> Add Service

* Name: `Create Namespace`
* Deployment Type: `Kubernetes`

##### Update Templates to only create a namespace

Setup -> Kube Prometheus Stack -> Services -> `Create Namespace`

* Under "templates" delete `deployment.yaml` and `service.yaml`
* Trim `values.yaml` to only contain the namespace values

```yaml
name: create-namespace

createNamespace: true
namespace: ${infra.kubernetes.namespace}
```

#### Create the Install CRDs Service

This service is needed to create the Custom Resource Definitions (CRDs) used by the helm chart.

https://helm.sh/docs/topics/charts/#limitations-on-crds

Setup -> Kube Prometheus Stack -> Services -> Add Service

* Name: `Install CRDs`
* Deployment `Type: Kubernetes`

##### Manually add the CRDs from the helm chart

Setup -> Kube Prometheus Stack -> Services -> `Install CRDs`

While nasty and probably not the way we `should` do this... we have limited time and it gets the job done.

* Delete `deployment.yaml`, `namespace.yaml`, and `service.yaml`
* Delete the `templates` folder
* Update the `values.yaml` to the following

```yaml
name: install-crds
```

* Clone the `prometheus-community` repo locally

```bash
docker clone git@github.com:prometheus-community/helm-charts.git
```

Manifests -> (upper right hand corner) -> Upload Inline Manifests -> Choose Files

* Upload all files from the `helm-charts/charts/kube-prometheus-stack/crds` directory into a directory named `crds`

You should now be good to have harness create the CRDs for you. Required before applying the Helm chart.

#### Create the Kube Prometheus Stack service

This will leverage the `kube-prometheus-stack` Helm3 chart to install

* `prometheus`
* `grafana`
* `alertmanager`

Setup -> Kube Prometheus Stack -> Services -> Add Service

* Name: Kube Prometheus Stack
* Deployment Type: Kubernetes

##### Add the Helm3 chart

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

Setup -> Kube Prometheus Stack -> Services -> `Kube Prometheus Stack`

Manifests -> (upper right hand corner) -> Link Remote Manifests -> Choose Files

* Manifest Format: `Helm Chart From Helm Repository`
* Helm Repository: `prometheus-community`
* Chart Name: `kube-prometheus-stack`
* Chart Version: `19.0.1`
* Helm Version: `v3`

In order to get the latest version, you can do the following if you have Helm3 installed locally:

```bash
❯ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" already exists with the same configuration, skipping

❯ helm repo update
Hang tight while we grab the latest from your chart repositories...
...
...Successfully got an update from the "prometheus-community" chart repository
...
Update Complete. ⎈Happy Helming!⎈

❯ helm search repo prometheus-community
NAME                                              	CHART VERSION	APP VERSION	DESCRIPTION
prometheus-community/alertmanager                 	0.12.2       	v0.22.1    	The Alertmanager handles alerts sent by client ...
prometheus-community/kube-prometheus-stack        	19.0.1       	0.50.0     	kube-prometheus-stack collects Kubernetes manif...
```

##### Add Helm3 overrides

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

The default configuration has a couple things that fail out of the gate. Further investigation as to why needs to occur, but for now this allowed the services to come up without errors.

Setup -> Kube Prometheus Stack -> Services -> `Kube Prometheus Stack`

Configuration -> Values YAML Override -> (upper right hand corner) -> Edit

* Store Type: `Inline`

```yaml
prometheusOperator:
  admissionWebhooks:
    enabled: false
  patch:
    enabled: false
  tls:
    enabled: false
  tlsProxy:
    enabled: false

grafana:
  grafana.ini:
    server:
      domain: test.rockhowse.com
      root_url: http://test.rockhowse.com/grafana
      serve_from_sub_path: true
```

Given that this disabled the admission webhooks and tls it `IS NOT` viable for a production system. This is for local testing only.

### Create Pipeline using services created above

Setup -> Kube Prometheus Stack -> Services -> Pipelines -> Add Pipeline

Name: `Deploy Kube Prometheus Stack`

#### Add Create Namespace stage

Setup -> Kube Prometheus Stack -> Services -> Pipelines -> `Deploy Kube Prometheus Stack` -> Stage 1

* Step Name: `Create Namespace`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Create Namespace`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

#### Add Install CRDs stage

Setup -> Kube Prometheus Stack -> Services -> Pipelines -> `Deploy Kube Prometheus Stack` -> Stage 2

* Step Name: `Install CRDs`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Install CRDs`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

#### Add Install CRDs stage

Setup -> Kube Prometheus Stack -> Services -> Pipelines -> `Deploy Kube Prometheus Stack` -> Stage 3

* Step Name: `Kube Prometheus Stack`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Kube Prometheus Stack`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

You should now be able to run this pipeline with no issues.

### Verify prometheus, graphana and alertmanager are running in your cluster

You can make use of the `kubectl` command to verify you see the `kube-prometheus-stack` running in the `monitoring` namespace:

```bash
❯ kubectl get pods -n monitoring
NAME                                                              READY   STATUS      RESTARTS   AGE
alertmanager-release-0868173b-25eb-3ecf-alertmanager-0            2/2     Running     0          45h
prometheus-release-0868173b-25eb-3ecf-prometheus-0                2/2     Running     0          45h
release-0868173b-25eb-3ecf-8da9-3191976d02a0-grafana-6947dbbt72   2/2     Running     0          11h
release-0868173b-25eb-3ecf-8da9-3191976d02a0-grafana-test         0/1     Completed   0          11h
release-0868173b-25eb-3ecf-8da9-3191976d02a0-kube-state-me64x2f   1/1     Running     0          45h
release-0868173b-25eb-3ecf-8da9-3191976d02a0-prometheus-nop9824   1/1     Running     0          45h
release-0868173b-25eb-3ecf-operator-55b966b9b7-pkhn5              1/1     Running     0          45h
```
