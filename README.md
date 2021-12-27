# Deploy
```
$SUBSCRIPTION_NAME=

$AKS_RG_NAME=
$AKS_NAME=

az login
az account set -s $SUBSCRIPTION_NAME
az aks get-credentials --admin -n $AKS_NAME -g $AKS_RG_NAME

k config set-context --current --namespace etcd-2
k apply -f etcd-2.3.8/deployment.yaml
```

# Test

## Option 1
```
# Get the IP of the pod
k get pods -o jsonpath='{.items[0].status.podIP}'

# Run a temporary pod to test out etcd
k run -it --rm shell --image=ubuntu:20.04 bash

# Install etcdctl
apt update && apt install -y etcd-client

POD_IP=<COPY POD IP FROM ABOVE>
etcdctl --endpoints=http://$POD_IP:4001 ls
etcdctl --endpoints=http://$POD_IP:4001 set mykey-local myvalue-local
etcdctl --endpoints=http://$POD_IP:4001 get mykey-local
etcdctl --endpoints=http://$POD_IP:4001 ls
```

## Option 2

> **NOTE**: In order for this to work, need to add `127.0.0.1:4001` to `ETCD_ADVERTISE_CLIENT_URLS` in `deployment.yaml` as follows:
> ```
>        - name: ETCD_ADVERTISE_CLIENT_URLS
>          value: "http://$(HOST_IP):2379,http://$(HOST_IP):4001,http://127.0.0.1:4001"
> ```

```
POD_NAME=$(k get pods -o jsonpath='{.items[0].metadata.name}')
k port-forward pod/$POD_NAME 4001

etcdctl --endpoints=http://localhost:4001 ls
etcdctl --endpoints=http://localhost:4001 set mykey-remote myvalue-remote
etcdctl --endpoints=http://localhost:4001 get mykey-remote
etcdctl --endpoints=http://localhost:4001 ls

```

# Scaling
At this point, scaling the deployment will result in more **INDEPENDENT** etcd instances, not a clustered etcd deployment.
For that the config needs to be changed as described in etcd's [Clustering Guide](https://etcd.io/docs/v2.3/clustering/#static) but we'll need to use stateful sets and a service so that we have declarative URLs that we can use for each replica. That'll come in the next PR.