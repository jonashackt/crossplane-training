 [🔼 training overview](README.md)

# 8. Additional Topics

## 8.1 Handling Provider upgrades the GitOps way

Imagine a provider beeing configured like this:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-ec2
spec:
  package: xpkg.upbound.io/upbound/provider-aws-ec2:v1.1.0
  packagePullPolicy: Always
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
```

Now if `provider-aws-ec2:v1.1.1` gets released, the `revisionActivationPolicy: Automatic` will lead to a automatically upgraded provider.

This can be great - or it can lead to a sitation, where the new provider version doesn't work as expected (e.g. because it presents a https://docs.crossplane.io/latest/concepts/providers/#unhealthypackagerevision) - and thus beeing rolled back, also automatically. This could lead to a situation, where the provider gets unusable from time to time.

If we want to do Provider upgrades in a GitOps fashion through a git commit to a git repository, we should configure the `packagePullPolicy` to `IfNotPresent` instead of `Always` (which means " Check for new packages every minute and download any matching package that isn’t in the cache", see https://docs.crossplane.io/master/concepts/packages/#configuration-package-pull-policy) - BUT leave the `revisionActivationPolicy` to `Automatic`! Since otherwise, the Provider will never get active and healty! See https://docs.crossplane.io/master/concepts/packages/#revision-activation-policy), but I didn't find it documented that way!

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-ec2
spec:
  package: xpkg.upbound.io/upbound/provider-aws-ec2:v1.1.0
  packagePullPolicy: IfNotPresent # Only download the package if it isn’t in the cache.
  revisionActivationPolicy: Automatic # Otherwise our Provider never gets activate & healthy
  revisionHistoryLimit: 1
```


## 8.2 Provider & Configuration Package Upgrades with Renovate

[Renovate supports Crossplane by the end of November 2023](https://github.com/renovatebot/renovate/pull/25911). The Renovate docs [tell us how we can configure Crossplane support](https://docs.renovatebot.com/modules/manager/crossplane/):

> 📝 To use the crossplane manager you must set your own fileMatch pattern. The crossplane manager has no default fileMatch pattern, because there is no common filename or directory name convention for Crossplane YAML files. The crossplane manager supports these depTypes: configuration, function, provider

So we always need to explicitely configure Renovate to let it handle our Crossplane Provider & Configuration Package updates for us!

The simplest configuration would be:

```yaml
"crossplane": {
    "fileMatch": ["\\.yaml$"]
  }
```

It makes sense, if most of the files are for Crossplane.


## 8.3 Custom Readiness checks

[Custom readiness checks](https://docs.crossplane.io/latest/concepts/compositions/#resource-readiness-checks) allow Compositions to define what custom conditions to meet for a resource to be Ready.

> 📝 By default Crossplane considers a Composite Resource or Claim as READY when the status of all created resource are Type: Ready and Status: True. Some resources, for example, a ProviderConfig, don’t have a Kubernetes status and are never considered Ready.



## 8.4 Bootstrapping Crossplane with ArgoCD

TBD. See https://github.com/jonashackt/crossplane-argocd


## 8.5 Import existing Resources into Crossplane

https://docs.crossplane.io/latest/guides/import-existing-resources/

It's possible to import existing Resources into Crossplane as Managed Resources: 

> 📝 Crossplane can discover and import existing Provider resources by matching the `crossplane.io/external-name` annotation in a managed resource.

> For example, to import an existing GCP Network named my-existing-network, create a new managed resource and use the my-existing-network in the annotation.

```yaml
apiVersion: compute.gcp.crossplane.io/v1beta1
kind: Network
metadata:
  annotations:
    crossplane.io/external-name: my-existing-network
```

## 8.6 Composition Functions

TBD. See https://docs.crossplane.io/latest/concepts/composition-functions/





