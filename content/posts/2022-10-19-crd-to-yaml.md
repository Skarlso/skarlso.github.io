+++
author = "hannibal"
categories = ["go", "showcase"]
date = "2022-10-19T01:01:00+01:00"
title = "Generate a sample YAML file from a CRD"
url = "/2022/10/19/crd-to-yaml"
comments = true
+++

Hello.

This one is a quick update. Just a showcase really.

I wrote a tool to generate a sample YAML file from a CRD.

Given a CRD like [this](https://github.com/kubernetes-sigs/cluster-api-provider-aws/blob/b58d6141229f49201e3e2abbde3453c5d420037a/config/crd/bases/infrastructure.cluster.x-k8s.io_awsclusters.yaml) one, it would output a generate yaml sample like this:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata: {}
spec:
  additionalTags: {}
  bastion:
    allowedCIDRBlocks: ["string"]
    ami: string
    disableIngressRules: true
    enabled: true
    instanceType: string
  controlPlaneEndpoint:
    host: string
    port: 1
  controlPlaneLoadBalancer:
    additionalSecurityGroups: ["string"]
    crossZoneLoadBalancing: true
    healthCheckProtocol: string
    name: string
    scheme: string
    subnets: ["string"]
  identityRef:
    kind: AWSCluster
    name: string
  imageLookupBaseOS: string
  imageLookupFormat: string
  imageLookupOrg: string
  network:
    cni:
      cniIngressRules:
      - description: string
        fromPort: 1
        protocol: string
        toPort: 1
    securityGroupOverrides: {}
    subnets:
    - availabilityZone: string
      cidrBlock: string
      id: string
      ipv6CidrBlock: string
      isIpv6: true
      isPublic: true
      natGatewayId: string
      routeTableId: string
      tags: {}
    vpc:
      availabilityZoneSelection: string
      availabilityZoneUsageLimit: 1
      cidrBlock: string
      id: string
      internetGatewayId: string
      ipv6:
        cidrBlock: string
        egressOnlyInternetGatewayId: string
        poolId: string
      tags: {}
  region: string
  s3Bucket:
    controlPlaneIAMInstanceProfile: string
    name: string
    nodesIAMInstanceProfiles: ["string"]
  sshKeyName: string
status:
  bastion:
    addresses:
    - address: string
      type: string
    availabilityZone: string
    ebsOptimized: true
    enaSupport: true
    iamProfile: string
    id: string
    imageId: string
    instanceState: string
    networkInterfaces: ["string"]
    nonRootVolumes:
    - deviceName: string
      encrypted: true
      encryptionKey: string
      iops: 1
      size: 1
      throughput: 1
      type: string
    privateIp: string
    publicIp: string
    rootVolume:
      deviceName: string
      encrypted: true
      encryptionKey: string
      iops: 1
      size: 1
      throughput: 1
      type: string
    securityGroupIds: ["string"]
    spotMarketOptions:
      maxPrice: string
    sshKeyName: string
    subnetId: string
    tags: {}
    tenancy: string
    type: string
    userData: string
    volumeIDs: ["string"]
  conditions:
  - lastTransitionTime: string
    message: string
    reason: string
    severity: string
    status: string
    type: string
  failureDomains: {}
  networkStatus:
    apiServerElb:
      attributes:
        crossZoneLoadBalancing: true
        idleTimeout: 1
      availabilityZones: ["string"]
      dnsName: string
      healthChecks:
        healthyThreshold: 1
        interval: 1
        target: string
        timeout: 1
        unhealthyThreshold: 1
      listeners:
      - instancePort: 1
        instanceProtocol: string
        port: 1
        protocol: string
      name: string
      scheme: string
      securityGroupIds: ["string"]
      subnetIds: ["string"]
      tags: {}
    securityGroups: {}
  ready: true
```

The link to the repo is [here](https://github.com/Skarlso/crd-to-sample-yaml). Enjoy.

Feedback is welcomed.

Thanks,

Gergely.
