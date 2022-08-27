# Cluster api with digitalocean and microk8s

This repository is a demonstration on using clusterapi to bootstrap a Kubernetes 

Follow the digitalocean documentation on setting up the environment for [clusterapi](https://github.com/kubernetes-sigs/cluster-api-provider-digitalocean/blob/main/docs/getting-started.md#setup-environment)

## Install `clusterctl`

## Setup Bootstrap cluster


``` shell
sudo snap install microk8s --channel=1.25/stable
sudo microk8s.config  > ~/.kube/config
sudo microk8s enable dns rbac
```

## Setup clusterctl configuration

On your `~/.cluster-api/`, create the `clusterctl.yaml` file.  For now point the content to this location `control-plane-microk8s/v0.1.0/control-plane-components.yaml` and `boostrap-microk8s/v0.1.0/bootstrap-components.yaml`

Use the [bootstrap yamls](bootstrap-microk8s/) and [control-plane yamls](control-plane-microk8s/)

Example:

``` yaml
providers:
  - name: "microk8s"
    url: "/home/thor/workspace/domk8s-cluster-api/bootstrap-microk8s/v0.1.0/bootstrap-components.yaml"
    type: "BootstrapProvider"
  - name: "microk8s"
    url: "/home/thor/workspace/domk8s-cluster-api/control-plane-microk8s/v0.1.0/control-plane-components.yaml"
    type: "ControlPlaneProvider"
```

The clusterapi enforces strict SemVersioning of the components.  The local path must follow the component's version defined in the [`metadata.yaml`](bootstrap-microk8s/v0.1.0/metadata.yaml)


## Install digitalocean infrastructure provider

```
$ export DO_B64ENCODED_CREDENTIALS="$(echo -n "${DIGITALOCEAN_ACCESS_TOKEN}" | base64 | tr -d '\n')"

# Initialize a management cluster with digitalocean infrastructure provider and microk8s boostrap and controlplane provider instead of kubeadm
$ clusterctl init --infrastructure digitalocean --bootstrap microk8s --control-plane microk8s
```

It should show you something like this

``` shell
kubectl get po -A -o wide
NAMESPACE      NAME                                                            READY   STATUS    RESTARTS      AGE   IP             NODE    NOMINATED NODE   READINESS GATES
kube-system    calico-kube-controllers-884556756-rtwvp                         1/1     Running   1 (29m ago)   52m   10.1.205.12    norse   <none>           <none>
cert-manager   cert-manager-7778d64785-4h7wt                                   1/1     Running   1 (29m ago)   45m   10.1.205.13    norse   <none>           <none>
cert-manager   cert-manager-cainjector-5c7b85f464-t6h27                        1/1     Running   1 (29m ago)   45m   10.1.205.14    norse   <none>           <none>
kube-system    calico-node-skd2x                                               1/1     Running   1 (29m ago)   52m   192.168.2.24   norse   <none>           <none>
cert-manager   cert-manager-webhook-58b97ccf69-tmfs2                           1/1     Running   1 (29m ago)   45m   10.1.205.10    norse   <none>           <none>
kube-system    coredns-d489fb88-dthsr                                          1/1     Running   1 (29m ago)   50m   10.1.205.11    norse   <none>           <none>
capi-system    capi-controller-manager-67c759dd56-64cjv                        1/1     Running   0             44s   10.1.205.19    norse   <none>           <none>
capdo-system   capdo-controller-manager-7d69c9f848-qwbpr                       2/2     Running   0             42s   10.1.205.22    norse   <none>           <none>
capm-system    capi-bootstrap-microk8s-controller-manager-7cfbc67f95-bc5ll     2/2     Running   0             44s   10.1.205.20    norse   <none>           <none>
capm-system    capi-cp-provider-microk8s-controller-manager-7d57d5b7b7-jbgmd   2/2     Running   0             43s   10.1.205.21    norse   <none>           <none>
```

## Setup the Kubernetes cluster

``` shell
kubectl create ns do-mk8s
kubectl apply -f do-cluster-api.yaml
```

The entire process takes a few minutes to complete.


### Show the cluster

``` shell

$ kubectl -n do-mk8s get cluster
NAME      PHASE         AGE    VERSION
do-mk8s   Provisioned   5m5s   

$ kubectl -n do-mk8s get machine

NAME                          CLUSTER   NODENAME                      PROVIDERID                 PHASE     AGE   VERSION
do-mk8s-control-plane-c7v4c   do-mk8s   do-mk8s-control-plane-4qrsf   digitalocean://314245169   Running   13m   v1.23.0
do-mk8s-md-59bfdcfb6b-67kkw   do-mk8s   do-mk8s-worker-md-xdsv2       digitalocean://314245618   Running   15m   v1.23.0
do-mk8s-control-plane-nrkb4   do-mk8s   do-mk8s-control-plane-r5sfh   digitalocean://314245620   Running   13m   v1.23.0
do-mk8s-control-plane-bgvk6   do-mk8s   do-mk8s-control-plane-ht69b   digitalocean://314245622   Running   13m   v1.23.0
do-mk8s-md-59bfdcfb6b-sq45m   do-mk8s   do-mk8s-worker-md-lzscj       digitalocean://314245616   Running   15m   v1.23.0


$ kubectl -n do-mk8s get domachine
NAME                          CLUSTER   STATE    READY   INSTANCEID                 MACHINE
do-mk8s-control-plane-4qrsf   do-mk8s   active   true    digitalocean://314245169   do-mk8s-control-plane-c7v4c
do-mk8s-worker-md-xdsv2       do-mk8s   active   true    digitalocean://314245618   do-mk8s-md-59bfdcfb6b-67kkw
do-mk8s-worker-md-lzscj       do-mk8s   active   true    digitalocean://314245616   do-mk8s-md-59bfdcfb6b-sq45m
do-mk8s-control-plane-r5sfh   do-mk8s   active   true    digitalocean://314245620   do-mk8s-control-plane-nrkb4
do-mk8s-control-plane-ht69b   do-mk8s   active   true    digitalocean://314245622   do-mk8s-control-plane-bgvk6
thor@norse:~/workspace/domk8s-cluster-api$ 

# show microk8s controller
$ kubectl -n do-mk8s get microk8scontrolplanes.controlplane
NAME                    READY   INITIALIZED   REPLICAS   READY REPLICAS   UNAVAILABLE REPLICAS
do-mk8s-control-plane   true    true          3          3                

```
### Get the kubeconfig from the cluster

``` shell
$ clusterctl -n do-mk8s get kubeconfig do-mk8s > /tmp/kubeconfig.yaml 

$ export KUBECONFIG=/tmp/kubeconfig.yaml

$ kubectl get no -o wide
NAME                          STATUS   ROLES    AGE   VERSION                    INTERNAL-IP       EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION      CONTAINER-RUNTIME
do-mk8s-control-plane-r5sfh   Ready    <none>   36m   v1.23.9-2+88a2c6a14e7008   157.245.145.186   <none>        Ubuntu 22.04 LTS   5.15.0-41-generic   containerd://1.5.13
do-mk8s-control-plane-ht69b   Ready    <none>   35m   v1.23.9-2+88a2c6a14e7008   157.245.145.188   <none>        Ubuntu 22.04 LTS   5.15.0-41-generic   containerd://1.5.13
do-mk8s-worker-md-lzscj       Ready    <none>   35m   v1.23.9-2+88a2c6a14e7008   68.183.228.5      <none>        Ubuntu 22.04 LTS   5.15.0-41-generic   containerd://1.5.13
do-mk8s-worker-md-xdsv2       Ready    <none>   36m   v1.23.9-2+88a2c6a14e7008   157.245.150.8     <none>        Ubuntu 22.04 LTS   5.15.0-41-generic   containerd://1.5.13
do-mk8s-control-plane-4qrsf   Ready    <none>   43m   v1.23.9-2+88a2c6a14e7008   157.245.154.226   <none>        Ubuntu 22.04 LTS   5.15.0-41-generic   containerd://1.5.13

$ kubectl get pods -A

NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   calico-node-kgpq7                         1/1     Running   0          5m34s
kube-system   calico-node-pjg7l                         1/1     Running   0          5m34s
kube-system   calico-node-mvf9v                         1/1     Running   0          5m34s
kube-system   calico-node-mw5x9                         1/1     Running   0          5m33s
ingress       nginx-ingress-microk8s-controller-xmzmk   1/1     Running   0          5m24s
kube-system   calico-node-hvjjx                         1/1     Running   0          5m18s
kube-system   calico-kube-controllers-5fdd565b5-p6kbn   1/1     Running   0          12m
ingress       nginx-ingress-microk8s-controller-pxjsx   1/1     Running   0          5m26s
ingress       nginx-ingress-microk8s-controller-p9c64   1/1     Running   0          5m34s
ingress       nginx-ingress-microk8s-controller-9ps6m   1/1     Running   0          5m22s
kube-system   coredns-64c6478b6c-mtlvw                  1/1     Running   0          9m43s
ingress       nginx-ingress-microk8s-controller-nkpwl   1/1     Running   0          9m43s

```

### Teardown

``` shell
kubectl -n do-mk8s delete cluster do-mk8s

kubectl delete ns do-mk8s
```