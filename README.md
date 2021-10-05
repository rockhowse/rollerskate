# rollerskate

Sample project to test integration of a CI/CD solution using harness.io and drone for use with some of the more common integrations listed below:

* Harness.io on k8s:
  * https://docs.harness.io/article/7in9z2boh6-kubernetes-quickstart
* Prometheus, Graphana and Alertmanager on k8s:
  * https://blog.marcnuri.com/prometheus-grafana-setup-minikube
* Vault on k8s:
  * https://learn.hashicorp.com/tutorials/vault/kubernetes-raft-deployment-guide?in=vault/kubernetes
* Artifactory on k8s:
  * https://jfrog.com/screencast/jfrog-artifactory-ha-cluster-deployment-kubernetes-using-helm/
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
4. [(WIP) vault](./003-VAULT.md)
5. [(WIP) artifactory](./004-ARTIFACTORY.md)
6. [(WIP) drone](./005-DRONE.md)
7. [(WIP) jira](./005-JIRA.md)
