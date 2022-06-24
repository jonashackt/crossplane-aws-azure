# crossplane-kind-eks
[![Build Status](https://github.com/jonashackt/crossplane-kind-eks/workflows/provision/badge.svg)](https://github.com/jonashackt/crossplane-kind-eks/actions)
[![License](http://img.shields.io/:license-mit-blue.svg)](https://github.com/jonashackt/crossplane-kind-eks/blob/master/LICENSE)
[![renovateenabled](https://img.shields.io/badge/renovate-enabled-yellow)](https://renovatebot.com)

Example project showing how to create EKS clusters using crossplane in kind finally run by GitHub Actions


Crossplane https://crossplane.io/ claims to be the "The cloud native control plane framework". It introduces a new way how to manage any cloud resource (beeing it Kubernetes-native or not). It's an alternative Infrastructure-as-Code tooling to Terraform, AWS CDK/Bicep or Pulumi and introduces a higher level of abstraction - based on Kubernetes CRDs. 

> No code required, it’s all declarative!

Litterally the best intro post to crossplane for me was https://blog.crossplane.io/crossplane-vs-cloud-infrastructure-addons/ - here the real key benefits especially compared to other tools are described. Without marketing blabla. If you love deep dives, I can also recommend Nate Reid's blog https://vrelevant.net/ who works as Staff Solutions Engineer at Upbound.


# Comparing Crossplane to Cloud provider infrastructure addons


Crossplane can be also compared to AWS Controllers for Kubernetes (ACK) https://aws.amazon.com/blogs/containers/aws-controllers-for-kubernetes-ack/, Azure Service Operator for Kubernetes (ASO) https://github.com/Azure/azure-service-operator or Google Config Connector https://cloud.google.com/config-connector/docs/overview, which all enable the management of cloud resources through the Kubernetes API. Crossplane providers are even generated from ACK and ASO https://blog.crossplane.io/accelerating-crossplane-provider-coverage-with-ack-and-azure-code-generation-towards-100-percent-coverage-of-all-cloud-services/

> The Crossplane community believes that the typical developer using Kubernetes to deploy their application shouldn’t have to deal with low level infrastructure APIs. 

Crossplane claims to be different:

> Drawing on our experiences as platform builders, SREs, and application developers we’ve designed Crossplane as a toolkit to build your own custom resources on top of any API - often those of the cloud providers. We think this approach is critical to enable usable self-service infrastructure in Kubernetes.

https://blog.crossplane.io/crossplane-vs-cloud-infrastructure-addons/


So it's all about effective and usable self-service infrastructure!



Order Pizza with crossplane https://blog.crossplane.io/providers-101-ordering-pizza-with-kubernetes-and-crossplane/




Crossplane also enables effective multi-cloud interoperability: https://next.redhat.com/2020/10/29/production-ready-deployments-using-the-crossplane-operator-to-manage-and-provision-cloud-native-services/





# crossplane basic concepts

https://crossplane.io/docs/v1.8/concepts/overview.html

* [Managed Resourced (MR)](https://crossplane.io/docs/v1.8/concepts/managed-resources.html): Kubernetes custom resources (CRDs) that represent infrastructure primitives (mostly in cloud providers). All crossplane Managed Resources could be found via https://doc.crds.dev/ 
* [Composite Resources (XR)](https://crossplane.io/docs/v1.8/concepts/composition.html): compose Managed Resources into higher level infrastructure units (especially interesting for platform teams). They are defined by:
    * a `CompositeResourceDefinition` (XRD) (which defines an OpenAPI schema the `Composition` needs to be conform to)
    * (optional) `CompositeResourceClaims` (XRC) (which is an abstraction of the XR for the application team to consume) - but is fantastic to hold the exact configuration parameters for the concrete resources you want to provision
    * a `Composition` that describes the actual infrastructure primitives aka `Managed Resources` used to build the Composite Resource. One XRD could have multiple Compositions - e.g. to one for every environment like development, stating and production
    * and configured by a `Configuration`
* [Packages](https://crossplane.io/docs/v1.8/concepts/packages.html): OCI container images to handle distribution, version updates, dependency management & permissions for Providers & Configurations. Packages were formerly named `Stacks`.
    * [Providers](https://crossplane.io/docs/v1.8/concepts/providers.html): are Packages that bundle a set of Managed Resources & __a Controller to provision infrastructure resources__ - all providers can be found on GitHub, e.g. [provider-aws](https://github.com/crossplane-contrib/provider-aws) or on [docs.crds.dev](https://doc.crds.dev/github.com/crossplane/provider-aws). A [list of all available Providers](https://github.com/orgs/crossplane-contrib/repositories?type=all) can also be found on GitHub.
    * [Configuration](https://crossplane.io/docs/v1.8/getting-started/create-configuration.html): define your own Composite Resources (XRs) & package them via `kubectl crossplane build configuration` (now they are a Package) - and push them to an OCI registry via `kubectl crossplane push configuration`. With this Configurations can also be easily installed into other crossplane clusters.



## Composite Resources (XR)

https://crossplane.io/docs/v1.8/concepts/composition.html#how-it-works

> The first step towards using Composite Resources is configuring crossplane so that it knows what XRs you’d like to exist, and what to do when someone creates one of those XRs. This is done using a `CompositeResourceDefinition` (XRD) resource and one or more `Composition` resources.

![composition-how-it-works](screenshots/composition-how-it-works.svg)

The platform team itself typically has the permissions to create XRs directly. Everyone else uses a lightweight proxy resource called `CompositeResourceClaim` (XC or simply "claim") to create them with crossplane.

If you're familiar with Terrafrom you can think of an XRD as similar to `variable` blocks of a Terrafrom module. The `Composition` could then be seen as the rest of the HCL code describing how to instrument those variables to create actual resources.

Composite Resources can also be nested together, which allows for higher level abstractions and better separation of concerns. Think of an AWS EKS cluster where you need lot's of network / subnetting setup (which can form one XR) and the actual EKS cluster creation with NodeGroups etc (which would be the XR using the networking XR). More in-depth details on nested XRs can be found here https://vrelevant.net/crossplane-beyond-the-basics-nested-xrs-and-composition-selectors/ 



# Getting started with crossplane

In order to use crossplane we'll need any kind of Kubernetes cluster to let it operate in. This management cluster with crossplane installed will then provision the defined infrastructure. Using any managed Kubernetes cluster like EKS, AKS and so on is possible - or even a local [Minikube](https://minikube.sigs.k8s.io/docs/start/), [kind](https://kind.sigs.k8s.io) or [k3d](https://k3d.io/).


## Install prerequisites & fire up a K8s cluster with kind

https://crossplane.io/docs/v1.8/getting-started/install-configure.html

Be sure to have kind, the package manager Helm and kubectl installed:

```shell
brew install kind helm kubectl
```

Also we should install the crossplane CLI

```
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /usr/local/bin
```

Now the `kubectl crossplane --help` command should be ready to use.


Now spin up a local kind cluster

```shell
kind create cluster --image kindest/node:v1.23.0 --wait 5m
```


## Install Crossplane with Helm

https://crossplane.io/docs/v1.8/getting-started/install-configure.html#install-crossplane

```shell
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm upgrade --install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

Check crossplane version installed with `helm list -n crossplane-system` :

```shell
$ helm list -n crossplane-system
NAME      	NAMESPACE        	REVISION	UPDATED                              	STATUS  	CHART           	APP VERSION
crossplane	crossplane-system	1       	2022-06-21 09:28:21.178357 +0200 CEST	deployed	crossplane-1.8.1	1.8.1
```

Before we can actually apply a Provider we have to make sure that crossplane is actually healthy and running. Therefore we can use the `kubectl wait` command like this:

```shell
kubectl wait --for=condition=ready pod -l app=crossplane --namespace crossplane-system --timeout=120s
```

Otherwise we will run into errors like this when applying a `Provider`:

```shell
error: resource mapping not found for name: "provider-aws" namespace: "" from "provider-aws.yaml": no matches for kind "Provider" in version "pkg.crossplane.io/v1"
ensure CRDs are installed first
```

Finally check crossplane status with `kubectl get all -n crossplane-system`:

```shell
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



## Configure Crossplane to access AWS

https://crossplane.io/docs/v1.8/reference/configure.html

https://crossplane.io/docs/v1.8/cloud-providers/aws/aws-provider.html

https://github.com/crossplane-contrib/provider-aws


### Create aws-creds.conf file

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

### Create AWS Provider secret

Now we need to use the `aws-creds.conf` file to create the crossplane AWS Provider secret:

```
kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=./aws-creds.conf
```

If everything went well there should be a new `aws-creds` Secret ready:

![provider-aws-secret](screenshots/provider-aws-secret.png)



### Install the crossplane AWS Provider

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

Before we can actually apply a `ProviderConfig` to our AWS provider we have to make sure that it's actually healthy and running. Therefore we can use the `kubectl wait` command like this:

```shell
kubectl wait --for=condition=healthy --timeout=120s provider/provider-aws
```

Otherwise we may run into errors like this when applying the `ProviderConfig` right after the Provider.


### Create ProviderConfig to consume the Secret containing AWS credentials

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



# Prepare your IDE to help understand crossplane manifests

https://blog.upbound.io/moving-crossplane-package-authoring-from-plain-yaml-to-ide-aided-development/

Upbound VS Code plugin https://blog.upbound.io/crossplane-vscode-plugin-announcement/

![upbound-vscode-extension](screenshots/upbound-vscode-extension.png)




# Provision a S3 Bucket with crossplane

__The crossplane core Controller and the Provider AWS Controller should now be ready to provision any infrastructure component in AWS!__

You heard right: we don't create a Kubernetes based infrastructure component - but we start with a simple S3 Bucket.

Therefore we can have a look into the crossplane AWS provider API docs: https://doc.crds.dev/github.com/crossplane/provider-aws/s3.aws.crossplane.io/Bucket/v1beta1@v0.18.1


## Defining all Composite Resource components to provide an AWS EKS cluster

https://crossplane.io/docs/v1.8/concepts/composition.html#defining-composite-resources

> A CompositeResourceDefinition (or XRD) defines the type and schema of your XR. It lets Crossplane know that you want a particular kind of XR to exist, and what fields that XR should have.

Since defining your own CompositeResourceDefinitions and Compositions is the main work todo with crossplane, it's always good to know the full Reference documentation which can be found here https://crossplane.io/docs/v1.8/reference/composition.html

One of the things to know is that crossplane automatically injects some common 'machinery' into the manifests of the XRDs and Compositions: https://crossplane.io/docs/v1.8/reference/composition.html#composite-resources-and-claims


### Craft a Composite Resource (XR) or Claim (XRC)

Crossplane could look quite intimidating when having a first look. There are few guides around to show how to approach a setup when using crossplane the first time. For me I read lots of docs - and in the end found, that starting by crafting the Composite Resource (XR) or a corresponding Claim (XRC) might be a great option. Since we get ourselves into thinking what we actually want to provision. You can choose between writing an XR __OR__ XRC! You don't need both, since the XR will be generated from the XRC, if you choose to craft a XRC.

https://crossplane.io/docs/v1.8/reference/composition.html#composite-resources-and-claims

Since we want to create a S3 Bucket, here's an suggestion for an [claim.yaml](crossplane-s3/claim.yaml):

```yaml
---
# Use the spec.group/spec.versions[0].name defined in the XRD
apiVersion: crossplane.jonashackt.io/v1alpha1
kind: S3Bucket
metadata:
  # Only claims are namespaced, unlike XRs.
  namespace: default
  name: managed-s3
spec:
  # The compositionSelector allows you to match a Composition by labels rather
  # than naming one explicitly. It is used to set the compositionRef if none is
  # specified explicitly.
  compositionSelector:
    matchLabels:
      environment: development
      region: eu-central
      provider: aws
  # The writeConnectionSecretToRef field specifies a Kubernetes Secret that this
  # XR(C) should write its connection details (if any) to (XR need to specify a namespace here)
  writeConnectionSecretToRef:
    name: managed-s3-connection-details
  # Parameters for the Composition to provide the Managed Resources (MR) with
  # to create the actual infrastructure components
  parameters:
    bucketName: microservice-ui-nuxt-js-static-bucket
    region: eu-central-1
```



### Defining a CompositeResourceDefinition (XRD) for our S3 Bucket

All possible fields an XRD can have are documented here:

https://crossplane.io/docs/v1.8/reference/composition.html#compositeresourcedefinitions

The field `spec.versions.schema` must contain a OpenAPI schema, which is similar to the ones used by any Kubernetes CRDs. They determine what fields the XR (and claim) will have. The full CRD documentation and a guide on how to write OpenAPI schemas could be found in the Kubernetes docs: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

Note that crossplane will be automatically extended this section. Therefore the following fields are used by crossplane and will be ignored if they're found in the schema:

    spec.resourceRef
    spec.resourceRefs
    spec.claimRef
    spec.writeConnectionSecretToRef
    status.conditions
    status.connectionDetails


So our Composite Resource Definition (XRD) for our S3 Bucket could look like [crossplane-s3/xrd.yaml](crossplane-s3/xrd.yaml):

```yaml
---
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  # XRDs must be named 'x<plural>.<group>'
  name: xs3buckets.crossplane.jonashackt.io
spec:
  # This XRD defines an XR in the 'crossplane.jonashackt.io' API group.
  # The XR or Claim must use this group together with the spec.versions[0].name as it's apiVersion, like this:
  # 'crossplane.jonashackt.io/v1alpha1'
  group: crossplane.jonashackt.io
  
  # XR names should always be prefixed with an 'X'
  names:
    kind: XS3Bucket
    plural: xs3buckets
  
  # This type of XR offers a claim, which should have the same name without the 'X' prefix
  claimNames:
    kind: S3Bucket
    plural: s3buckets
  
  # The keys the XR writes to it's connection secret
  # which will act as a filter during aggregation of the connection secret from
  # composed resources.
  connectionSecretKeys:
    - secretKey
    - accessKey
    - host
  
  # default Composition when none is specified (must match metadata.name of a provided Composition)
  # e.g. in composition.yaml
  defaultCompositionRef:
    name: s3bucket

  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    # OpenAPI schema (like the one used by Kubernetes CRDs). Determines what fields
    # the XR (and claim) will have. Will be automatically extended by crossplane.
    # See https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
    # for full CRD documentation and guide on how to write OpenAPI schemas
    schema:
      openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          # We define 2 needed parameters here one has to provide as XR or Claim spec.parameters
          properties:
            bucketName:
              type: string
            region:
              type: string
          required:
            - bucketName
            - region
```



### Craft a Composition to manage our needed cloud resources

The main work in crossplane has to be done crafting the Compositions. This is because they interact with the infrastructure primitives the cloud provider APIs provide.

Detailled docs to many of the possible manifest configurations can be found here https://crossplane.io/docs/v1.8/reference/composition.html#compositions

A Composite to manage an S3 Bucket in AWS with public access for static website hosting could for example look like this [crossplane-s3/composition.yaml](crossplane-s3/composition.yaml):

```yaml
---
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: s3bucket
  labels:
    # An optional convention is to include a label of the XRD. This allows
    # easy discovery of compatible Compositions.
    crossplane.io/xrd: xeksclusters.crossplane.jonashackt.io
    # The following label marks this Composition for AWS. This label can 
    # be used in 'compositionSelector' in an XR or Claim.
    provider: aws
spec:
  # Each Composition must declare that it is compatible with a particular type
  # of Composite Resource using its 'compositeTypeRef' field. The referenced
  # version must be marked 'referenceable' in the XRD that defines the XR.
  compositeTypeRef:
    apiVersion: crossplane.jonashackt.io/v1alpha1
    kind: XS3Bucket
  
  # When an XR is created in response to a claim Crossplane needs to know where
  # it should create the XR's connection secret. This is configured using the
  # 'writeConnectionSecretsToNamespace' field.
  writeConnectionSecretsToNamespace: crossplane-system
  
  # Each Composition must specify at least one composed resource template.
  resources:
    
    # Providing a unique name for each entry is good practice.
    # Only identifies the resources entry within the Composition. Required in future crossplane API versions.
    - name: bucket
      base:
        # see https://doc.crds.dev/github.com/crossplane/provider-aws/s3.aws.crossplane.io/Bucket/v1beta1@v0.18.1
        apiVersion: s3.aws.crossplane.io/v1beta1
        kind: Bucket
        metadata: {}
        spec:
          forProvider:
            # public-read should enable public access for static website hosting
            # see https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket
            acl: "public-read"
          # This must match with the metadata.name of our configure Provider in provider-aws.yaml
          providerConfigRef:
            name: provider-aws
          deletionPolicy: Delete
      
      # Each resource can optionally specify a set of 'patches' that copy fields
      # from (or to) the XR.
      patches:
        # All those fieldPath refer to XR or Claim spec.parameters
        - fromFieldPath: "spec.bucketName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.region"
          toFieldPath: "spec.forProvider.locationConstraint"
  
  # If you find yourself repeating patches a lot you can group them as a named
  # 'patch set' then use a PatchSet type patch to reference them.
  #patchSets:
```


### Create & build a Configuration using a crossplane.yaml

Finally we want to test-drive our Composite Resource, if it is able to create a S3 Bucket for us! Therefore we need a `crossplane.yaml` file as described in https://crossplane.io/docs/v1.8/getting-started/create-configuration.html#build-and-push-the-configuration 

See also https://crossplane.io/docs/v1.8/concepts/packages.html#configuration-packages

Our [crossplane-s3/crossplane.yaml](crossplane-s3/crossplane.yaml) is of `kind: Configuration` and defines the minimum crossplane version needed alongside the crossplane AWS provider:

```yaml
apiVersion: meta.pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: s3-bucket-composition
spec:
  crossplane:
    version: ">=v1.8"
  dependsOn:
    - provider: crossplane/provider-aws
      version: v0.22.0
```

Having this `crossplane.yaml` in place we can build our Configuration at last. On a command line go into the directory where the `crossplane.yaml` resides and run the `kubectl crossplane build` command:

```shell
$ cd crossplane-s3
$ kubectl crossplane build configuration
```









# Provision an EKS cluster with crossplane

__The crossplane core Controller and the Provider AWS Controller should now be ready to provision any infrastructure component in AWS!__

So in order to maximise the Inception let's provision an EKS based Kubernetes cluster in AWS with our crossplane Kubernetes cluster :) 

A EKS cluster is a complex AWS construct leveraging different basic AWS services. That's exactly what a crossplane Composite Resource (XR) is all about. So let's create our own Composite Resource.


## Defining all Composite Resource components to provide an AWS EKS cluster

https://crossplane.io/docs/v1.8/concepts/composition.html#defining-composite-resources

> A CompositeResourceDefinition (or XRD) defines the type and schema of your XR. It lets Crossplane know that you want a particular kind of XR to exist, and what fields that XR should have.

Since defining your own CompositeResourceDefinitions and Compositions is the main work todo with crossplane, it's always good to know the full Reference documentation which can be found here https://crossplane.io/docs/v1.8/reference/composition.html

One of the things to know is that crossplane automatically injects some common 'machinery' into the manifests of the XRDs and Compositions: https://crossplane.io/docs/v1.8/reference/composition.html#composite-resources-and-claims


### Craft a Composite Resource (XR) or Claim (XRC)

Crossplane could look quite intimidating when having a first look. There are few guides around to show how to approach a setup when using crossplane the first time. For me I read lots of docs - and in the end found, that starting by crafting the Composite Resource (XR) or a corresponding Claim (XRC) might be a great option. Since we get ourselves into thinking what we actually want to provision. You can choose between writing an XR __OR__ XRC! You don't need both, since the XR will be generated from the XRC, if you choose to craft a XRC.

https://crossplane.io/docs/v1.8/reference/composition.html#composite-resources-and-claims

Since we want to create a AWS EKS cluster, here's an suggestion for an [claim.yaml](eks-crossplane/claim.yaml):

```yaml
---
# Use the spec.group/spec.versions[0].name later defined in the XRD
apiVersion: crossplane.jonashackt.io/v1alpha1
kind: EKSCluster
metadata:
  # Only claims are namespaced, unlike XRs.
  namespace: default
  name: managed-eks
spec:
  # The compositionSelector allows you to match a Composition by labels rather
  # than naming one explicitly. It is used to set the compositionRef if none is
  # specified explicitly.
  compositionSelector:
    matchLabels:
      environment: development
      region: eu-central
      provider: aws
  # The writeConnectionSecretToRef field specifies a Kubernetes Secret that this
  # XR(C) should write its connection details (if any) to (XR need to specify a namespace here)
  writeConnectionSecretToRef:
    name: managed-eks-connection-details
  # Parameters for the Composition to provide the Managed Resources (MR) with
  # to create the actual infrastructure components
  parameters:
    region: eu-central-1
    k8s-version: "1.21"
    workers-size: 2
```



### Defining a CompositeResourceDefinition (XRD) for our EKS cluster

All possible fields an XRD can have are documented here:

https://crossplane.io/docs/v1.8/reference/composition.html#compositeresourcedefinitions

The field `spec.versions.schema` must contain a OpenAPI schema, which is similar to the ones used by any Kubernetes CRDs. They determine what fields the XR (and claim) will have. The full CRD documentation and a guide on how to write OpenAPI schemas could be found in the Kubernetes docs: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

Note that crossplane will be automatically extended this section. Therefore the following fields are used by crossplane and will be ignored if they're found in the schema:

    spec.resourceRef
    spec.resourceRefs
    spec.claimRef
    spec.writeConnectionSecretToRef
    status.conditions
    status.connectionDetails




# Higher level abstractions - or why it is so hard to write your own Compositions

Looking only at the crossplane docs and blog I thought something is missing: A curated library of higher level abstractions (like Pulumi Crosswalk). First initiatives from cloud vendors like AWS Blueprints for Crossplane: https://aws.amazon.com/blogs/opensource/introducing-aws-blueprints-for-crossplane/

But then I found the Upbound blog - and finally stumbled upon the __Upbound Platform Reference Architectures__ https://blog.upbound.io/azure-reference-platform/ which are intended as

> foundations to help accelerate understanding and application of Crossplane within your organization


### Use Upbound Platform Reference Architecture AWS to provision a EKS cluster

__What I didn't wanted to do is to duplicate some code__ into my `Composition` from all those sources I found (mainly this https://www.kloia.com/blog/production-ready-eks-cluster-with-crossplane, this https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/ and https://aws.amazon.com/blogs/opensource/introducing-aws-blueprints-for-crossplane/) about provisioning a EKS cluster with crossplane. 




### Install platform-ref-aws

https://github.com/upbound/platform-ref-aws#install-the-platform-configuration




# Conclusion

Crossplane really promising




Transforming Terraform code into Crossplane CRDs is now possible using crossplane generator Terrajet https://blog.crossplane.io/announcing-terrajet/



Crossplane's Roadmap can be viewed as GitHub project board https://github.com/orgs/crossplane/projects?type=classic There are interesting issues for the future:

Crossplane and ArgoCD integration issues: https://github.com/crossplane/crossplane/issues/2773

Follow the crossplane & Upbound blogs which provides the great insights https://blog.crossplane.io and https://blog.upbound.io/



# Links

https://crossplane.io/

Intro post https://medium.com/@khelge/crossplane-7197ad200013

Crossplane is based on OAM (3 roles: Developer, application operator & infrastructure operator) https://github.com/oam-dev/spec

https://www.kloia.com/blog/production-ready-eks-cluster-with-crossplane

https://www.forbes.com/sites/janakirammsv/2021/09/15/how-crossplane-transforms-kubernetes-into-a-universal-control-plane/

> Crossplane essentially transforms Kubernetes into a universal control plane that can orchestrate the lifecycle of public cloud-based services such as virtual machines, database instances, Big Data clusters, machine learning jobs, and other managed services offered by the hyperscalers. 

> Crossplane takes the concept of Infrastructure as Code (IaC) to the next level through its tight integration with Kubernetes.

Intro Slides: https://docs.google.com/presentation/d/1PxZweRpB6HElxd9qGK1McboGZ1kluCDCS5qxgYnX5f0/edit#slide=id.g8801599ecb_2_169


Promoted from sandbox to incubation by the CNCF: https://blog.crossplane.io/crossplane-cncf-incubation/ 

Crossplane vs. Terraform: https://blog.crossplane.io/crossplane-vs-terraform/





https://www.techtarget.com/searchitoperations/news/252508176/Crossplane-project-could-disrupt-infrastructure-as-code


Argo + crossplane https://morningspace.medium.com/using-crossplane-in-gitops-what-to-check-in-git-76c08a5ff0c4



VMWare support https://blog.crossplane.io/adding-vmware-support-to-crossplane-using-terraform/


### Crossplane + AWS

https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/

> Crossplane’s infrastructure provider for AWS relies on code generated by the AWS Go Code Generator, which is also used by [AWS Controllers for Kubernetes (ACK)](https://github.com/aws-controllers-k8s/community). 

https://www.kloia.com/blog/production-ready-eks-cluster-with-crossplane




AWS Blueprints for Crossplane: 

https://aws.amazon.com/blogs/opensource/introducing-aws-blueprints-for-crossplane/

> Writing your first Composition and debugging can be intimidating.

And it's quite a doublicate effort IMHO compared to other tools like Pulumi that provide higher level abstractions including well-architectured best practices (like Pulumi https://www.pulumi.com/docs/guides/crosswalk/aws/)

https://github.com/aws-samples/crossplane-aws-blueprints



### Stacks now called Packages

https://github.com/crossplane/crossplane/blob/master/design/design-doc-composition.md

>  Note that Crossplane packages were formerly known as 'Stacks'. This term has been rescoped to refer specifically to a package of infrastructure configuration but is frequently synonymous with packages in historical design documents and some remaining Crossplane APIs.


## Others

#### KUTTL - KUbernetes Test TooL

https://kuttl.dev/

example: https://github.com/aws-samples/crossplane-aws-blueprints/blob/main/tests/kuttl/test-suite.yaml