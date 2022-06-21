# crossplane-kind-eks
Example project showing how to create EKS clusters using Crossplane in kind finally run by GitHub Actions


Crossplane https://crossplane.io/ claims to be the "The cloud native control plane framework". It introduces a new way how to manage any cloud resource (beeing it Kubernetes-native or not). It's an alternative Infrastructure-as-Code tooling to Terraform, AWS CDK/Bicep or Pulumi and introduces a higher level of abstraction - based on Kubernetes CRDs.


## Crossplane basic concepts

https://crossplane.io/docs/v1.8/concepts/overview.html

* [Managed Resourced (MR)](https://crossplane.io/docs/v1.8/concepts/managed-resources.html): Kubernetes custom resources (CRDs) that represent infrastructure primitives (mostly in cloud providers). All Crossplane Managed Resources could be found via https://doc.crds.dev/ 
* [Providers](https://crossplane.io/docs/v1.8/concepts/providers.html): are Packages that bundle a set of Managed Resources & controllers to provision infrastructure resources - all providers can be found on GitHub, e.g. [provider-aws](https://github.com/crossplane-contrib/provider-aws)
* [Packages](https://crossplane.io/docs/v1.8/concepts/packages.html): OCI container images to handle distribution, version updates, dependency management & permissions for Providers & Configurations
* [Composite Resources (XR)](https://crossplane.io/docs/v1.8/concepts/composition.html): compose Managed Resources into higher level infrastructure units (especially interesting for platform teams). They are defined by a `CompositeResourceDefinition` (XRD) (incl. optional Claims (XRC)), a `Composition` and configured by a `Configuration`


### Composite Resources (XR)

https://crossplane.io/docs/v1.8/concepts/composition.html#how-it-works

> The first step towards using Composite Resources is configuring Crossplane so that it knows what XRs you’d like to exist, and what to do when someone creates one of those XRs. This is done using a `CompositeResourceDefinition` (XRD) resource and one or more `Composition` resources.

![composition-how-it-works](screenshots/composition-how-it-works.svg)

The platform team itself typically has the permissions to create XRs directly. Everyone else uses a lightweight proxy resource called `CompositeResourceClaim` (XC or simply "claim") to create them with Crossplane.

If you're familiar with Terrafrom you can think of an XRD as similar to `variable` blocks of a Terrafrom module. The `Composition` could then be seen as the rest of the HCL code describing how to instrument those variables to create actual resources.



## Getting started with Crossplane


### Fire up a K8s cluster with kind

https://crossplane.io/docs/v1.8/getting-started/install-configure.html

Be sure to have https://kind.sigs.k8s.io , the package manager Helm and kubectl installed:

```
brew install kind helm kubectl
```

Now spin up a local kind cluster

```shell
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


### Install Crossplane CLI

```
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /usr/local/bin
```

Now the `kubectl crossplane --help` command should be ready to use.


### Configure Crossplane to access AWS

https://crossplane.io/docs/v1.8/reference/configure.html

https://crossplane.io/docs/v1.8/cloud-providers/aws/aws-provider.html

https://github.com/crossplane-contrib/provider-aws