< Back to [training overview](README.md)

# 4. Compositions, XRDs & Claims

About Compositions, XRDs, XRs & Claims (XRCs)

Einführung in Compositions:
Was sind Composite Resources (XR), Claims, Composite Resource Definitions (XRD)?



## 4.1 CompositeResourceDefinitions (or XRDs)

https://docs.crossplane.io/latest/concepts/composite-resource-definitions/

> A CompositeResourceDefinition (or XRD) defines the type and schema of your XR. It lets Crossplane know that you want a particular kind of XR to exist, and what fields that XR should have.

Since defining your own CompositeResourceDefinitions and Compositions is the main work todo with Crossplane, it's always good to know the full Reference documentation which can be found here https://docs.crossplane.io/latest/concepts/compositions/

One of the things to know is that Crossplane automatically injects some common 'machinery' into the manifests of the XRDs and Compositions: https://docs.crossplane.io/latest/concepts/composite-resources/ 





## 4.2 Create your first Composite Resource Definition (XRD)

All possible fields an XRD can have [are documented in the docs](https://docs.crossplane.io/latest/concepts/composite-resource-definitions/).

The field `spec.versions.schema` must contain a OpenAPI schema, which is similar to the ones used by any Kubernetes CRDs. They determine what fields the XR (and Claim) will have. The full CRD documentation and a guide on how to write OpenAPI schemas [could be found in the Kubernetes docs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/).

Note that Crossplane will be automatically extended this section. Therefore the following fields are used by Crossplane and will be ignored if they're found in the schema:

    spec.resourceRef
    spec.resourceRefs
    spec.claimRef
    spec.writeConnectionSecretToRef
    status.conditions
    status.connectionDetails


Now create a new directory `apis/s3` with a new file `definition.yaml`. 

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  # XRDs must be named 'x<plural>.<group>'
  name: xobjectstorages.crossplane.jonashackt.io
spec:
  # This XRD defines an XR in the 'crossplane.jonashackt.io' API group.
  # The XR or Claim must use this group together with the spec.versions[0].name as it's apiVersion, like this:
  # 'crossplane.jonashackt.io/v1alpha1'
  group: crossplane.jonashackt.io
  
  # XR names should always be prefixed with an 'X'
  names:
    kind: XObjectStorage
    plural: xobjectstorages
  # This type of XR offers a claim, which should have the same name without the 'X' prefix
  claimNames:
    kind: ObjectStorage
    plural: objectstorages
  
  # default Composition when none is specified (must match metadata.name of a provided Composition)
  # e.g. in composition.yaml
  defaultCompositionRef:
    name: objectstorage-composition

  versions:
  # This version should only change, when the parameters defined also change - and thus create a new version
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
              parameters:
                type: object
                properties:
                  bucketName:
                    type: string
                  region:
                    type: string
                required:
                  - bucketName
                  - region
```

Install the XRD into our cluster with:

```shell
kubectl apply -f apis/s3/definition.yaml
```

Now the XRD should be installed:

```shell
kubectl get compositeresourcedefinition
NAME                                       ESTABLISHED   OFFERED   AGE
xobjectstorages.crossplane.jonashackt.io   True          True      4d2h
```

We can double check the CRDs beeing created with `kubectl get crds` and filter them using `grep` to our group name `crossplane.jonashackt.io`:

```shell
$ kubectl get crds | grep crossplane.jonashackt.io
objectstorages.crossplane.jonashackt.io                         2022-06-27T09:54:18Z
xobjectstorages.crossplane.jonashackt.io                        2022-06-27T09:54:18Z
```





Aufbau & Entwicklung von XRDs (Kubernetes CRDs)


## 4.3 Build your first Composition

The main work in Crossplane has to be done crafting the Compositions. This is because they interact with the infrastructure primitives the cloud provider APIs provide.

Detailled docs to many of the possible manifest configurations can be found here https://docs.crossplane.io/latest/concepts/compositions/


Now create a new file `composition.yaml` inside `apis/s3`:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: objectstorage-composition
  labels:
    # An optional convention is to include a label of the XRD. This allows
    # easy discovery of compatible Compositions.
    crossplane.io/xrd: xobjectstorages.crossplane.jonashackt.io
    # The following label marks this Composition for AWS. This label can 
    # be used in 'compositionSelector' in an XR or Claim.
    provider: aws
spec:
  # Each Composition must declare that it is compatible with a particular type
  # of Composite Resource using its 'compositeTypeRef' field. The referenced
  # version must be marked 'referenceable' in the XRD that defines the XR.
  compositeTypeRef:
    apiVersion: crossplane.jonashackt.io/v1alpha1
    kind: XObjectStorage
  
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
        # see https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v1.3.1/resources/s3.aws.upbound.io/Bucket/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        metadata: {}
        spec:
          deletionPolicy: Delete
      
      patches:
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
```

As you can see we now use the Managed Resource `Bucket` inside `spec.resources`. We also use so called `patches` to insert parameters into the Bucket. We'll have a deeper look into these Patches later. For now both should be already known from the Composite Resource Definition.

Apply the Composition via:

```shell
kubectl apply -f apis/s3/composition.yaml
```

Have a look at your installed Composition via:

```shell
kubectl get composition
NAME                        XR-KIND          XR-APIVERSION                       AGE
objectstorage-composition   XObjectStorage   crossplane.jonashackt.io/v1alpha1   1m2s
```

Also have a look at the available revisions of your Compositions via `kubectl get compositionrevision`:

```shell
kubectl get compositionrevision
NAME                                REVISION   XR-KIND          XR-APIVERSION                       AGE
objectstorage-composition-a5bf2cb   1          XObjectStorage   crossplane.jonashackt.io/v1alpha1   1m4s
```

There's only one revision for now. We'll come back here later!







## 4.4 Your first Composite Resource (XR) or Claim (XRC) (AWS S3)

Crossplane could look quite intimidating when having a first look. There are few guides around to show how to approach a setup when using Crossplane the first time. You can choose between writing an XR __OR__ XRC! You don't need both, since the XR will be generated from the XRC, if you choose to craft a XRC.

https://docs.crossplane.io/latest/concepts/composite-resources/

Since we want to create a S3 Bucket, we create our first Composite Resource Claim - or simply Claim. Therefore create a new file `objectstorage.yaml` in `infrastructure/s3`:

 here's an suggestion for an [claim.yaml](crossplane-contrib/provider-aws/s3/claim.yaml):

```yaml
# Use the spec.group/spec.versions[0].name defined in the XRD
apiVersion: crossplane.jonashackt.io/v1alpha1
kind: ObjectStorage
metadata:
  # Only claims are namespaced, unlike XRs.
  namespace: default
  name: managed-s3
spec:
  # The compositionRef specifies which Composition this XR will use to compose
  # resources when it is created, updated, or deleted.
  compositionRef:
    name: objectstorage-composition
  
  # Parameters for the Composition to provide the Managed Resources (MR) with
  # to create the actual infrastructure components
  parameters:
    bucketName: crossplane-training-yourNameHere
    region: eu-central-1
```


Apply your first Claim via kubectl:

```shell
kubectl apply -f infrastructure/s3/objectstorage.yaml
```

Again in the AWS console your Bucket should show up:

![](docs/aws-console-initial-bucket-deployed.png)

> Your first Crossplane Composition (incl. it's MRs) should come alive.

When something goes wrong with the validation while applying your Claim, this could look like this:

```shell
$ kubectl apply -f claim.yaml
error: error validating "claim.yaml": error validating data: [ValidationError(S3Bucket.metadata): unknown field "crossplane.io/external-name" in io.k8s.apimachinery.pkg.apis.meta.v1.ObjectMeta_v2, ValidationError(S3Bucket.spec): unknown field "parameters" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec, ValidationError(S3Bucket.spec.writeConnectionSecretToRef): missing required field "namespace" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec.writeConnectionSecretToRef, ValidationError(S3Bucket.spec): missing required field "bucketName" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec, ValidationError(S3Bucket.spec): missing required field "region" in io.jonashackt.crossplane.v1alpha1.S3Bucket.spec]; if you choose to ignore these errors, turn validation off with --validate=false
```

The Crossplane validation is a great way to debug your yaml configuration - it hints you to the actual problems that are still present.

From Crossplane CLI version 1.5 on there's also the way to pre-validate the Claims and Compositions. For more info see the next sections.




## 4.5 Validate your Claims against XRDs

Before using the command have a look at the command reference: https://docs.crossplane.io/latest/cli/command-reference/#beta-validate:

> The crossplane beta validate command validates compositions against provider or XRD schemas using the Kubernetes API server’s validation library.

So let's grab a `definition.yaml` and validate a `claim.yaml` against it:

```shell
crossplane beta validate --cache-dir ~/.crossplane apis/s3/definition.yaml infrastructure/s3/objectstorage.yaml
[✓] crossplane.jonashackt.io/v1alpha1, Kind=ObjectStorage, managed-upbound-s3 validated successfully
Total 1 resources: 0 missing schemas, 1 success cases, 0 failure cases
```

To prevent the command from polluting our projects with `.crossplane` directories, we should also provide a `--cache-dir ~/.crossplane` flag, which will deposit the directory in the user profile folder.


### 4.5.1 Validate a full directory against XRDs or Provider schemes

We can also validate a full directory:

```shell
crossplane beta validate --cache-dir ~/.crossplane apis/s3/definition.yaml upbound/provider-aws/provider
[✓] crossplane.jonashackt.io/v1alpha1, Kind=ObjectStorage, managed-upbound-s3 validated successfully
[!] could not find CRD/XRD for: apiextensions.crossplane.io/v1, Kind=Composition
[!] could not find CRD/XRD for: pkg.crossplane.io/v1, Kind=Provider
[!] could not find CRD/XRD for: aws.upbound.io/v1beta1, Kind=ProviderConfig
[!] could not find CRD/XRD for: apiextensions.crossplane.io/v1, Kind=CompositeResourceDefinition
[✓] s3.aws.upbound.io/v1beta1, Kind=Bucket, crossplane-argocd-s3-bucket validated successfully
[✓] s3.aws.upbound.io/v1beta1, Kind=BucketPublicAccessBlock, crossplane-argocd-s3-bucket-pab validated successfully
[✓] s3.aws.upbound.io/v1beta1, Kind=BucketOwnershipControls, crossplane-argocd-s3-bucket-osc validated successfully
[✓] s3.aws.upbound.io/v1beta1, Kind=BucketACL, crossplane-argocd-s3-bucket-acl validated successfully
[✓] s3.aws.upbound.io/v1beta1, Kind=BucketWebsiteConfiguration, crossplane-argocd-s3-bucket-websiteconf validated successfully
Total 10 resources: 4 missing schemas, 6 success cases, 0 failure cases
```


### 4.5.2 Composition Validation

To be able to validate Compositions & XRs, we need another command in the game: `crossplane beta render`:

https://docs.crossplane.io/latest/cli/command-reference/#validate-render-command-output

> You can pipe the output of `crossplane beta render` into `crossplane beta validate` to validate complete Crossplane resource pipelines, including XRs, compositions and composition functions.

Therefore we need to use the `--include-full-xr` command with `crossplane beta render` and the `-` option with `crossplane beta validate` like that:

```shell
crossplane beta render apis/s3/composition.yaml --include-full-xr | crossplane beta validate upbound/provider-aws/provider/provider-aws-s3.yaml -
```






## 4.6 Compositions using multiple MRs: Patch & Transforms

Aufbau & Entwicklung von Compositions: Patch & Transforms

TODO: Patch & Transforms

https://docs.crossplane.io/latest/concepts/compositions/#changing-resource-fields


? https://github.com/awslabs/crossplane-on-eks/blob/main/doc/patching-101.md


> Discussion: Using Compositions vs. MR-only 






## 4.7 Extend your Composition using multiple MRs

Now let's enhance your already existant Composition using all the needed Managed Resources to provision a publicly accessible S3 Bucket (as we already did in [1.6 Hands-On: Using multiple Managed Resources (public accessible S3 Bucket)](#16-hands-on-using-multiple-managed-resources-public-accessible-s3-bucket)). Therefore add all addidional Managed Resources to the file `apis/s3/composition.yaml`.


```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: objectstorage-composition
  labels:
    # An optional convention is to include a label of the XRD. This allows
    # easy discovery of compatible Compositions.
    crossplane.io/xrd: xobjectstorages.crossplane.jonashackt.io
    # The following label marks this Composition for AWS. This label can 
    # be used in 'compositionSelector' in an XR or Claim.
    provider: aws
spec:
  # Each Composition must declare that it is compatible with a particular type
  # of Composite Resource using its 'compositeTypeRef' field. The referenced
  # version must be marked 'referenceable' in the XRD that defines the XR.
  compositeTypeRef:
    apiVersion: crossplane.jonashackt.io/v1alpha1
    kind: XObjectStorage
  
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
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/Bucket/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        metadata: {}
        spec:
          deletionPolicy: Delete
      
      patches:
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"

    - name: bucketpublicaccessblock
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketPublicAccessBlock/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketPublicAccessBlock
        spec:
          forProvider:
            blockPublicAcls: false
            blockPublicPolicy: false
            ignorePublicAcls: false
            restrictPublicBuckets: false

      patches:
        - fromFieldPath: "spec.parameters.bucketPABName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucketRef.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"

    - name: bucketownershipcontrols
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketOwnershipControls/v1beta1#doc:spec-forProvider-rule-objectOwnership
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketOwnershipControls
        spec:
          forProvider:
            rule:
              - objectOwnership: ObjectWriter

      patches:
        - fromFieldPath: "spec.parameters.bucketOSCName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucketRef.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"

    - name: bucketacl
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketACL/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketACL
        spec:
          forProvider:
            acl: "public-read"

      patches:
        - fromFieldPath: "spec.parameters.bucketAclName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucketRef.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"
  
    - name: bucketwebsiteconfiguration
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketWebsiteConfiguration/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketWebsiteConfiguration
        spec:
          forProvider:
            indexDocument:
              - suffix: index.html

      patches:
        - fromFieldPath: "spec.parameters.bucketWebConfName"
          toFieldPath: "metadata.name"
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "spec.forProvider.bucketRef.name"
        - fromFieldPath: "spec.parameters.region"
          toFieldPath: "spec.forProvider.region"

  # If you find yourself repeating patches a lot you can group them as a named
  # 'patch set' then use a PatchSet type patch to reference them.
  #patchSets:
```


Let's create Composition via:

```shell
kubectl apply -f upbound/provider-aws-s3/composition.yaml
```

Now we should also see another Revision `2` of our Composition was installed (with additional resources). Therefore run `kubectl get compositionrevision`:

```shell
kubectl get compositionrevision
NAME                                REVISION   XR-KIND          XR-APIVERSION                       AGE
objectstorage-composition-4b6be94   2          XObjectStorage   crossplane.jonashackt.io/v1alpha1   2m42s
objectstorage-composition-a5bf2cb   1          XObjectStorage   crossplane.jonashackt.io/v1alpha1   4d2h
```



## 4.8 Opt out of automatic Composition Updates in XRs/Claims

Composition Updates are applied automatically to all XRs/Claims by default. [As the docs state](https://docs.crossplane.io/knowledge-base/guides/composition-revisions/):

> If you have 10 PlatformDB XRs all using the big-platform-db Composition, all 10 of those XRs will be instantly updated in accordance with any updates you make to the big-platform-db Composition.

So if we change a Composition, all the XRs will be updated. This can be a wanted behavior - or not at all.

In the first steps with Crossplane you might encounter the issue already:

```shell
cannot compose resources: cannot parse base template of composed resource "securitygrouprule-cluster-inbound": cannot change the kind or group of a composed resource from ec2.aws.upbound .io/v1beta1, Kind=SecurityGroupRule to ec2.aws.upbound.io/v1beta1, Kind=SecurityGroupIngressRule (possible composed resource template mismatch) 
```

> Composition Revisions allow XRs to opt out of automatic updates.

If you don't want Composition to do automatic updates, you need to [use Composition Revisions](https://docs.crossplane.io/knowledge-base/guides/composition-revisions/) in your XRs/XRCs via the `compositionUpdatePolicy: Manual` keyword:

```yaml
apiVersion: example.org/v1alpha1
kind: PlatformDB
metadata:
  name: example
spec:
  parameters:
    storageGB: 20
  # The Manual policy specifies that you do not want this XR to update to the
  # current CompositionRevision automatically.
  compositionUpdatePolicy: Manual
  compositionRef:
    name: example
  writeConnectionSecretToRef:
    name: db-conn
```



## 4.9 Troubleshooting

https://docs.crossplane.io/knowledge-base/guides/troubleshoot/

> Per Kubernetes convention, Crossplane keeps errors close to the place they happen. This means that if your claim is not becoming ready due to an issue with your Composition or with a composed resource you’ll need to “follow the references” to find out why. Your claim will only tell you that the XR is not yet ready.


The docs also tell us what they mean by "follow the references":

* Find your XR by running `kubectl describe <claim-kind> <claim-metadata.name>` and look for its “Resource Ref” (aka `spec.resourceRef`).
* Run `kubectl describe` on your XR. This is where you’ll find out about issues with the Composition you’re using, if any.
* If there are no issues but your XR doesn’t seem to be becoming ready, take a look for the “Resource Refs” (or `spec.resourceRefs`) to find your composed resources.
* Run `kubectl describe` on each referenced composed resource to determine whether it is ready and what issues, if any, it is encountering.



