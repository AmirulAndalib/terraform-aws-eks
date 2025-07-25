# Frequently Asked Questions

- [Setting `disk_size` or `remote_access` does not make any changes](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#Settings-disk_size-or-remote_access-does-not-make-any-changes)
- [I received an error: `expect exactly one securityGroup tagged with kubernetes.io/cluster/<NAME> ...`](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#i-received-an-error-expect-exactly-one-securitygroup-tagged-with-kubernetesioclustername-)
- [Why are nodes not being registered?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#why-are-nodes-not-being-registered)
- [Why are there no changes when a node group's `desired_size` is modified?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#why-are-there-no-changes-when-a-node-groups-desired_size-is-modified)
- [How do I access compute resource attributes?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#how-do-i-access-compute-resource-attributes)
- [What add-ons are available?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#what-add-ons-are-available)
- [What configuration values are available for an add-on?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#what-configuration-values-are-available-for-an-add-on)

### Setting `disk_size` or `remote_access` does not make any changes

`disk_size`, and `remote_access` can only be set when using the EKS managed node group default launch template. This module defaults to providing a custom launch template to allow for custom security groups, tag propagation, etc. If you wish to forgo the custom launch template route, you can set `use_custom_launch_template = false` and then you can set `disk_size` and `remote_access`.

### I received an error: `expect exactly one securityGroup tagged with kubernetes.io/cluster/<CLUSTER_NAME> ...`

⚠️ `<CLUSTER_NAME>` would be the name of your cluster

By default, EKS creates a cluster primary security group that is created outside of the module and the EKS service adds the tag `{ "kubernetes.io/cluster/<CLUSTER_NAME>" = "owned" }`. This on its own does not cause any conflicts for addons such as the AWS Load Balancer Controller until users decide to attach both the cluster primary security group and the shared node security group created by the module (by setting `attach_cluster_primary_security_group = true`). The issue is not with having multiple security groups in your account with this tag key:value combination, but having multiple security groups with this tag key:value combination attached to nodes in the same cluster. There are a few ways to resolve this depending on your use case/intentions:

1. If you want to use the cluster primary security group, you can disable the creation of the shared node security group with:

```hcl
  create_node_security_group = false # default is true

  eks_managed_node_group = {
    example = {
      attach_cluster_primary_security_group = true # default is false
    }
  }
  # Or for self-managed
  self_managed_node_group = {
    example = {
      attach_cluster_primary_security_group = true # default is false
    }
  }
```

2. By not attaching the cluster primary security group. The cluster primary security group has quite broad access and the module has instead provided a security group with the minimum amount of access to launch an empty EKS cluster successfully and users are encouraged to open up access when necessary to support their workload.

```hcl
  eks_managed_node_group = {
    example = {
      attach_cluster_primary_security_group = true # default is false
    }
  }
  # Or for self-managed
  self_managed_node_group = {
    example = {
      attach_cluster_primary_security_group = true # default is false
    }
  }
```

In theory, if you are attaching the cluster primary security group, you shouldn't need to use the shared node security group created by the module. However, this is left up to users to decide for their requirements and use case.

If you choose to use [Custom Networking](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html), make sure to only attach the security groups matching your choice above in your ENIConfig resources. This will ensure you avoid redundant tags.

### Why are nodes not being registered?

Nodes not being able to register with the EKS control plane is generally due to networking mis-configurations.

1. At least one of the cluster endpoints (public or private) must be enabled.

If you require a public endpoint, setting up both (public and private) and restricting the public endpoint via setting `cluster_endpoint_public_access_cidrs` is recommended. More info regarding communication with an endpoint is available [here](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html).

2. Nodes need to be able to contact the EKS cluster endpoint. By default, the module only creates a public endpoint. To access the endpoint, the nodes need outgoing internet access:

- Nodes in private subnets: via a NAT gateway or instance along with the appropriate routing rules
- Nodes in public subnets: ensure that nodes are launched with public IPs (enable through either the module here or your subnet setting defaults)

**Important: If you apply only the public endpoint and configure the `cluster_endpoint_public_access_cidrs` to restrict access, know that EKS nodes will also use the public endpoint and you must allow access to the endpoint. If not, then your nodes will fail to work correctly.**

3. The private endpoint can also be enabled by setting `cluster_endpoint_private_access = true`. Ensure that VPC DNS resolution and hostnames are also enabled for your VPC when the private endpoint is enabled.

4. Nodes need to be able to connect to other AWS services to function (download container images, make API calls to assume roles, etc.). If for some reason you cannot enable public internet access for nodes you can add VPC endpoints to the relevant services: EC2 API, ECR API, ECR DKR and S3.

### Why are there no changes when a node group's `desired_size` is modified?

The module is configured to ignore this value. Unfortunately, Terraform does not support variables within the `lifecycle` block. The setting is ignored to allow autoscaling via controllers such as cluster autoscaler or Karpenter to work properly and without interference by Terraform. Changing the desired count must be handled outside of Terraform once the node group is created.

:info: See [this](https://github.com/bryantbiggs/eks-desired-size-hack) for a workaround to this limitation.

### How do I access compute resource attributes?

Examples of accessing the attributes of the compute resource(s) created by the root module are shown below. Note - the assumption is that your cluster module definition is named `eks` as in `module "eks" { ... }`:

- EKS Managed Node Group attributes

```hcl
eks_managed_role_arns = [for group in module.eks_managed_node_group : group.iam_role_arn]
```

- Self Managed Node Group attributes

```hcl
self_managed_role_arns = [for group in module.self_managed_node_group : group.iam_role_arn]
```

- Fargate Profile attributes

```hcl
fargate_profile_pod_execution_role_arns = [for group in module.fargate_profile : group.fargate_profile_pod_execution_role_arn]
```

### What add-ons are available?

The available EKS add-ons can be [found here](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html). You can also retrieve the available addons from the API using:

```sh
aws eks describe-addon-versions --query 'addons[*].addonName'
```

### What configuration values are available for an add-on?

> [!NOTE]
> The available configuration values will vary between add-on versions,
> typically more configuration values will be added in later versions as functionality is enabled by EKS.

You can retrieve the configuration value schema for a given addon using the following command:

```sh
aws eks describe-addon-configuration --addon-name <value> --addon-version <value> --query 'configurationSchema' --output text | jq
```

For example:

```sh
aws eks describe-addon-configuration --addon-name coredns --addon-version v1.11.1-eksbuild.8 --query 'configurationSchema' --output text | jq
```

Returns (at the time of writing):

```json
{
  "$ref": "#/definitions/Coredns",
  "$schema": "http://json-schema.org/draft-06/schema#",
  "definitions": {
    "Coredns": {
      "additionalProperties": false,
      "properties": {
        "affinity": {
          "default": {
            "affinity": {
              "nodeAffinity": {
                "requiredDuringSchedulingIgnoredDuringExecution": {
                  "nodeSelectorTerms": [
                    {
                      "matchExpressions": [
                        {
                          "key": "kubernetes.io/os",
                          "operator": "In",
                          "values": [
                            "linux"
                          ]
                        },
                        {
                          "key": "kubernetes.io/arch",
                          "operator": "In",
                          "values": [
                            "amd64",
                            "arm64"
                          ]
                        }
                      ]
                    }
                  ]
                }
              },
              "podAntiAffinity": {
                "preferredDuringSchedulingIgnoredDuringExecution": [
                  {
                    "podAffinityTerm": {
                      "labelSelector": {
                        "matchExpressions": [
                          {
                            "key": "k8s-app",
                            "operator": "In",
                            "values": [
                              "kube-dns"
                            ]
                          }
                        ]
                      },
                      "topologyKey": "kubernetes.io/hostname"
                    },
                    "weight": 100
                  }
                ]
              }
            }
          },
          "description": "Affinity of the coredns pods",
          "type": [
            "object",
            "null"
          ]
        },
        "computeType": {
          "type": "string"
        },
        "corefile": {
          "description": "Entire corefile contents to use with installation",
          "type": "string"
        },
        "nodeSelector": {
          "additionalProperties": {
            "type": "string"
          },
          "type": "object"
        },
        "podAnnotations": {
          "properties": {},
          "title": "The podAnnotations Schema",
          "type": "object"
        },
        "podDisruptionBudget": {
          "description": "podDisruptionBudget configurations",
          "enabled": {
            "default": true,
            "description": "the option to enable managed PDB",
            "type": "boolean"
          },
          "maxUnavailable": {
            "anyOf": [
              {
                "pattern": ".*%$",
                "type": "string"
              },
              {
                "type": "integer"
              }
            ],
            "default": 1,
            "description": "minAvailable value for managed PDB, can be either string or integer; if it's string, should end with %"
          },
          "minAvailable": {
            "anyOf": [
              {
                "pattern": ".*%$",
                "type": "string"
              },
              {
                "type": "integer"
              }
            ],
            "description": "maxUnavailable value for managed PDB, can be either string or integer; if it's string, should end with %"
          },
          "type": "object"
        },
        "podLabels": {
          "properties": {},
          "title": "The podLabels Schema",
          "type": "object"
        },
        "replicaCount": {
          "type": "integer"
        },
        "resources": {
          "$ref": "#/definitions/Resources"
        },
        "tolerations": {
          "default": [
            {
              "key": "CriticalAddonsOnly",
              "operator": "Exists"
            },
            {
              "effect": "NoSchedule",
              "key": "node-role.kubernetes.io/control-plane"
            }
          ],
          "description": "Tolerations of the coredns pod",
          "items": {
            "type": "object"
          },
          "type": "array"
        },
        "topologySpreadConstraints": {
          "description": "The coredns pod topology spread constraints",
          "type": "array"
        }
      },
      "title": "Coredns",
      "type": "object"
    },
    "Limits": {
      "additionalProperties": false,
      "properties": {
        "cpu": {
          "type": "string"
        },
        "memory": {
          "type": "string"
        }
      },
      "title": "Limits",
      "type": "object"
    },
    "Resources": {
      "additionalProperties": false,
      "properties": {
        "limits": {
          "$ref": "#/definitions/Limits"
        },
        "requests": {
          "$ref": "#/definitions/Limits"
        }
      },
      "title": "Resources",
      "type": "object"
    }
  }
}
```
