# rollerskate

Sample project to test integration of a CI/CD solution including the following

* Prometheus and Graphana on k8s: https://blog.marcnuri.com/prometheus-grafana-setup-minikube
* Vault on k8s: https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes
* Artifactory on k8s: https://jfrog.com/screencast/jfrog-artifactory-ha-cluster-deployment-kubernetes-using-helm/
* Jira on k8s: https://community.atlassian.com/t5/Atlassian-Data-Center-on/Want-to-take-Atlasssian-Data-Center-on-Kubernetes-for-a-test/td-p/1711614
* Drone on k8s: https://readme.drone.io/runner/kubernetes/installation/
* Harness.io on k8s: https://docs.harness.io/article/7in9z2boh6-kubernetes-quickstart
* Concourse on k8s: https://github.com/concourse/concourse-chart

## Prerequisites

* Mac OS X
* helm v3.x
* docker
* minikube

### install minikube

```bash
brew install minikube
```

#### verify minikube version

```
❯ minikube version
minikube version: v1.23.2
commit: 0a0ad764652082477c00d51d2475284b5d39ceed
```

### start up minikube

At the time of this writing the latest EKS offering it 1.21.2 so that's what we will use.

```bash
minikube start --kubernetes-version=v1.21.2 start --cpus 4 --memory 12288 --driver=vmware
```

### Expected result

```bash
❯ minikube start --kubernetes-version=v1.21.2 start --cpus 4 --memory 12288 --driver=vmware
😄  minikube v1.23.2 on Darwin 11.6
✨  Using the vmware driver based on user configuration
💿  Downloading VM boot image ...
    > minikube-v1.23.1.iso.sha256: 65 B / 65 B [-------------] 100.00% ? p/s 0s
    > minikube-v1.23.1.iso: 225.22 MiB / 225.22 MiB  100.00% 89.31 MiB p/s 2.7s
👍  Starting control plane node minikube in cluster minikube
💾  Downloading Kubernetes v1.21.2 preload ...
    > preloaded-images-k8s-v13-v1...: 499.07 MiB / 499.07 MiB  100.00% 82.01 Mi
🔥  Creating vmware VM (CPUs=4, Memory=12288MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.21.2 on Docker 20.10.8 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

### Install Nginx ingress controller

```bash
❯ minikube addons enable ingress
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0
    ▪ Using image k8s.gcr.io/ingress-nginx/controller:v1.0.0-beta.3
    ▪ Using image k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.0
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled
```

#### Verify nginx pods are started

```bash
❯ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-n4gg9        0/1     Completed   0          3m30s
ingress-nginx-admission-patch-dm492         0/1     Completed   1          3m30s
ingress-nginx-controller-69bdbc4d57-ssq89   1/1     Running     0          3m31s
```

And getting a list of pod for the current context:

```bash
❯ minikube kubectl -- get pods -A
    > kubectl.sha256: 64 B / 64 B [--------------------------] 100.00% ? p/s 0s
    > kubectl: 50.11 MiB / 50.11 MiB [------------] 100.00% 94.48 MiB p/s 700ms
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-558bd4d5db-4ldlb           1/1     Running   0          23m
kube-system   etcd-minikube                      1/1     Running   0          23m
kube-system   kube-apiserver-minikube            1/1     Running   0          23m
kube-system   kube-controller-manager-minikube   1/1     Running   0          23m
kube-system   kube-proxy-4drx4                   1/1     Running   0          23m
kube-system   kube-scheduler-minikube            1/1     Running   0          23m
kube-system   storage-provisioner                1/1     Running   0          23m
```

Similar to docker, k8s can have multiple environments. Let's make sure we are using the minikube context:

```bash
❯ kubectl config use-context minikube

Switched to context "minikube".
```

### show cluster/system pod information

```bash
❯ kubectl cluster-info

Kubernetes master is running at https://192.168.231.2:8443
CoreDNS is running at https://192.168.231.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### start up k8s dashboard (web browser)

This opens the k8s dashboard which shows you the status of your running pods/services. This information is also avaible using the `kubectl` commands.

*NOTE* you can't CTRL+C this command or it  will stop the interface, run this command in a new console window.

```bash
❯  minikube dashboard
🔌  Enabling dashboard ...
    ▪ Using image kubernetesui/metrics-scraper:v1.0.7
    ▪ Using image kubernetesui/dashboard:v2.3.1
🤔  Verifying dashboard health ...
🚀  Launching proxy ...
🤔  Verifying proxy health ...
🎉  Opening http://127.0.0.1:50878/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
```

### install helm

```bash
brew install helm
```

#### verify helm 3.x

```bash
❯ helm version
version.BuildInfo{Version:"v3.7.0", GitCommit:"eeac83883cb4014fe60267ec6373570374ce770b", GitTreeState:"clean", GoVersion:"go1.17"}
```
