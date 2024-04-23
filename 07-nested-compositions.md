 [🔼 training overview](README.md)

# 7. Nested Compositions

> 📝 When Compositions use other Compositions it is called `Nested Compositions` or `Layering Compositions` 

[When to use Nested Compositions](https://docs.upbound.io/xp-arch-framework/building-apis/building-apis-compositions/#when-to-layer-compositions):

> As a best practice, always start by defining your compositions as a flat list of composed managed resources. The complexity involved with debugging a nested composition increases. Scenarios to consider using Nested Compositions:

1. If you find yourself repetitively copying around definitions of managed resources in your compositions, that would be an appropriate time to consider refactoring those resources into their own compositions and nest them.
2. If you have a set of common resource abstractions (such as a standard VPC or bucket) that you tend to use in tandem with other resources, you can compose them once and then nest then in other compositions as needed.
3. In cases where when another team is the owner of a particular managed resource. For example, suppose one team owns “infra,” but wishes to use IAM Policies established by a “security” team. The security team can author IAMPolicy composites and the “infra” team can consume those.

There's not really much documentation about Nested Compositions. There's [this section in the upbound docs about "Layering composite resources"](https://docs.upbound.io/xp-arch-framework/building-apis/building-apis-compositions/#layering-composite-resources). In the XRD docs there are only some hints [about the role of the `XRD.status` field](https://docs.upbound.io/xp-arch-framework/building-apis/building-apis-xrds/#xrd-status).



## 7.1 Example: Company website infrastructure

We simply use parts of [the scenario from this AWS guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/getting-started-cloudfront-overview.html) where one needs to create an S3 bucket to host the company subdomain and another S3 bucket for the root domain.

We will take the following steps in order to build our first nested Composition:

1. Create a XDomainHosting CompositeResourceDefinition (XRD)
2. Craft a Composition 'domain' that will implement the (XRD)
3. Create a Claim DomainHosting
4. Create a XDomainHosting CompositeResourceDefinition (XRD)
5. Craft a Composition 'subdomain' that will implement the (XRD)
6. Create a Claim SubDomainHosting
7. Create the CompositeResourceDefinition (XRD) for our Nested Composition
8. Craft the Nested Composition 'company-website' that will use the other Compositions




## 7.2 Create a XDomainHosting CompositeResourceDefinition (XRD)

First we need to create the folder `apis/company-website/domain` where we create a new Composition for our company's root domain in a new file `definition.yaml`.

The XRD should use a new `spec.group` name with the `domain` prefix (use your )

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdomainhostings.domain.<group>
spec:
  group: domain.<group.example.com>
  names:
    kind: XDomainHosting
    plural: xdomainhostings
  claimNames:
    kind: DomainHosting
    plural: domainhostings
  ...
```

You can simply copy the openAPIV3Schema in the `versions` from the ["4.1.2 Craft your XRD objectstorage for a AWS S3 Bucket"](https://github.com/jonashackt/crossplane-training/blob/main/04-compositions-xrds-claims.md#412-craft-your-xrd-objectstorage-for-a-aws-s3-bucket).

But we need to enhance the openAPIV3Schema to provide a output value `domainurl` inside the `schema.openAPIV3Schema.properties.status` field! Define the return value as your're already used to with other parameters.

If you're finished, install the XRD into our cluster with:

```shell
kubectl apply -f apis/company-website/domain/definition.yaml
```


> 💡 Only, if you're really stuck or don't know what to do, here's a working solution:

<details>
  <summary>🚀 Expand to see a working solution</summary>

[`apis/company-website/domain/definition.yaml`](apis/company-website/domain/definition.yaml):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdomainhostings.domain.crossplane.jonashackt.io
spec:
  group: domain.crossplane.jonashackt.io
  
  names:
    kind: XDomainHosting
    plural: xdomainhostings
  claimNames:
    kind: DomainHosting
    plural: domainhostings

  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          # We define 2 needed parameters here one has to provide as XR or Claim spec.parameters
          spec:
            type: object
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
          # defining return values
          status:
            type: object
            properties:
              domainurl:
                type: string
```
</details>


## 7.3 Craft a Composition 'domain' that will implement the (XRD)


Now let's craft the Composition to implement our XRD:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: domain
  labels:
    crossplane.io/xrd: xdomainhostings.domain.<group.example.com>
    provider: aws
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: domain.<group.example.com>/v1alpha1
    kind: XDomainHosting

  writeConnectionSecretsToNamespace: crossplane-system
  
  resources:
    - name: bucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        ...
```

The Composition should use the same resources as our already created Composition from ["4.6 Extend your Composition using multiple MRs"](https://github.com/jonashackt/crossplane-training/blob/main/04-compositions-xrds-claims.md#46-extend-your-composition-using-multiple-mrs).

In addition to the resources you need to enhance the Bucket's `patches` section with another entry. We want to save the Bucket's region-specific domain name for later usage. Therefore create a new patch of type `ToCompositeFieldPath` use the `bucketRegionalDomainName` in the `fromFieldPath`. You can find the definition of the output parameter in the `Bucket` in the AWS S3 Provider documentation in the Upbound Marketplace (now we will finally need one of the `status` fields...). The `toFieldPath` is already defined in our XRD: `status.domainurl`.


If you're finished, install the Composition into our cluster with:

```shell
kubectl apply -f apis/company-website/domain/composition.yaml
```


> 💡 Only, if you're really stuck or don't know what to do, here's a working solution:

<details>
  <summary>🚀 Expand to see a working solution</summary>

[`apis/company-website/domain/composition.yaml`](apis/company-website/domain/composition.yaml):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: domain
  labels:
    crossplane.io/xrd: xdomainhostings.domain.crossplane.jonashackt.io
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: domain.crossplane.jonashackt.io/v1alpha1
    kind: XDomainHosting
  
  writeConnectionSecretsToNamespace: crossplane-system
  
  resources:
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
        # provide the bucketRegionalDomainName for later use as status.domainurl
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.bucketRegionalDomainName
          toFieldPath: status.domainurl


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
        # use the bucketName parameter to create a derived bucketname-pab
        # see https://docs.crossplane.io/v1.12/concepts/composition/#patch-types
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-pab"
        - type: PatchSet
          patchSetName: bucketNameAndRegionPatchSet

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
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-osc"
        - type: PatchSet
          patchSetName: bucketNameAndRegionPatchSet

    - name: bucketacl
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketACL/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketACL
        spec:
          forProvider:
            acl: "public-read"

      patches:
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-acl"
        - type: PatchSet
          patchSetName: bucketNameAndRegionPatchSet
  
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
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-websiteconf"
        - type: PatchSet
          patchSetName: bucketNameAndRegionPatchSet

  # If you find yourself repeating patches a lot you can group them as a named
  # 'patch set' then use a PatchSet type patch to reference them.
  # see https://docs.crossplane.io/v1.12/concepts/composition/#compositions
  patchSets:
  - name: bucketNameAndRegionPatchSet
    patches:
    - fromFieldPath: "spec.parameters.bucketName"
      toFieldPath: "spec.forProvider.bucketRef.name"
    - fromFieldPath: "spec.parameters.region"
      toFieldPath: "spec.forProvider.region"
```
</details>


## 7.4 Create a Claim DomainHosting

We want to testdrive our Composition already at this point. Therefore we need to create a Claim.

> 📝 It's a best practive developing nested Composition to always test-drive each Composition in isolation also. This way you can be sure each and every part of your Compositions work as expected and not debug problems too late in the development cycle.

Thereore create a new directory `infrastructure/company-website/domain` which will mimic the Composition's folder structure and a file `companydomain.yaml` to inherit the Claim.

Define the Claim `DomainHosting` with two parameters as you're already used to: `bucketName` and `region`.


If you're finished, install the Composition into our cluster with:

```shell
kubectl apply -f infrastructure/company-website/domain/companydomain.yaml
```


> 💡 Only, if you're really stuck or don't know what to do, here's a working solution:

<details>
  <summary>🚀 Expand to see a working solution</summary>

[`infrastructure/company-website/domain/companydomain.yaml`](infrastructure/company-website/domain/companydomain.yaml):

```yaml
apiVersion: domain.crossplane.jonashackt.io/v1alpha1
kind: DomainHosting
metadata:
  namespace: default
  name: company-domain
spec:
  compositionRef:
    name: domain
  
  parameters:
    bucketName: crossplane-training-company-domain-jonas
    region: eu-central-1
```
</details>





## 7.5 Create a XDomainHosting CompositeResourceDefinition (XRD)

We want to implement the AWS website scenario with 2 S3 Buckets for hosting. Since we already created a Bucket for the root domain, we now need an additional Bucket for the sub domain.

Therefore create a folder `apis/company-website/subdomain` where we create a new Composition for our company's sub domain in a new file `definition.yaml`.

The XRD should use a new `spec.group` name with the `subdomain` prefix (use your )

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xsubdomainhostings.domain.<group>
spec:
  group: subdomain.<group.example.com>
  names:
    kind: XSubDomainHosting
    plural: xsubdomainhostings
  claimNames:
    kind: SubDomainHosting
    plural: subdomainhostings
```

You can simply copy the openAPIV3Schema in the `versions` from the ["7.2 Create a XDomainHosting CompositeResourceDefinition (XRD)"]().

As already done in the `xdomainhostings` XRD, we need to enhance the openAPIV3Schema to provide a output value `subdomainurl` inside the `schema.openAPIV3Schema.properties.status` field.

If you're finished, install the XRD into our cluster with:

```shell
kubectl apply -f apis/company-website/subdomain/definition.yaml
```


> 💡 Only, if you're really stuck or don't know what to do, here's a working solution:

<details>
  <summary>🚀 Expand to see a working solution</summary>

[`apis/company-website/subdomain/definition.yaml`](apis/company-website/subdomain/definition.yaml):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xsubdomainhostings.subdomain.crossplane.jonashackt.io
spec:
  group: subdomain.crossplane.jonashackt.io
  
  names:
    kind: XSubDomainHosting
    plural: xsubdomainhostings
  claimNames:
    kind: SubDomainHosting
    plural: subdomainhostings

  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          # We define 2 needed parameters here one has to provide as XR or Claim spec.parameters
          spec:
            type: object
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
          # defining return values
          status:
            type: object
            properties:
              subdomainurl:
                type: string
```
</details>


## 7.6 Craft a Composition 'subdomain' that will implement the (XRD)


Now let's craft the Composition to implement our XRD:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: domain
  labels:
    crossplane.io/xrd: xsubdomainhostings.subdomain.<group.example.com>
    provider: aws
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: subdomain.<group.example.com>/v1alpha1
    kind: XSubDomainHosting

  writeConnectionSecretsToNamespace: crossplane-system
  
  resources:
    - name: bucket
      base:
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: Bucket
        ...
```

The Composition is quite similar to the already created Composition from ["7.3 Craft a Composition 'domain' that will implement the (XRD)"]().

As already done in 7.3 we need to enhance the Bucket's `patches` section with another entry. As with the root Domain Composition we want to save the Bucket's region-specific domain name for later usage. The `toFieldPath` is already defined in our XRD: `status.subdomainurl`.


If you're finished, install the Composition into our cluster with:

```shell
kubectl apply -f apis/company-website/subdomain/composition.yaml
```


> 💡 Only, if you're really stuck or don't know what to do, here's a working solution:

<details>
  <summary>🚀 Expand to see a working solution</summary>

[`apis/company-website/subdomain/composition.yaml`](apis/company-website/subdomain/composition.yaml):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: subdomain
  labels:
    crossplane.io/xrd: xsubdomainhostings.subdomain.crossplane.jonashackt.io
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: subdomain.crossplane.jonashackt.io/v1alpha1
    kind: XSubDomainHosting
  
  writeConnectionSecretsToNamespace: crossplane-system
  
  resources:
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
        # provide the bucketRegionalDomainName for later use as status.subdomainurl
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.bucketRegionalDomainName
          toFieldPath: status.subdomainurl


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
        # use the bucketName parameter to create a derived bucketname-pab
        # see https://docs.crossplane.io/v1.12/concepts/composition/#patch-types
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-pab"
        - type: PatchSet
          patchSetName: bucketNameAndRegionPatchSet

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
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-osc"
        - type: PatchSet
          patchSetName: bucketNameAndRegionPatchSet

    - name: bucketacl
      base:
        # see https://marketplace.upbound.io/providers/upbound/provider-aws/v0.34.0/resources/s3.aws.upbound.io/BucketACL/v1beta1
        apiVersion: s3.aws.upbound.io/v1beta1
        kind: BucketACL
        spec:
          forProvider:
            acl: "public-read"

      patches:
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-acl"
        - type: PatchSet
          patchSetName: bucketNameAndRegionPatchSet
  
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
        - fromFieldPath: "spec.parameters.bucketName"
          toFieldPath: "metadata.name"
          transforms:
            - type: string
              string:
                fmt: "%s-websiteconf"
        - type: PatchSet
          patchSetName: bucketNameAndRegionPatchSet

  # If you find yourself repeating patches a lot you can group them as a named
  # 'patch set' then use a PatchSet type patch to reference them.
  # see https://docs.crossplane.io/v1.12/concepts/composition/#compositions
  patchSets:
  - name: bucketNameAndRegionPatchSet
    patches:
    - fromFieldPath: "spec.parameters.bucketName"
      toFieldPath: "spec.forProvider.bucketRef.name"
    - fromFieldPath: "spec.parameters.region"
      toFieldPath: "spec.forProvider.region"
```
</details>


## 7.7 Create a Claim SubDomainHosting

Again we want to testdrive our second Composition already at this point. Therefore we need to create a Claim. Create a new directory `infrastructure/company-website/subdomain` which will mimic the Composition's folder structure and a file `companysubdomain.yaml` to inherit the Claim.

Define the Claim `SubDomainHosting` with two parameters as you're already used to: `bucketName` and `region`.


If you're finished, install the Composition into our cluster with:

```shell
kubectl apply -f infrastructure/company-website/subdomain/subcompanydomain.yaml
```


> 💡 Only, if you're really stuck or don't know what to do, here's a working solution:

<details>
  <summary>🚀 Expand to see a working solution</summary>

[`infrastructure/company-website/subdomain/companysubdomain.yaml`](infrastructure/company-website/subdomain/companysubdomain.yaml):

```yaml
apiVersion: subdomain.crossplane.jonashackt.io/v1alpha1
kind: SubDomainHosting
metadata:
  namespace: default
  name: company-sub-domain
spec:
  compositionRef:
    name: subdomain
  
  parameters:
    bucketName: crossplane-training-company-sub-domain-jonas
    region: eu-central-1
```
</details>



## 7.8 Create the CompositeResourceDefinition (XRD) for our Nested Composition

Now we're finally where we wanted to be: We can start to create our Nested Composition. A Nested Composition also needs a CompositeResourceDefinition (XRD) - just as a "normal" Composition.

Therefore inside the already existant directory `apis/company-website` create a new file `definition.yaml` to inherit the Nested Composition's XRD:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xcompanywebsites.website.<group>
spec:
  group: website.<group.example.com>
  names:
    kind: XCompanyWebsite
    plural: xcompanywebsites
  claimNames:
    kind: CompanyWebsite
    plural: companywebsites
  ...
```

Also define two input parameters called `companywebsiteprefix` and `region`.

The output parameters in the `status` field should get a field `companywebsiteurls`, which is NOT of type `string` BUT of type `array`. Remember that we defined our Compositions to have `domainurl` and `subdomainurl` fields defined? Both should be saved into the array later. The array definition can look like this:

```yaml
          # define output parameters
          status:
            type: object
            properties:
              companywebsiteurls:
                type: array
                items:
                  type: string
```

If you're finished, install the XRD for our Nested Composition into our cluster with:

```shell
kubectl apply -f apis/company-website/definition.yaml
```


> 💡 Only, if you're really stuck or don't know what to do, here's a working solution:

<details>
  <summary>🚀 Expand to see a working solution</summary>

[`apis/company-website/definition.yaml`](apis/company-website/definition.yaml):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xcompanywebsites.website.crossplane.jonashackt.io
spec:
  group: website.crossplane.jonashackt.io
  names:
    kind: XCompanyWebsite
    plural: xcompanywebsites
  claimNames:
    kind: CompanyWebsite
    plural: companywebsites
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          # define input parameters
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  companywebsiteprefix:
                    type: string
                  region:
                    type: string
                required:
                  - companywebsiteprefix
                  - region
          # define output parameters
          status:
            type: object
            properties:
              companywebsiteurls:
                type: array
                items:
                  type: string
```
</details>


## 7.9 Craft the Nested Composition 'company-website' that will use the other Compositions

We can finally craft the Nested Composition that will use the already created Compositions `XDomainHosting` and `XSubDomainHosting`. As a starting point use the following:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: company-website
spec:
  compositeTypeRef:
    apiVersion: website.<group.example.com>/v1alpha1
    kind: XCompanyWebsite
  ...
```

Now the question is: How to we use the other Compositions inside of another Composition? We simply use the CRDs we defined through the XRDs in the `resources` section. Here's an example:

```yaml
  resources:
    ### Nested use of XDomainHosting XR
    - name: compositeDomain
      base:
        apiVersion: domain.<group.example.com>/v1alpha1
        kind: XDomainHosting
      patches:
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.parameters.region
        ...
```

While crafting Nested Compositions, we can also finally make real use of Patch transforms (as described in ["4.5.3 Example Transforms"](https://github.com/jonashackt/crossplane-training/blob/main/04-compositions-xrds-claims.md#453-example-transforms)). Since we defined a `spec.parameters.companywebsiteprefix` in the Nested Composition XRD, we can use it now as the source of `spec.parameters.bucketName` for each Composition we use and add a `transforms` section. Have a look into the mentioned section 4.5.3 and create a patch transform that will use the `companywebsiteprefix` and suffix it either with `-domain-jonas` or with `-sub-domain-jonas`. 


Also we want to leverage our defined `status.domainurl` and `status.subdomainurl` and use a patch of type `ToCompositeFieldPath` to save them in the Nested Composition's array `status.companywebsiteurls`. You can access the array simply by using an index like this: `toFieldPath: status.companywebsiteurls[0]`.


If you're finished, install the Nested Composition into our cluster with:

```shell
kubectl apply -f apis/company-website/composition.yaml
```



> 💡 Only, if you're really stuck or don't know what to do, here's a working solution:

<details>
  <summary>🚀 Expand to see a working solution</summary>

[`apis/company-website/composition.yaml`](apis/company-website/composition.yaml):

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: company-website
spec:
  compositeTypeRef:
    apiVersion: website.crossplane.jonashackt.io/v1alpha1
    kind: XCompanyWebsite
  
  writeConnectionSecretsToNamespace: crossplane-system

  resources:
    ### Nested use of XDomainHosting XR
    - name: compositeDomain
      base:
        apiVersion: domain.crossplane.jonashackt.io/v1alpha1
        kind: XDomainHosting
      patches:
        - fromFieldPath: spec.parameters.companywebsiteprefix
          toFieldPath: spec.parameters.bucketName
          transforms:
            - type: string
              string:
                fmt: "%s-domain"
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.parameters.region        
        # provide the domainurl as output
        - type: ToCompositeFieldPath
          fromFieldPath: status.domainurl
          toFieldPath: status.companywebsiteurls[0]
          policy:
            fromFieldPath: Required
    

    ### Nested use of XSubDomainHosting XR
    - name: compositeSubDomain
      base:
        apiVersion: subdomain.crossplane.jonashackt.io/v1alpha1
        kind: XSubDomainHosting
      patches:
        - fromFieldPath: spec.parameters.companywebsiteprefix
          toFieldPath: spec.parameters.bucketName
          transforms:
            - type: string
              string:
                fmt: "%s-sub-domain"
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.parameters.region        
        # provide the domainurl as output
        - type: ToCompositeFieldPath
          fromFieldPath: status.subdomainurl
          toFieldPath: status.companywebsiteurls[1]
          policy:
            fromFieldPath: Required
```
</details>



## 7.10 Create a Claim CompanyWebsite to provision the Nested Composition

Finally create a new Claim to issue the provisioning of our Nested Composition. Simply create a file `companywebsite.yaml` in the `infrastructure/company-website` directory:

```yaml
apiVersion: website.crossplane.jonashackt.io/v1alpha1
kind: CompanyWebsite
metadata:
  namespace: default
  name: company-website
spec:
  parameters:
    companywebsiteprefix: crossplane-training-company
    region: eu-central-1
```

Apply the Claim via

```shell
kubectl apply -f infrastructure/company-website/companywebsite.yaml
```

Now have a look at your `k9s` events: what's happening? Are your S3 Buckets showing up in the AWS console?



> 📝 If you successfully build the two Compositions and the Nested Composition, you can also enhance the setup with CloudFront and certificates (as described in the AWS scenario linked in section 7.1). But that's up to you :)