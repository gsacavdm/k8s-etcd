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

POD_IP=<COPY POD IP FROM ABOVE>
etcdctl --endpoints=http://$POD_IP:4001 ls
etcdctl --endpoints=http://$POD_IP:4001 set mykey myvalue
etcdctl --endpoints=http://$POD_IP:4001 get mykey
etcdctl --endpoints=http://$POD_IP:4001 ls
```

## Option 2
Doesn't work, I suspect because of the reverse proxy results in calls via `127.0.0.1` whereas etcd isn't set up with this IP in its `advertise-client-urls`. Rather, I need to add a service properly and use that to call it instead of using the reverse proxy.

```
POD_NAME=$(k get pods -o jsonpath='{.items[0].metadata.name}')
k port-forward pod/$POD_NAME 4001

etcdctl --endpoints=http://localhost:4001 ls
```