# integrate artifactory into k8s

This assumes you have completed the prerequisites and walks through the following install instructions:

https://www.jfrog.com/confluence/display/JFROG/Installing+Artifactory#InstallingArtifactory-HelmInstallation

## export MASTER_KEY

```bash
export MASTER_KEY=$(openssl rand -hex 32)
echo ${MASTER_KEY}
```

## export JOIN_KEY

```bash
export JOIN_KEY=$(openssl rand -hex 32)
echo ${JOIN_KEY}
```

## create the artifactory namespace

```bash
kubectl create namespace artifactory
```

## Add the JFrog repo into helm

```bash
helm repo add jfrog https://charts.jfrog.io
```

## install the artifactory helm chart using the config above

```bash

```

## export the service on nodeport 3080

TODO: expose port to make it accessible outside cluster
