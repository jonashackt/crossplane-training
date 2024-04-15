# crossplane-training
Crossplane training & hands-on workshop materials

This repo contains a training about [crossplane.io](https://www.crossplane.io/) and uses lecture & hands-on tasks to introduce the __cloud native control plane framework__. This repo focusses on AWS, but can be easily adapted to other (cloud) computing vendors.

## Prerequisites

AWS iam user with rights to deploy S3, IAM roles, EC2 instances, VPCs & networking components (Subnets, SecurityGroups & Rules, etc), EKS clusters & node groups.





### Install Crossplane CLI

> For this training we need Crossplane CLI >= `v1.15.0`!

https://github.com/crossplane/crossplane/releases/tag/v1.15.0

> Enhancements to the Crossplane CLI: New subcommands like `crossplane beta validate` for schema validation, `crossplane beta top` for resource utilization views similar to `kubectl top pods`, and `crossplane beta convert` for converting resources to newer formats or configurations.

Install crossplane CLI:

```shell
curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh" |sh
```

If that produces an error like `Failed to download Crossplane CLI. Please make sure version current exists on channel stable.`, try to manually craft the download link:

```shell
curl --output crank "https://releases.crossplane.io/stable/current/bin/linux_amd64/crank"
chmod +x crank
sudo mv crank /usr/local/bin/crossplane
```

Be sure to have the `v1.15.0` version installed as a minimum, otherwise the `crossplane beta validate` command won't work:

```shell
crossplane --version
v1.15.0
```




# Training overview


## 1. Intro & basic concepts

### 1.1 Crossplane concepts: Managed Resources, Providers & Families, Configuration Packages

IaC with control planes: what about state?

The "management cluster"

Basic concepts, Managed Resources (MR), Providers (incl. crossplane-contrib vs. upbound, Provider Families, based on Terraform)

Embedding Crossplane into GitOps processes


### 1.2 Hands-On: Create your first management cluster (kind)

Hands-on: Aufbau eines lokalen Managed Clusters mit Crossplane (Basis: kind)

Installation Crossplane (incl. Diskussion Helm, ArgoCD)


### 1.3 Hands-On: Install & configure a Crossplane Provider

Provider-Konfiguration am Beispiel AWS: Secret, Provider, ProviderConfig (incl. Diskussion Unterschiede zu Provider Azure)

Einstieg in die Crossplane-Doku: Marketplace Provider-Doku, crossplane Doku


### 1.4 Hands-On: Using your first Managed Resource (AWS S3)

Hands-on: Nutzung Managed Resources am Beispiel AWS S3, Provisionierung, Erweiterung um public Access (multiple Managed Resources)

`managed-s3`

> Your first Crossplane managed infrastructure resource should come alive.


### 1.5 Validate your MR against Provider schemes

crossplane CLI: Validierung von MRs gegen Provider-Schemas


### 1.6 Hands-On: Using multiple Managed Resources (public accessible S3 Bucket)

Using multiple Managed Resources without Compositions is possible... but :)

`managed-s3-public`



## 2. Compositions, XRDs & Claims

### 2.1 About Compositions, XRDs, XRs & Claims (XRCs)

Einführung in Compositions:
Was sind Composite Resources (XR), Claims, Composite Resource Definitions (XRD)?


### 2.2 Hands-On: Building your first Composite Resource Definition (XRD)

Aufbau & Entwicklung von XRDs (Kubernetes CRDs)


### 2.3 Hands-On: Building your first Composition

`composition-s3`

Hands-On: Entwicklung einer ersten Composition auf Basis einer MRs


### 2.4 Hands-On: Crafting your first XR/Claim (AWS S3)

Entwicklung von XRs bzw. Claims

