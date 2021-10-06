# install artifactory + nginx

Process outlined here:

https://www.jfrog.com/confluence/display/JFROG/Installing+the+JFrog+Platform+Using+Helm+Chart

The rest of this process assumes you are using harness.io

## Add the jfrog helm server

Setup -> Connectors -> Artifact Servers -> Add

* Type: `Helm Repository`
* Display Name: `jfrog-platform`
* Hosting Platform: `HTTP Repository`
* Repository URL: `https://charts.jfrog.io`
* Username:
* Password/Token:

Enable for all environments

## Create Application, add services and update pipeline

The configuration for harness is a bit more complicated than it first appears so outlining details here.

### Add New Application

Setup -> Add Application

* Name: `JFrog Platform`

### Add Environment

Setup -> JFrog Platform -> Environments -> Add Environment

* Name: `local-minikube`

#### Add Infrastructure Definition

Setup -> JFrog Platform -> Environments -> local-minikube

* Name: `local-minikube-cluster`
* Cloud Provider Type: `Kubernetes Cluster`
* Deployment Type: `Kubernetes`
* Use already Provisioned Infrastructure -> `Minikube Cluster`
* Namespace: `jfrog-platform`
* Release Name: `r-${infra.kubernetes.infraId}`
  * NOTE: We reduce `release-` to `r-` due to length of k8s names limited to 63 characters =P

### Add Workflow

Setup -> JFrog Platform -> Workflows -> Add Workflow

* Name: `Rolling Deployment`
* Workflow Type: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `JFrog Platform`
* Infrastructure Definition: `local-minikube-cluster`

#### Templatize Workflow

In order for us to use the same `Rolling Deployment` for more than just the hard coded environment, service and infrastructure we need to templatize it.

Setup -> JFrog Platform -> Workflows -> `Rolling Deployment`

* Upper right hand corner -> Edit
* Environment -> (Click the [T] upper right hand corner) -> `${Environment}`
* Service -> (Click the [T] upper right hand corner) -> `${Service}`
* Infrastructure Definition -> (Click the [T] upper right hand corner) -> `${InfraDefinition_KUBERNETES}`

Now this should be ready to use for all our services and pipelines below =)

### Add Services

Due to some changes in Helm3, we need to do a bit of manual stuff (can probably automate this).

#### Add Create Namespace Service

Setup -> JFrog Platform -> Services -> Add Service

* Name: `Create Namespace`
* Deployment Type: `Kubernetes`

##### Update Templates to only create a namespace

Setup -> JFrog Platform -> Services -> `Create Namespace`

* Under "templates" delete `deployment.yaml` and `service.yaml`
* Trim `values.yaml` to only contain the namespace values

```yaml
name: create-namespace

createNamespace: true
namespace: ${infra.kubernetes.namespace}
```

#### Create the JFrog Platform service

This will leverage the `jfrog-platform` Helm3 chart to install

* `artifactory`
* `postgres`
* `nginx`

Setup -> JFrog Platform -> Services -> Add Service

* Name: `JFrog Platform`
* Deployment Type: `Kubernetes`

##### Add the Helm3 chart

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

Setup -> JFrog Platform -> Services -> `JFrog Platform`

Manifests -> (upper right hand corner) -> Link Remote Manifests -> Choose Files

* Manifest Format: `Helm Chart From Helm Repository`
* Helm Repository: `jfrog-platform`
* Chart Name: `jfrog-platform`
* Chart Version: `0.10.1`
* Helm Version: `v3`

In order to get the latest version, you can do the following if you have Helm3 installed locally:

```bash
❯ helm repo add jfrog https://charts.jfrog.io
"jfrog" has been added to your repositories

❯ helm repo update
Hang tight while we grab the latest from your chart repositories...
...
...Successfully got an update from the "jfrog" chart repository
...
Update Complete. ⎈Happy Helming!⎈

❯ helm search repo jfrog/jfrog-platform
NAME                	CHART VERSION	APP VERSION	DESCRIPTION
jfrog/jfrog-platform	0.10.1       	7.25.7     	The Helm chart for JFrog Platform (Universal, h...
```

##### Add Helm3 overrides

`THIS IS NOT A PRODUCTION LEVEL CONFIGURATION!`

The default configuration has a bunch of services we aren't going to use and each of them require hundreds of gigs of persistent volume claims by default. So let's trim it down a bit.

Setup -> JFrog Platform -> Services -> `JFrog Platform`

Configuration -> Values YAML Override -> (upper right hand corner) -> Edit

* Store Type: `Inline`

