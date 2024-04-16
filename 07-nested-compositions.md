 [🔼 training overview](README.md)

# 7. Nested Compositions

> 📝 When Compositions use other Compositions it is called `Nested Compositions` or `Layering Compositions` 

There's not really much documentation about Nested Compositions. There's [this section in the upbound docs about "Layering composite resources"](https://docs.upbound.io/xp-arch-framework/building-apis/building-apis-compositions/#layering-composite-resources).

In the XRD docs there are only some hints [about the role of the `XRD.status` field](https://docs.upbound.io/xp-arch-framework/building-apis/building-apis-xrds/#xrd-status).

## 7.1 A good example: Build a EKS Cluster with Crossplane

Most information [is provided by this blog post](https://vrelevant.net/crossplane-beyond-the-basics-nested-xrs-and-composition-selectors/) and some examples like [this (watch out, this is based on the crossplane aws provider!)](https://github.com/cem-altuner/crossplane-prod-ready-eks) and [this](https://github.com/upbound/configuration-eks).

So why not bootstrap a EKS cluster with Crossplane?!

To achieve this goal we need to do the following:

1. Add EKS, ECS & IAM Providers
2. Craft a Networking Composition based on EC2 & IAM
3. Craft a EKS Cluster Composition
4. A Nested XR for Networking & EKS Cluster Compositions
5. Accessing the Crossplane provisioned EKS cluster



## 7.2 Add EKS, ECS & IAM Providers

We first need to add 3 more Crossplane Providers from the upbound provider families for EKS, EC2 and IAM.

Therefore create 3 new Provider manifests.

A [`upbound/provider-aws/provider/provider-aws-eks.yaml`](upbound/provider-aws/provider/provider-aws-eks.yaml):

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-aws-eks
spec:
  package: xpkg.upbound.io/upbound/provider-aws-eks:v1.2.1
  packagePullPolicy: IfNotPresent # Only download the package if it isn’t in the cache.
  revisionActivationPolicy: Automatic # Otherwise our Provider never gets activate & healthy
  revisionHistoryLimit: 1
```

a [`upbound/provider-aws/provider/provider-aws-ec2.yaml`](upbound/provider-aws/provider/provider-aws-ec2.yaml):

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-aws-ec2
spec:
  package: xpkg.upbound.io/upbound/provider-aws-ec2:v1.2.1
  packagePullPolicy: IfNotPresent # Only download the package if it isn’t in the cache.
  revisionActivationPolicy: Automatic # Otherwise our Provider never gets activate & healthy
  revisionHistoryLimit: 1
```

and a [`crossplane/provider/upbound-provider-aws-iam.yaml`](crossplane/provider/upbound-provider-aws-iam.yaml):

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-aws-iam
spec:
  package: xpkg.upbound.io/upbound/provider-aws-iam:v1.2.1
  packagePullPolicy: IfNotPresent # Only download the package if it isn’t in the cache.
  revisionActivationPolicy: Automatic # Otherwise our Provider never gets activate & healthy
  revisionHistoryLimit: 1
```



## 7.3 Craft a Networking Composition based on EC2 & IAM

Can be found in `apis/networking/` directory:

* XRD: [`apis/networking/definition.yaml`](apis/networking/definition.yaml)


<details>
  <summary>expand full yaml</summary>

  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xnetworkings.net.aws.crossplane.jonashackt.io
spec:
  group: net.aws.crossplane.jonashackt.io
  names:
    kind: XNetworking
    plural: xnetworkings
  claimNames:
    kind: Networking
    plural: networkings
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            # defining input parameters
            spec:
              type: object
              properties:
                id:
                  type: string
                  description: ID of this Network that other objects will use to refer to it.
                parameters:
                  type: object
                  description: Network configuration parameters.
                  properties:
                    region:
                      type: string
                  required:
                    - region
              required:
                - id
                - parameters
            # defining return values
            status:
              type: object
              properties:
                subnetIds:
                  type: array
                  items:
                    type: string
                securityGroupClusterIds:
                  type: array
                  items:
                    type: string
  ```
</details>

* Composition: [`apis/networking/composition.yaml`](apis/networking/composition.yaml)

<details>
  <summary>expand full yaml</summary>
  
  ```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: networking
  labels:
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: net.aws.crossplane.jonashackt.io/v1alpha1
    kind: XNetworking

  writeConnectionSecretsToNamespace: crossplane-system

  patchSets:
  - name: networkconfig
    patches:
    - type: FromCompositeFieldPath
      fromFieldPath: spec.id
      toFieldPath: metadata.labels[net.aws.crossplane.jonashackt.io/network-id] # the network-id other Composition MRs (like EKSCluster) will use
    - type: FromCompositeFieldPath
      fromFieldPath: spec.parameters.region
      toFieldPath: spec.forProvider.region

  resources:
    ### VPC and InternetGateway
    - name: platform-vcp
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: VPC
        spec:
          forProvider:
            cidrBlock: 10.0.0.0/16
            enableDnsSupport: true
            enableDnsHostnames: true
            tags:
              Owner: Platform Team
              Name: platform-vpc
      patches:
        - type: PatchSet
          patchSetName: networkconfig
        - fromFieldPath: spec.id
          toFieldPath: metadata.name
    
    - name: gateway
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: InternetGateway
        spec:
          forProvider:
            vpcIdSelector:
              matchControllerRef: true
      patches:
        - type: PatchSet
          patchSetName: networkconfig


    ### Subnet Configuration
    - name: subnet-public-eu-central-1a
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: Subnet
        metadata:
          labels:
            access: public
        spec:
          forProvider:
            mapPublicIpOnLaunch: true
            cidrBlock: 10.0.0.0/24
            vpcIdSelector:
              matchControllerRef: true
            tags:
              kubernetes.io/role/elb: "1"
      patches:
        - type: PatchSet
          patchSetName: networkconfig
        # define eu-central-1a as zone & availabilityZone
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: metadata.labels.zone
          transforms:
            - type: string
              string:
                fmt: "%sa"
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.availabilityZone
          transforms:
            - type: string
              string:
                fmt: "%sa"
        # provide the subnetId for later use as status.subnetIds entry
        - type: ToCompositeFieldPath
          fromFieldPath: metadata.annotations[crossplane.io/external-name]
          toFieldPath: status.subnetIds[0]
    
    - name: subnet-public-eu-central-1b
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: Subnet
        metadata:
          labels:
            access: public
        spec:
          forProvider:
            mapPublicIpOnLaunch: true
            cidrBlock: 10.0.1.0/24
            vpcIdSelector:
              matchControllerRef: true
            tags:
              kubernetes.io/role/elb: "1"
      patches:
        - type: PatchSet
          patchSetName: networkconfig
          # define eu-central-1b as zone & availabilityZone
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: metadata.labels.zone
          transforms:
            - type: string
              string:
                fmt: "%sb"
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.availabilityZone
          transforms:
            - type: string
              string:
                fmt: "%sb"
          # provide the subnetId for later use as status.subnetIds entry
        - type: ToCompositeFieldPath
          fromFieldPath: metadata.annotations[crossplane.io/external-name]
          toFieldPath: status.subnetIds[1]

    - name: subnet-public-eu-central-1c
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: Subnet
        metadata:
          labels:
            access: public
        spec:
          forProvider:
            mapPublicIpOnLaunch: true
            cidrBlock: 10.0.2.0/24
            vpcIdSelector:
              matchControllerRef: true
            tags:
              kubernetes.io/role/elb: "1"
      patches:
        - type: PatchSet
          patchSetName: networkconfig
          # define eu-central-1c as zone & availabilityZone
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: metadata.labels.zone
          transforms:
            - type: string
              string:
                fmt: "%sc"
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.availabilityZone
          transforms:
            - type: string
              string:
                fmt: "%sc"
          # provide the subnetId for later use as status.subnetIds entry
        - type: ToCompositeFieldPath
          fromFieldPath: metadata.annotations[crossplane.io/external-name]
          toFieldPath: status.subnetIds[2]  

    ### SecurityGroup & SecurityGroupRules Cluster API server
    - name: securitygroup-cluster
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: SecurityGroup
        metadata:
          labels:
            net.aws.crossplane.jonashackt.io: securitygroup-cluster
        spec:
          forProvider:
            description: cluster API server access
            name: securitygroup-cluster
            vpcIdSelector:
              matchControllerRef: true
      patches:
        - type: PatchSet
          patchSetName: networkconfig
        - fromFieldPath: spec.id
          toFieldPath: metadata.name
          # provide the securityGroupId for later use as status.securityGroupClusterIds entry
        - type: ToCompositeFieldPath
          fromFieldPath: metadata.annotations[crossplane.io/external-name]
          toFieldPath: status.securityGroupClusterIds[0]

    - name: securitygrouprule-cluster-inbound
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: SecurityGroupRule
        spec:
          forProvider:
            #description: Allow pods to communicate with the cluster API server & access API server from kubectl clients
            type: ingress
            cidrBlocks:
              - 0.0.0.0/0
            fromPort: 443
            toPort: 443
            protocol: tcp
            securityGroupIdSelector:
              matchLabels:
                net.aws.crossplane.jonashackt.io: securitygroup-cluster
      patches:
        - type: PatchSet
          patchSetName: networkconfig

    - name: securitygrouprule-cluster-outbound
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: SecurityGroupRule
        spec:
          forProvider:
            description: Allow internet access from the cluster API server
            type: egress
            cidrBlocks: # Destination
              - 0.0.0.0/0
            fromPort: 0
            toPort: 0
            protocol: tcp
            securityGroupIdSelector:
              matchLabels:
                net.aws.crossplane.jonashackt.io: securitygroup-cluster
      patches:
        - type: PatchSet
          patchSetName: networkconfig

    ### Route, RouteTable & RouteTableAssociations
    - name: route
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: Route
        spec:
          forProvider:
            destinationCidrBlock: 0.0.0.0/0
            gatewayIdSelector:
              matchControllerRef: true
            routeTableIdSelector:
              matchControllerRef: true
      patches:
        - type: PatchSet
          patchSetName: networkconfig

    - name: routeTable
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: RouteTable
        spec:
          forProvider:
            vpcIdSelector:
              matchControllerRef: true
      patches:
      - type: PatchSet
        patchSetName: networkconfig

    - name: mainRouteTableAssociation
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: MainRouteTableAssociation
        spec:
          forProvider:
            routeTableIdSelector:
              matchControllerRef: true
            vpcIdSelector:
              matchControllerRef: true
      patches:
        - type: PatchSet
          patchSetName: networkconfig

    - name: RouteTableAssociation-public-a
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: RouteTableAssociation
        spec:
          forProvider:
            routeTableIdSelector:
              matchControllerRef: true
            subnetIdSelector:
              matchControllerRef: true
              matchLabels:
                access: public
      patches:
        - type: PatchSet
          patchSetName: networkconfig
        # define eu-central-1a as subnetIdSelector.matchLabels.zone
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.subnetIdSelector.matchLabels.zone
          transforms:
            - type: string
              string:
                fmt: "%sa"

    - name: RouteTableAssociation-public-b
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: RouteTableAssociation
        spec:
          forProvider:
            routeTableIdSelector:
              matchControllerRef: true
            subnetIdSelector:
              matchControllerRef: true
              matchLabels:
                access: public
      patches:
        - type: PatchSet
          patchSetName: networkconfig
        # define eu-central-1b as subnetIdSelector.matchLabels.zone
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.subnetIdSelector.matchLabels.zone
          transforms:
            - type: string
              string:
                fmt: "%sb"

    - name: RouteTableAssociation-public-c
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: RouteTableAssociation
        spec:
          forProvider:
            routeTableIdSelector:
              matchControllerRef: true
            subnetIdSelector:
              matchControllerRef: true
              matchLabels:
                access: public
      patches:
        - type: PatchSet
          patchSetName: networkconfig
        # define eu-central-1c as subnetIdSelector.matchLabels.zone
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.subnetIdSelector.matchLabels.zone
          transforms:
            - type: string
              string:
                fmt: "%sc"
  ```
</details>


For the start, let's simply apply our first XRD, Composition and Claim manually like that:

```shell
# Networking XRD & Composition
kubectl apply -f apis/networking/definition.yaml
kubectl apply -f apis/networking/composition.yaml
# Precheck if Network works
kubectl apply -f examples/networking/claim.yaml
```

I found that the simplest way to follow what Crossplane is doing, is to look into the events ( via typing `:events`) in k9s:

![](docs/follow-crossplane-events-in-k9s.png)

And simply press `ENTER` to see the actual event message. This helped me a lot in the development process (no need to run `kubectl get crossplane` all the time and manually copy the CRD names to a `kubectl describe xyz-crd`).



Managed Resources need to reference other Managed Resources. For example, a `SecurityGroupRule` needs to reference a `SecurityGroup`:

```yaml
...
    ### SecurityGroups & Rules
    - name: securitygroup-nodepool
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: SecurityGroup
        spec:
          forProvider:
            description: Cluster communication with worker nodes
            name: securitygroup-nodepool
            vpcIdSelector:
              matchControllerRef: true

      patches:
        - type: PatchSet
          patchSetName: networkconfig
        - fromFieldPath: spec.id
          toFieldPath: metadata.name
          # provide the securityGroupId for later use as status.securityGroupIds entry
        - type: ToCompositeFieldPath
          fromFieldPath: metadata.annotations[crossplane.io/external-name]
          toFieldPath: status.securityGroupIds[0]

    - name: securitygroup-nodepool-rule
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: SecurityGroupRule
        spec:
          forProvider:
            type: egress
            cidrBlocks:
              - 0.0.0.0/0
            fromPort: 0
            protocol: tcp
            securityGroupIdSelector:
              matchLabels:
                net.aws.crossplane.jonashackt.io: securitygroup
            toPort: 0
      patches:
        - type: PatchSet
          patchSetName: networkconfig
          ...
```

In this example, we get the following error in our k8s events:

```shell
cannot resolve references: mg.Spec.ForProvider.SecurityGroupID: no resources matched selector
```

https://docs.crossplane.io/latest/concepts/managed-resources/#referencing-other-resources states

> Some fields in a managed resource may depend on values from other managed resources. For example a VM may need the name of a virtual network to use.

> Managed resources can reference other managed resources by external name, name reference or selector.

The problem is, we don't specify the `net.aws.crossplane.jonashackt.io: securitygroup` label on our `SecurityGroup`! Doing that the problem is gone:

```yaml
        kind: SecurityGroup
        metadata:
          labels:
            net.aws.crossplane.jonashackt.io: securitygroup
```


There should now be an event showing up containing `Successfully composed resources` in our `eks-vpc-j8s5k` XR.



## 7.4 Craft a EKS Cluster Composition

Can be found in `apis/eks/` directory:

* XRD: [`apis/eks/definition.yaml`](apis/eks/definition.yaml)

<details>
  <summary>expand full yaml</summary>

  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  # XRDs must be named 'x<plural>.<group>'
  name: xeksclusters.eks.aws.crossplane.jonashackt.io
spec:
  # This XRD defines an XR in the 'crossplane.jonashackt.io' API group.
  # The XR or Claim must use this group together with the spec.versions[0].name as it's apiVersion, like this:
  # 'crossplane.jonashackt.io/v1alpha1'
  group: eks.aws.crossplane.jonashackt.io
  
  # XR names should always be prefixed with an 'X'
  names:
    kind: XEKSCluster
    plural: xeksclusters
  # This type of XR offers a claim, which should have the same name without the 'X' prefix
  claimNames:
    kind: EKSCluster
    plural: ekscluster
  
  # default Composition when none is specified (must match metadata.name of a provided Composition)
  # e.g. in composition.yaml
  defaultCompositionRef:
    name: aws-eks

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
          # defining input parameters
          spec:
            type: object
            properties:
              id:
                type: string
                description: ID of this Cluster that other objects will use to refer to it.
              parameters:
                type: object
                description: EKS configuration parameters.
                properties:
                  # Using subnetIds & securityGroupClusterIds from XNetworking to configure VPC
                  subnetIds:
                    type: array
                    items:
                      type: string
                  securityGroupClusterIds:
                    type: array
                    items:
                      type: string
                  region:
                    type: string
                  nodes:
                    type: object
                    description: EKS node configuration parameters.
                    properties:
                      count:
                        type: integer
                        description: Desired node count, from 1 to 10.
                    required:
                    - count
                required:
                - subnetIds
                - securityGroupClusterIds
                - region
                - nodes
            required:
            - id
            - parameters
          # defining return values
          status:
            type: object
            properties:
              clusterStatus:
                description: The status of the control plane
                type: string
              nodePoolStatus:
                description: The status of the node pool
                type: string
  ```
</details>

* Composition: [`apis/eks/composition.yaml`](apis/eks/composition.yaml)

<details>
  <summary>expand full yaml</summary>

  ```yaml
  apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: aws-eks
  labels:
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: eks.aws.crossplane.jonashackt.io/v1alpha1
    kind: XEKSCluster
  
  writeConnectionSecretsToNamespace: crossplane-system

  patchSets:
  - name: clusterconfig
    patches:
    - fromFieldPath: spec.parameters.region
      toFieldPath: spec.forProvider.region

  resources:
    ### Cluster Configuration
    - name: eksCluster
      base:
        apiVersion: eks.aws.upbound.io/v1beta1
        kind: Cluster
        metadata:
          annotations:
            meta.upbound.io/example-id: eks/v1beta1/cluster
            uptest.upbound.io/timeout: "2400"
        spec:
          forProvider:
            roleArnSelector:
              matchControllerRef: true
              matchLabels:
                role: clusterRole
            vpcConfig:
              - endpointPrivateAccess: true
                endpointPublicAccess: true
      patches:
        - type: PatchSet
          patchSetName: clusterconfig
        - fromFieldPath: spec.id
          toFieldPath: metadata.name
        # Using the XNetworking defined securityGroupClusterIds & subnetIds for the vpcConfig
        - fromFieldPath: spec.parameters.securityGroupClusterIds
          toFieldPath: spec.forProvider.vpcConfig[0].securityGroupIds
        - fromFieldPath: spec.parameters.subnetIds
          toFieldPath: spec.forProvider.vpcConfig[0].subnetIds

        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.status
          toFieldPath: status.clusterStatus    
      readinessChecks:
        - type: MatchString
          fieldPath: status.atProvider.status
          matchString: ACTIVE

    - name: kubernetesClusterAuth
      base:
        apiVersion: eks.aws.upbound.io/v1beta1
        kind: ClusterAuth
        spec:
          forProvider:
            clusterNameSelector:
              matchControllerRef: true
      patches:
        - type: PatchSet
          patchSetName: clusterconfig
        - fromFieldPath: spec.writeConnectionSecretToRef.namespace
          toFieldPath: spec.writeConnectionSecretToRef.namespace
        - fromFieldPath: spec.id
          toFieldPath: spec.writeConnectionSecretToRef.name
          transforms:
            - type: string
              string:
                fmt: "%s-access"
      connectionDetails:
        - fromConnectionSecretKey: kubeconfig

    ### Cluster Role and Policies
    - name: clusterRole
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: Role
        metadata:
          labels:
            role: clusterRole
        spec:
          forProvider:
            assumeRolePolicy: |
              {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": [
                                "eks.amazonaws.com"
                            ]
                        },
                        "Action": [
                            "sts:AssumeRole"
                        ]
                    }
                ]
              }
      
    
    - name: clusterRolePolicyAttachment
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: RolePolicyAttachment
        spec:
          forProvider:
            policyArn: arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
            roleSelector:
              matchControllerRef: true
              matchLabels:
                role: clusterRole


    ### NodeGroup Configuration
    - name: nodeGroupPublic
      base:
        apiVersion: eks.aws.upbound.io/v1beta1
        kind: NodeGroup
        spec:
          forProvider:
            clusterNameSelector:
              matchControllerRef: true
            nodeRoleArnSelector:
              matchControllerRef: true
              matchLabels:
                role: nodegroup
            subnetIdSelector:
              matchLabels:
                access: public
            scalingConfig:
              - minSize: 1
                maxSize: 10
                desiredSize: 1
            instanceTypes: # TODO: we can support to have that parameterized also
              - t3.medium
      patches:
        - type: PatchSet
          patchSetName: clusterconfig
        - fromFieldPath: spec.parameters.nodes.count
          toFieldPath: spec.forProvider.scalingConfig[0].desiredSize
        - fromFieldPath: spec.id
          toFieldPath: spec.forProvider.subnetIdSelector.matchLabels[net.aws.crossplane.jonashackt.io/network-id]
        - type: ToCompositeFieldPath
          fromFieldPath: status.atProvider.status
          toFieldPath: status.nodePoolStatus  
      readinessChecks:
      - type: MatchString
        fieldPath: status.atProvider.status
        matchString: ACTIVE

    ### Node Role and Policies
    - name: nodegroupRole
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: Role
        metadata:
          labels:
            role: nodegroup
        spec:
          forProvider:
            assumeRolePolicy: |
              {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Principal": {
                            "Service": [
                                "ec2.amazonaws.com"
                            ]
                        },
                        "Action": [
                            "sts:AssumeRole"
                        ]
                    }
                ]
              }
      

    - name: workerNodeRolePolicyAttachment
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: RolePolicyAttachment
        spec:
          forProvider:
            policyArn: arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
            roleSelector:
              matchControllerRef: true
              matchLabels:
                role: nodegroup
      

    - name: cniRolePolicyAttachment
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: RolePolicyAttachment
        spec:
          forProvider:
            policyArn: arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
            roleSelector:
              matchControllerRef: true
              matchLabels:
                role: nodegroup
      
    - name: containerRegistryRolePolicyAttachment
      base:
        apiVersion: iam.aws.upbound.io/v1beta1
        kind: RolePolicyAttachment
        spec:
          forProvider:
            policyArn: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
            roleSelector:
              matchControllerRef: true
              matchLabels:
                role: nodegroup
      
  ```
</details>

For testing we simply use `kubectl apply -f`:


```shell
# EKS XRD & Composition
kubectl apply -f apis/eks/definition.yaml
kubectl apply -f apis/eks/composition.yaml

