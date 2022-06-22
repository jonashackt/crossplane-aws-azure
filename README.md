# crossplane-kind-eks
[![Build Status](https://github.com/jonashackt/crossplane-kind-eks/workflows/provision/badge.svg)](https://github.com/jonashackt/crossplane-kind-eks/actions)
[![License](http://img.shields.io/:license-mit-blue.svg)](https://github.com/jonashackt/crossplane-kind-eks/blob/master/LICENSE)
[![renovateenabled](https://img.shields.io/badge/renovate-enabled-yellow)](https://renovatebot.com)

Example project showing how to create EKS clusters using crossplane in kind finally run by GitHub Actions


Crossplane https://crossplane.io/ claims to be the "The cloud native control plane framework". It introduces a new way how to manage any cloud resource (beeing it Kubernetes-native or not). It's an alternative Infrastructure-as-Code tooling to Terraform, AWS CDK/Bicep or Pulumi and introduces a higher level of abstraction - based on Kubernetes CRDs.


## crossplane basic concepts

https://crossplane.io/docs/v1.8/concepts/overview.html

* [Managed Resourced (MR)](https://crossplane.io/docs/v1.8/concepts/managed-resources.html): Kubernetes custom resources (CRDs) that represent infrastructure primitives (mostly in cloud providers). All crossplane Managed Resources could be found via https://doc.crds.dev/ 
* [Composite Resources (XR)](https://crossplane.io/docs/v1.8/concepts/composition.html): compose Managed Resources into higher level infrastructure units (especially interesting for platform teams). They are defined by:
    * a `CompositeResourceDefinition` (XRD)
    * and an optional `CompositeResourceClaims` (XRC)
    * a `Composition`
    * and configured by a `Configuration`
* [Packages](https://crossplane.io/docs/v1.8/concepts/packages.html): OCI container images to handle distribution, version updates, dependency management & permissions for Providers & Configurations
    * [Providers](https://crossplane.io/docs/v1.8/concepts/providers.html): are Packages that bundle a set of Managed Resources & controllers to provision infrastructure resources - all providers can be found on GitHub, e.g. [provider-aws](https://github.com/crossplane-contrib/provider-aws) or on [docs.crds.dev](https://doc.crds.dev/github.com/crossplane/provider-aws). A [list of all available Providers](https://github.com/orgs/crossplane-contrib/repositories?type=all) can also be found on GitHub.
    * [Configuration](https://crossplane.io/docs/v1.8/getting-started/create-configuration.html): define your own Composite Resources (XRs) & package them via `kubectl crossplane build configuration` (now they are a Package) - and push them to an OCI registry via `kubectl crossplane push configuration`. With this Configurations can also be easily installed into other crossplane clusters.



### Composite Resources (XR)

https://crossplane.io/docs/v1.8/concepts/composition.html#how-it-works

> The first step towards using Composite Resources is configuring crossplane so that it knows what XRs you’d like to exist, and what to do when someone creates one of those XRs. This is done using a `CompositeResourceDefinition` (XRD) resource and one or more `Composition` resources.

![composition-how-it-works](screenshots/composition-how-it-works.svg)

The platform team itself typically has the permissions to create XRs directly. Everyone else uses a lightweight proxy resource called `CompositeResourceClaim` (XC or simply "claim") to create them with crossplane.

If you're familiar with Terrafrom you can think of an XRD as similar to `variable` blocks of a Terrafrom module. The `Composition` could then be seen as the rest of the HCL code describing how to instrument those variables to create actual resources.



## Getting started with crossplane

In order to use crossplane we'll need any kind of Kubernetes cluster to let it operate in. This management cluster with crossplane installed will then provision the defined infrastructure. Using any managed Kubernetes cluster like EKS, AKS and so on is possible - or even a local [Minikube](https://minikube.sigs.k8s.io/docs/start/), [kind](https://kind.sigs.k8s.io) or [k3d](https://k3d.io/).


### Fire up a K8s cluster with kind

https://crossplane.io/docs/v1.8/getting-started/install-configure.html

Be sure to have kind, the package manager Helm and kubectl installed:

```shell
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


#### Create aws-creds.conf file

https://crossplane.io/docs/v1.8/getting-started/install-configure.html#get-aws-account-keyfile

I assume here that you have [aws CLI installed and configured](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). So that the command `aws configure` should work on your system. With this prepared we can create an `aws-creds.conf` file:

```shell
echo "[default]
aws_access_key_id = $(aws configure get aws_access_key_id)
aws_secret_access_key = $(aws configure get aws_secret_access_key)
" > aws-creds.conf
```

> Don't ever check this file into source control - it holds your AWS credentials! For this repository I added `*-creds.conf` to the [.gitignore](.gitignore) file. 

If you're using a CI system like GitHub Actions (as this repository is based on), you need to have both `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` configured as Repository Secrets:

![github-actions-secrets](screenshots/github-actions-secrets.png)

Also make sure to have your `Default region` configured locally - or as a `env:` variable in your CI system. All three needed variables [in GitHub Actions](.github/workflows/provision.yml) for example look like this:

```yaml
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_DEFAULT_REGION: 'eu-central-1'
```

#### Create AWS Provider secret

Now we need to use the `aws-creds.conf` file to create the crossplane AWS Provider secret:

```
kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=./aws-creds.conf
```

If everything went well there should be a new `aws-creds` Secret ready:

![provider-aws-secret](screenshots/provider-aws-secret.png)



#### Install the crossplane AWS Provider

https://crossplane.io/docs/v1.8/concepts/packages.html#installing-a-package

We can install crossplane Packages (which can be Providers or Configurations) via the crossplane CLI with for example:

```shell
kubectl crossplane install provider crossplane/provider-aws:v0.22.0
```

Or we can create our own [provider-aws.yaml](provider-aws.yaml) file like this:

> This `kind: Provider` with `apiVersion: pkg.crossplane.io/v1` is completely different from the `kind: Provider` which we want to consume! These use `apiVersion: meta.pkg.crossplane.io/v1`.

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: crossplane/provider-aws:v0.22.0
  packagePullPolicy: IfNotPresent
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
```

The `packagePullPolicy` configuration here is crucial, since we can configure an update strategy for the Provider here. A full table of all possible fields can be found in the docs: https://crossplane.io/docs/v1.8/concepts/packages.html#specpackagepullpolicy

Now install the AWS provider using `kubectl`:

```
kubectl apply -f provider-aws.yaml
```

With this our first crossplane Provider has been installed. You may check it with `kubectl get provider`:

```shell
$ kubectl get provider
NAME           INSTALLED   HEALTHY   PACKAGE                           AGE
provider-aws   True        Unknown   crossplane/provider-aws:v0.22.0   13s
```

#### Create ProviderConfig to consume the Secret containing AWS credentials

https://crossplane.io/docs/v1.8/getting-started/install-configure.html#configure-the-provider

https://crossplane.io/docs/v1.8/cloud-providers/aws/aws-provider.html#optional-setup-aws-provider-manually

Now we need to create `ProviderConfig` object that will tell the AWS Provider where to find it's AWS credentials. Therefore we create a [provider-config-aws.yaml](provider-config-aws.yaml):

```yaml
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

> Crossplane resources use the `ProviderConfig` named `default` if no specific ProviderConfig is specified, so this ProviderConfig will be the default for all AWS resources.

The `secretRef.name` and `secretRef.key` has to match the fields of the already created Secret.

Apply it with:

```shell
kubectl apply -f provider-config-aws.yaml
```






# Links

https://crossplane.io/

Intro post https://medium.com/@khelge/crossplane-7197ad200013

Crossplane is based on OAM (3 roles: Developer, application operator & infrastructure operator) https://github.com/oam-dev/spec

https://www.kloia.com/blog/production-ready-eks-cluster-with-crossplane

https://www.forbes.com/sites/janakirammsv/2021/09/15/how-crossplane-transforms-kubernetes-into-a-universal-control-plane/

> Crossplane essentially transforms Kubernetes into a universal control plane that can orchestrate the lifecycle of public cloud-based services such as virtual machines, database instances, Big Data clusters, machine learning jobs, and other managed services offered by the hyperscalers. 

> Crossplane takes the concept of Infrastructure as Code (IaC) to the next level through its tight integration with Kubernetes.


Stacks and Applications: https://blog.crossplane.io/crossplane-v0-9-new-package-types-for-providers-stacks-and-applications/

Argo + crossplane https://morningspace.medium.com/using-crossplane-in-gitops-what-to-check-in-git-76c08a5ff0c4