```yaml
postgresql:
  persistence:
    size: 2Gi
rabbitmq:
  enabled: false
redis:
  enabled: false
artifactory:
  enabled: true
  persistence:
    size: 8Gi
artifactory-ha:
  enabled: false
xray:
  enabled: false
distribution:
  enabled: false
mission-control:
  enabled: false
pipelines:
  enabled: false
```

NOTE: if you want to use `artifactory-ha` it requires an `enterprise` licence, which you don't get by default if you sign up for the trial. Ideally for production we would want to leverage the `artifactory-ha` configuration.

### Create Pipeline using services created above

Setup -> JFrog Platform -> Services -> Pipelines -> Add Pipeline

Name: `Deploy JFrog Platform`

#### Add Create Namespace stage

Setup -> JFrog Platform -> Services -> Pipelines -> `Deploy JFrog Platform` -> Stage 1

* Step Name: `Create Namespace`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `Create Namespace`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

#### Add JFrog Platform stage

Setup -> JFrog Platform -> Services -> Pipelines -> `Deploy JFrog Platform` -> Stage 2

* Step Name: `JFrog Platform`
* Execute Workflow: `Rolling Deployment`
* Environment: `local-minikube`
* Service: `JFrog Platform`
* Infrastructure Definition: `local-minikube-cluster`
* Option to Skip: `Do not skip`

You should now be able to run this pipeline with no issues.

### Verify artifactory, nginx, and postgres are running in your cluster

You can make use of the `kubectl` command to verify you see services running in the`jfrog-platform` namespace:

```bash
❯ kubectl get pods -n jfrog-platform
NAME                                                              READY   STATUS    RESTARTS   AGE
r-c1826f2c-7c7b-362f-9ebc-e1d164910986-artifactory-0              1/1     Running   0          45m
r-c1826f2c-7c7b-362f-9ebc-e1d164910986-artifactory-nginx-7kgzrl   1/1     Running   0          45m
r-c1826f2c-7c7b-362f-9ebc-e1d164910986-postgresql-0               1/1     Running   0          45m
```

## Add Ingress for artifactory

I believe this can be done using a helm3 override... but documenting the hand creation for now.

### validate nginx ingress is up and working

In order for us to install an ingress for the grafana service, we will need to verify the ingress is started:

```bash
❯ kubectl get pod -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-n4gg9        0/1     Completed   0          13h
ingress-nginx-admission-patch-dm492         0/1     Completed   1          13h
ingress-nginx-controller-69bdbc4d57-ssq89   1/1     Running     0          13h
```

### update the example ingress

Due to the naming of the services... you will need to update the `ingresses/artifactory.yml` file with your specific configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: artifactory-server
  annotations:
    ingress.kubernetes.io/force-ssl-redirect: "false"
    ingress.kubernetes.io/proxy-body-size: "0"
    ingress.kubernetes.io/proxy-read-timeout: "600"
    ingress.kubernetes.io/proxy-send-timeout: "600"
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^/(v2)/token /artifactory/api/docker/null/v2/token;
      rewrite ^/(v2)/([^\/]*)/(.*) /artifactory/api/docker/$2/$1/$3;
spec:
  rules:
    - host: <YOUR_HOST_NAME>
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <YOUR_ARTIFACTORY_NGINX_SERVICE_NAME>
                port:
                  number: 80
```

NOTE: we are explicitly disabling `TLS` and routing all traffic to `/` to this service. There might be a way to utilize a non-root baseURL or require a different hostname.

### Apply the ingress

Once you have updated your ingress, you should be able to apply it using:

```bash
kubectl apply -f ingresses/artifactory.yml -n jfrog-platform
```

### Connecting to artifactory

To connect to grafana, we are depending on a simple DNS lookup that we define locally in a similar fashion to the minikube docs.

https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/

#### get your minikube ip

```bash
❯ minikube ip
192.168.231.2
```

#### create DNS/hosts entry

You will need your minikube cluster's ip to update your `/etc/hosts` so your DNS name resolves correctly.

```bash
sudo vim /etc/hosts
```

Add the minikube ip + hostname to `/etc/hosts`

```bash
# DNS entry for testing minikube

192.168.231.2 test.rockhowse.com
```

#### Test grafana

Open a browser to:

```url
http://test.rockhowse.com/
```

And you should be re-directed to the artifactory login page:

* username: `admin`
* password: `password`

##### Reset the admin password

Once you have logged in, you will be required to change your password.

##### Enter artifactory licence

You will also be required to input the licence you recieved when signing up for an artifactory trial.