# If you choose this example (non-nested) claim, be sure to change the subnetIds and securitygroupid according the the Networking claim executed before!

# Precheck if EKSCluster works
kubectl apply -f examples/eks/claim.yaml 
```

Errors in the events like this are normal, since the EKS Cluster needs it's time to be provisioned before NodeGroups etc. can be assigned:

```shell
cannot resolve references: mg.Spec.ForProvider.ClusterName: referenced field was empty (referenced resource may not yet be ready) 
```

This also shows up in the AWS console:

![](docs/eks-cluster-initial-provisioning.png)


Now if the `NodeGroup` comes up with the following

```shell
cannot resolve references: mg.Spec.ForProvider.SubnetIds: no resources matched selector 
```

there's a problem, where the NodeGroup can't find it's SubnetIds.

```yaml
    - name: nodeGroupPublic
      base:
        apiVersion: eks.aws.upbound.io/v1beta1
        kind: NodeGroup
        spec:
          forProvider:
            clusterNameSelector:
              matchControllerRef: true
            nodeRoleArnSelector:
              matchControllerRef: true
              matchLabels:
                role: nodegroup
            subnetIdSelector:
              matchLabels:
                access: public
            scalingConfig:
              - minSize: 1
                maxSize: 10
                desiredSize: 1
            instanceTypes: # TODO: we can support to have that parameterized also
              - t3.medium
      patches:
        - type: PatchSet
          patchSetName: clusterconfig
        - fromFieldPath: spec.parameters.nodes.count
          toFieldPath: spec.forProvider.scalingConfig[0].desiredSize
        - fromFieldPath: spec.id
          toFieldPath: spec.forProvider.subnetIdSelector.matchLabels[aws.crossplane.jonashackt.io/network-id]
          ...
