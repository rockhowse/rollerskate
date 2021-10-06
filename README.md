# rollerskate

Sample project to test integration of a CI/CD solution using harness.io and drone for use with some of the more common integrations listed below:

* Harness.io on k8s:
  * https://docs.harness.io/article/7in9z2boh6-kubernetes-quickstart
* Prometheus, Graphana and Alertmanager on k8s:
  * https://blog.marcnuri.com/prometheus-grafana-setup-minikube
* Artifactory on k8s:
  * https://jfrog.com/screencast/jfrog-artifactory-ha-cluster-deployment-kubernetes-using-helm/
* Consul on k8s:
  * https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube?in=vault/kubernetes#install-the-consul-helm-chart
* Vault on k8s:
  * https://learn.hashicorp.com/tutorials/vault/kubernetes-minikube?in=vault/kubernetes
* Drone on k8s:
  * https://readme.drone.io/runner/kubernetes/installation/
* Jira on k8s:
  * https://community.atlassian.com/t5/Atlassian-Data-Center-on/Want-to-take-Atlasssian-Data-Center-on-Kubernetes-for-a-test/td-p/1711614

## Prerequisites

* Mac OS X
* helm v3.x
* docker
* minikube

## Step-by-step guides

1. [minikube](./000-MINIKUBE.md)
2. [harness](./001-HARNESS.md)
3. [kube-prometheus-stack](./002-MONITORING.md)
4. [artifactory](./003-ARTIFACTORY.md)
5. [consul](./004-HASHICORP.md)
6. [vault](./004-HASHICORP.md)
7. [(WIP) drone](./006-DRONE.md)
8. [(WIP) jira](./007-JIRA.md)