> Your first Crossplane Composition (incl. it's MRs) should come alive.


### 2.5 Validate your Claims against XRDs

crossplane CLI: Validierung von Claims gegen XRDs


### 2.6 Compositions using multiple MRs: Patch & Transforms

Aufbau & Entwicklung von Compositions: Patch & Transforms


### 2.6 Hands-On: Extend your Compositions using multiple MRs

Hands-On: Entwicklung einer Composition mit mehreren MRs


> Discussion: Using Compositions vs. MR-only 



## 3. Managing Compositions: Configuration Packages & Revisions

Development: How to structure your Composition repository

https://docs.crossplane.io/latest/concepts/packages/

Configuration Packages: Nutzung, Revisions

Hands-On: Anlegen eines eigenen Packages, optional build per crossplane xpkg
Versionierung von Compositions

Steps to create your own Configuration Package:

0. Install Crossplane CLI
1. Create crossplane.yaml
2. Building the Package using Crossplane CLI
3. Pushing the Package file .xpkg to a Container Registry (e.g. GitHub CR)
4. Optional: Building & Publishing your Configuration Package automatically with GitHub Actions
5. Install the Configuration Package into the management cluster
6. Optional: Create a new Package Revision and install it into the management cluster


### 3.1 Hands-On: Create crossplane.yaml

As [stated in the docs](https://docs.crossplane.io/latest/concepts/packages/#create-a-configuration) Crossplane has a feature where one can create Configuration Packages containing specific Compositions packaged in a OCI container. So let's build a Configuration Package from our AWS S3 Composition here.

There's a template one could use to create the crossplane.yaml using the crossplane CLI: https://docs.crossplane.io/latest/cli/command-reference/#beta-xpkg-init 

```shell
crossplane beta xpkg init crossplane-training-s3 configuration-template
```

The command uses the following template: https://github.com/crossplane/configuration-template (one could provide arbitrary repositories to the command).


Here's also a full example [`crossplane.yaml`](crossplane.yaml):

```yaml
apiVersion: meta.pkg.crossplane.io/v1alpha1
kind: Configuration
metadata:
  name: crossplane-eks-cluster
  annotations:
    # Set the annotations defining the maintainer, source, license, and description of your Configuration
    meta.crossplane.io/maintainer: Jonas Hecht iam@jonashackt.io
    meta.crossplane.io/source: github.com/jonashackt/crossplane-eks-cluster
    # Set the license of your Configuration
    meta.crossplane.io/license: MIT
    meta.crossplane.io/description: |
      Crossplane Configuration delivering CRDs to provision AWS EKS clusters.
    meta.crossplane.io/readme: |
      It's a Nested Composition with separate AWS Networking & EKS cluster setups.
spec:
  dependsOn:
    - provider: xpkg.upbound.io/upbound/provider-aws-ec2
      version: ">=v1.1.1"
    - provider: xpkg.upbound.io/upbound/provider-aws-iam
      version: ">=v1.1.1"
    - provider: xpkg.upbound.io/upbound/provider-aws-eks
      version: ">=v1.1.1"
  crossplane:
    version: ">=v1.15.1-0"
```

Don't forget to add the `metadate.annotations` in order to prevent the error `crossplane: error: failed to build package: not exactly one package meta type` - see also https://stackoverflow.com/questions/78200917/crossplane-error-failed-to-build-package-not-exactly-one-package-meta-type

Also we should define on which providers our Configuration depends on - and also on which Crossplane version.




### 3.2 Hands-On: Building the Package using Crossplane CLI

The Crossplane CLI [has the right command for us](https://docs.crossplane.io/latest/concepts/packages/#build-the-package):

```shell
crossplane xpkg build --package-root=. --examples-root="./examples" --ignore=".github/workflows/*,crossplane/provider/*,kuttl-test.yaml,tests/compositions/s3/*" --verbose
```

Note that including YAML files that aren’t Compositions or CompositeResourceDefinitions, including Claims isn’t supported.

This can be done by appending `--ignore="file.xyz,directory/*"`, which will ignore a file `file.xyz` and all files in directory `directory`. [Sadly, ingoring directories completely isn't supported right now](https://docs.crossplane.io/latest/concepts/packages/#build-the-package) - so we need to define all our kuttl test directories respectively.

Also appending `--verbose` makes a lot of sense to see what's going on.



### 3.3 Hands-On: Pushing the Package file .xpkg to GitHub Container Registry

There's also a `crossplane xpkg push` command to publish the Configuration package. So let's create a new GitHub package matching our repository:

```shell
crossplane xpkg push ghcr.io/jonashackt/crossplane-training-s3:v0.0.1
```

You can leverage the Container image tag as version number for your Configuration here.

If the command gives the following error, we need to setup Authentication for your Docker Registry:

```shell
crossplane: error: failed to push package file crossplane-eks-cluster-7badc365c06a.xpkg: Post "https://ghcr.io/v2/jonashackt/crossplane-eks-cluster/blobs/uploads/": GET https://ghcr.io/token?scope=repository%3Ajonashackt%2Fcrossplane-eks-cluster%3Apull&scope=repository%3Ajonashackt%2Fcrossplane-eks-cluster%3Apush%2Cpull&service=ghcr.io: DENIED: requested access to the resource is denied
```

The ` crossplane xpkg push --help` helps us:

> Credentials for the registry are automatically retrieved from xpkg
login and dockers configuration as fallback.

So we need to login to GitHub Container Registry first in order to be able to push our OCI image:

```shell
echo $CR_PAT | docker login ghcr.io -u YourAccountOrGHOrgaNameHere --password-stdin
```

Make sure to use a Personal Access Token as described in this post https://www.codecentric.de/wissens-hub/blog/github-container-registry with the following scopes (`repo`, `write:packages` and `delete:packages`):

![](docs/github-container-registry-pat-scopes.png)

Additionally we need to add the domain configuration like this: `--domain=https://ghcr.io`. Otherwise the default domain is `upbound.io` which will lead to non pushed Configurations - only visible via the `verbose` flag.

With this our `crossplane xpkg push` command should work as expected:

```shell
$ crossplane xpkg push ghcr.io/jonashackt/crossplane-eks-cluster:v0.0.1 --domain=https://ghcr.io --verbose

2024-03-21T16:39:48+01:00	DEBUG	Found package in directory	{"path": "crossplane-eks-cluster-7badc365c06a.xpkg"}
2024-03-21T16:39:48+01:00	DEBUG	Getting credentials for server	{"serverURL": "ghcr.io"}
2024-03-21T16:39:48+01:00	DEBUG	No profile specified, using default profile
2024-03-21T16:39:49+01:00	DEBUG	Pushed package	{"path": "crossplane-eks-cluster-7badc365c06a.xpkg", "ref": "ghcr.io/jonashackt/crossplane-eks-cluster:v0.0.1"}
```

Now head over to your GitHub Organisation's `Packages` tab and search for the newly created package:

![](docs/github-container-registry-package-connect-repository.png)

Click onto the package and connect the GitHub Repository.

Also - on the right - click on `Package settings` and scroll down to the `Danger Zone`. There click on `Change visibility` and change it to public. Now your Crossplane Configuration should be available for download without login.

If everything went fine, the package / OCI image should now be visible at your repository.


### 3.4 Optional Hands-On: Building & Publishing your Configuration Package automatically with GitHub Actions

So let's finally do it all automatically on Composition code changes (git commit/push) using GitHub Actions. We simply extend our workflow at [.github/workflows/test-composition-and-publish-to-ghcr.yml](.github/workflows/test-composition-and-publish-to-ghcr.yml) and do all the steps from above:

```yaml
name: publish

on: [push]

env:
  GHCR_PAT: ${{ secrets.GHCR_PAT }}
  CONFIGURATION_VERSION: "v0.0.2"

jobs:
  resouces-rendering-test:
    ...

jobs:
  build-configuration-and-publish-to-ghcr:
    needs: resouces-rendering-test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_PAT }}

      - name: Install Crossplane CLI
        run: |
          curl -sL "https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh" |sh
          sudo mv crossplane /usr/local/bin

      - name: Build Crossplane Configuration package & publish it to GitHub Container Registry
        run: |
          echo "### Build Configuration .xpkg file"
          crossplane xpkg build --package-root=. --examples-root="./examples" --ignore=".github/workflows/*,crossplane/install/*,crossplane/provider/*,kuttl-test.yaml,tests/compositions/eks/*,tests/compositions/networking/*" --verbose

          echo "### Publish as OCI image to GHCR"
          crossplane xpkg push "ghcr.io/jonashackt/crossplane-eks-cluster:$CONFIGURATION_VERSION" --domain=https://ghcr.io --verbose
```

As we added the `.github/workflows` directory with a workflow yaml file, the `crossplane xpkg build` command also tries to include it. Therefore the command locally need to exclude the workflow file also:

```shell
crossplane xpkg build --package-root=. --examples-root="./examples" --ignore=".github/workflows/*,crossplane/install/*,crossplane/provider/*,kuttl-test.yaml,tests/compositions/eks/*,tests/compositions/networking/*" --verbose
```

`--ignore=".github/*` won't work, since the command doesn't support to exclude directories - only wildcards IN directories.

Also to prevent the following error:

```shell
crossplane: error: failed to push package file crossplane-eks-cluster-7badc365c06a.xpkg: PUT https://ghcr.io/v2/jonashackt/crossplane-eks-cluster/manifests/v0.0.2: DENIED: installation not allowed to Write organization package
```

we use the Personal Access Token (PAT) we already created above also in our GitHub Actions Workflow instead of the default `GITHUB_TOKEN` in order to have the correct permissions. Therefore create it as a new Repository Secret:

![](docs/create-repository-secret.png)

With this we should also be able to use a ENV var for our Configuration version or even `latest`.


### 3.5 Hands-On: Install the Configuration Package in Crossplane's management cluster

As described in https://docs.crossplane.io/latest/concepts/packages/#install-a-configuration use a manifest like the following to install the Configuration. Therefore create a new file in the `apis` directory (since this is a Crossplane API / CRD we will be using) and name it `crossplane-training-s3.yaml` (just as our Composition is named):

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Configuration
metadata:
  name: crossplane-training-s3
spec:
  package: ghcr.io/jonashackt/crossplane-training-s3:v0.0.1
```

Now install all APIs in the `apis` folder via:

```shell
kubectl apply -f apis/
```

As you're already used to, create (or re-use) a Claim to use the newly installed API:

```yaml
apiVersion: crossplane.jonashackt.io/v1alpha1
kind: ObjectStorage
metadata:
  namespace: default
  name: managed-upbound-s3
spec:
  compositionRef:
    name: objectstorage-composition
  
  parameters:
    bucketName: spring2024-bucket
    region: eu-central-1
```

Run `kubectl get crossplane` and after applying the new Configuration here, there should appear all the Compositions and XRDs.




## 4. Testing Compositions

Tests vs. Integration tests

uptest vs. kuttl




### Always Test your Composition before publishing it

To prevent the `build-configuration-and-publish-to-ghcr` step from running before the test job successfully finished, [we use the `needs: resouces-rendering-test` keyword as described here](https://stackoverflow.com/a/65698892/4964553).

Now our Pipeline should look like this:

![](docs/only-publish-when-test-successful-pipeline.png)






## 5. Nested Compositions

Wie können Compositions wiederverwendet werden?
Rückgabe-Werte (status) Nutzen in Nested Compositions
Hands-On: Entwicklung einer ersten Nested Composition





## 6. Bootstrapping Crossplane using GitOps principles

### 6.1 Handling Provider upgrades in a GitOps fashion


### 6.2 Provider upgrades with Renovate


### 6.3 Crossplane & ArgoCD






Entwicklungs-Guidelines / typische Pitfalls