```

That's because the label of all networking components changed to `net.aws.crossplane.jonashackt.io/network-id`. So let's fix that!

Now finally the NodeGroups are correctly assigned to the EKS cluster:

The `Successfully composed resources` message in the event `xekscluster/deploy-target-eks-cb87r` looks promising:

![](docs/eks-cluster-with-nodegroups.png)



## 7.5 A Nested XR for Networking & EKS Cluster Compositions

Can be found in `apis/` directory:

* XRD: [`apis/definition.yaml`](apis/definition.yaml)
* Composition: [`apis/composition.yaml`](apis/composition.yaml)

With this Composition we're able to use both pre-defined Compositions `XNetworking` and `XEKSCluster` and thus implement a nested Composite Resource:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: kubernetes-cluster
spec:
  compositeTypeRef:
    apiVersion: k8s.crossplane.jonashackt.io/v1alpha1
    kind: XKubernetesCluster
  
  writeConnectionSecretsToNamespace: crossplane-system

  resources:
    ### Nested use of XNetworking XR
    - name: compositeNetworkEKS
      base:
        apiVersion: net.aws.crossplane.jonashackt.io/v1alpha1
        kind: XNetworking
      patches:
        - fromFieldPath: spec.id
          toFieldPath: spec.id
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.parameters.region
        # provide the subnetIds & securityGroupIds for later use
        - type: ToCompositeFieldPath
          fromFieldPath: status.subnetIds
          toFieldPath: status.subnetIds
          policy:
            fromFieldPath: Required
        - type: ToCompositeFieldPath
          fromFieldPath: status.securityGroupIds
          toFieldPath: status.securityGroupIds
          policy:
            fromFieldPath: Required
    
    ### Nested use of XEKSCluster XR
    - name: compositeClusterEKS
      base:
        apiVersion: eks.aws.crossplane.jonashackt.io/v1alpha1
        kind: XEKSCluster
      connectionDetails:
        - fromConnectionSecretKey: kubeconfig
      patches:
        - fromFieldPath: spec.id
          toFieldPath: spec.id
        - fromFieldPath: spec.id
          toFieldPath: metadata.annotations[crossplane.io/external-name]
        - fromFieldPath: metadata.uid
          toFieldPath: spec.writeConnectionSecretToRef.name
          transforms:
            - type: string
              string:
                fmt: "%s-eks"
        - fromFieldPath: spec.writeConnectionSecretToRef.namespace
          toFieldPath: spec.writeConnectionSecretToRef.namespace
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.parameters.region
        - fromFieldPath: spec.parameters.nodes.count
          toFieldPath: spec.parameters.nodes.count
        - fromFieldPath: status.subnetIds
          toFieldPath: spec.parameters.subnetIds
          policy:
            fromFieldPath: Required
        - fromFieldPath: status.securityGroupIds
          toFieldPath: spec.parameters.securityGroupIds
          policy:
            fromFieldPath: Required
```

