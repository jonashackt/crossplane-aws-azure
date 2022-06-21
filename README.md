# crossplane-kind-eks
Example project showing how to create EKS clusters using Crossplane in kind finally run by GitHub Actions


### Fire up a K8s cluster with kind

https://crossplane.io/docs/v1.8/getting-started/install-configure.html

Be sure to have https://kind.sigs.k8s.io , the package manager Helm and kubectl installed:

```
brew install kind helm kubectl
```

Now spin up a local kind cluster

```
kind create cluster --image kindest/node:v1.23.0 --wait 5m
```


### Install Crossplane with Helm

https://crossplane.io/docs/v1.8/getting-started/install-configure.html#install-crossplane

```shell
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm upgrade --install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

Check crossplane status with `helm list -n crossplane-system` and `kubectl get all -n crossplane-system`:

```shell
$ helm list -n crossplane-system
NAME      	NAMESPACE        	REVISION	UPDATED                              	STATUS  	CHART           	APP VERSION
crossplane	crossplane-system	1       	2022-06-21 09:28:21.178357 +0200 CEST	deployed	crossplane-1.8.1	1.8.1

$ kubectl get all -n crossplane-system
NAME                                           READY   STATUS    RESTARTS   AGE
pod/crossplane-7c88c45998-d26wl                1/1     Running   0          69s
pod/crossplane-rbac-manager-8466dfb7fc-db9rb   1/1     Running   0          69s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/crossplane                1/1     1            1           69s
deployment.apps/crossplane-rbac-manager   1/1     1            1           69s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/crossplane-7c88c45998                1         1         1       69s
replicaset.apps/crossplane-rbac-manager-8466dfb7fc   1         1         1       69s
```