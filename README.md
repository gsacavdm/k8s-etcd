# Deploy
```
$SUBSCRIPTION_NAME=

$AKS_RG_NAME=
$AKS_NAME=

az login
az account set -s $SUBSCRIPTION_NAME
az aks get-credentials --admin -n $AKS_NAME -g $AKS_RG_NAME

kubectl config set-context --current --namespace etcd-2
kubectl apply -f etcd-2.3.8/deployment.yaml
```

# Test

## Option 1
```
# Run a temporary pod to test out etcd
kubectl run -it --rm shell --image=ubuntu:20.04 bash

# Install etcdctl
apt update && apt install -y etcd-client

POD_IP=<COPY POD IP FROM ABOVE>
etcdctl --endpoints=http://etcd.etcd-2.cluster.svc.local:4001 ls
etcdctl --endpoints=http://etcd.etcd-2.cluster.svc.local:4001 set mykey-local myvalue-local
etcdctl --endpoints=http://etcd.etcd-2.cluster.svc.local:4001 get mykey-local
etcdctl --endpoints=http://etcd.etcd-2.cluster.svc.local:4001 ls
```

## Option 2

> **NOTE**: In order for this to work, need to add `127.0.0.1:4001` to `ETCD_ADVERTISE_CLIENT_URLS` in `statefulset.yaml` as follows:
> ```
>        - name: ETCD_ADVERTISE_CLIENT_URLS
>          value: "http://$(HOST_IP):2379,http://$(HOST_IP):4001,http://etcd.etcd-2.svc.cluster.local:4001,http://127.0.0.1:4001"
> ```

```
kubectl port-forward pod/etcd-0 4001

# Install etcdctl on your host if necessary via apt install etcd-client

etcdctl --endpoints=http://localhost:4001 ls
etcdctl --endpoints=http://localhost:4001 set mykey-remote myvalue-remote
etcdctl --endpoints=http://localhost:4001 get mykey-remote
etcdctl --endpoints=http://localhost:4001 ls
```

# Scaling
At this point, we're setting up etcd using [static discovery](https://etcd.io/docs/v2.3/clustering/#static)
as observed from the `env:` section of `stateful-set.yaml`.

This means that you can't use `kubectl scale sts etcd --replicas=SOMEVALUE`.
* If `SOMEVALUE` is greater than the starting number (and the list of entries in the env var `ETCD_INITIAL_CLUSTER`, the new pods will error out indicating *couldn't find local name "etcd-SOMEVALUE" in the initial cluster configuration*. You can still recover from this by scaling back to the original number of replicas.
* If `SOMEVALUE` is less than the starting number but still enough to achieve quorum, everything will continue to work though the primary replica will endlessly show errors in the log that it can't communicate with the missing replicas. You can't recover from this by scaling back to the original number of replicas as the new ones that kubernetes creates will fail registartion with the error *member XXX has already been bootstrapped*. At this point you need to delete all the pods or the stateful set.
* If `SOMEVALUE` is less than what's needed for quorum for the starting number (and the list of entries in the env var `ETCD_INITIAL_CLUSTER`, the remaining pods cycle starting new elections and unable to get quorum. Read commands will work but writes won't. You can't recover from this by scaling back to the original number of replicas as the new ones that kubernetes creates will fail registration with the error *member XXX has already been bootstrapped*. At this point you need to delete all the pods or the stateful set.

It's entirely possible some tweaks can be done to the initial setup or something else to support scaling but that's TBD.

For now, scaling requires manually updating the `stateful-set.yaml` file with the following changes:
1. Update `spec.replicas` accordingly
1. Update `spec.template.spec.containers[0].env` with name `ETCD_INITIAL_CLUSTER` to add/remove entries to match `replicas` - 1.

Then you can apply the update as follows:
> IMPORTANT: You need to delete the existing stateful-set first otherwise, given Kubernetes' rollout policy, you mmight get conflicts between the replicas with the old config and the new replicas with the new config. TODO fix here is to generate a unique `ETCD_INITIAL_CLUSTER_TOKEN` (maybe?)
```
kubectl delete sts etcd
kubectl apply -f stateful-set.yaml
```

# TODO
* Improve this readme with an overview and more context
* Try out the discovery service
* Add etcd 3x