For the start, let's simply apply our first XRD, Composition and Claim manually like that:

```shell
# Nested XRD & Composition
kubectl apply -f apis/definition.yaml
kubectl apply -f apis/composition.yaml

# Check if full Cluster provisioning works
kubectl apply -f examples/claim.yaml
```



## 7.6 Accessing the Crossplane provisioned EKS cluster

https://docs.crossplane.io/knowledge-base/guides/connection-details/

In our eks cluster [claim](upbound/provider-aws/apis/eks/claim.yaml) we defined a 

```yaml
  writeConnectionSecretToRef:
    name: eks-cluster-kubeconfig
```

inside our nested claim. This will create a k8s `Secret` called `eks-cluster-kubeconfig`, where the kubeconfig will be stored.

Let's extract the kubeconfig:

```shell
kubectl get secret eks-cluster-kubeconfig -o jsonpath='{.data.kubeconfig}' | base64 --decode > ekskubeconfig
```

Now integrate the contents of the `ekskubeconfig` file into your `~/.kube/config` (better with VSCode!) and switch over to the new kube context e.g. using https://github.com/ahmetb/kubectx. If you're on the new context of our Crossplane bootstrapped EKS cluster, check if everything works:

```shell
$ kubectl get nodes
NAME                                          STATUS   ROLES    AGE   VERSION
ip-10-0-0-173.eu-central-1.compute.internal   Ready    <none>   34m   v1.29.0-eks-5e0fdde
ip-10-0-1-149.eu-central-1.compute.internal   Ready    <none>   34m   v1.29.0-eks-5e0fdde
ip-10-0-2-90.eu-central-1.compute.internal    Ready    <none>   34m   v1.29.0-eks-5e0fdde
```
