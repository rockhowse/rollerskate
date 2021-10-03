# configure harness for k8s

This assumes you have completed the prerequisites and walks through the following install instructions:

https://docs.harness.io/article/7in9z2boh6-kubernetes-quickstart

## Install Delegate

The Delegate handles all communications between the cluster and your harness Manager (SaaS and Self Hosted).

### Generate Delegate config and download it

```bash
tar -zxvf harness-delegate-kubernetes.tar.gz

cd harness-delegate-kubernetes
```

### Modify the Delegate's resources

By default, the delegate tries to set up with 8GB of memory and 1.0 vCPU which is overkill for local testing.

Locate the "resources" block which looks something like this by default:

```yaml
        resources:
          limits:
            cpu: "1.0"
            memory: "8Gi"
```

And update it to the following:

```yaml
        resources:
          limits:
            cpu: "0.5"
            memory: "2Gi"
```

### Apply the k8s configuration for the Delegate

```bash
kubectl apply -f harness-delegate.yaml
```

### Check to see if the Delegate is available

We can use the k8s "watch" functionality to make sure the delegate comes up as expected.

```bash
❯ kubectl get pods -n harness-delegate -w
NAME                              READY   STATUS    RESTARTS   AGE
minikube-test-delegate-obbbit-0   1/1     Running   1          2d8h
```

## Continue the quickstart guide

Once you have the delegate up and running, you should be good to continue the kubernetes quickstart guide.
