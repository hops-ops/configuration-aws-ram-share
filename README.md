# aws-ram-share

Share AWS resources across accounts using AWS Resource Access Manager (RAM).

## Why RAM Sharing?

**Without RAM sharing:**
- Each account must own its own resources (duplicate IPAM pools, Transit Gateways, etc.)
- IP address conflicts when teams manually coordinate CIDR ranges
- No central visibility into cross-account resource usage
- Complex IAM role chaining for cross-account access

**With RAM sharing:**
- Centralized resources shared to OUs or specific accounts
- IPAM pools enable automatic, conflict-free IP allocation across accounts
- Transit Gateway hub-spoke architecture with single TGW
- Private CA shared for consistent certificate issuance
- Clear audit trail of what's shared with whom

# Prerequisites

Set up RAM sharing with: 

```
aws ram enable-sharing-with-aws-organization
```

## The Journey

### Stage 1: Getting Started (Single Share)

Share a single resource with one account - the simplest pattern.

```yaml
apiVersion: aws.hops.ops.com.ai/v1alpha1
kind: RAMShare
metadata:
  name: minimal-share
spec:
  region: us-east-1
  name: minimal-share

  resources:
    - name: my-pool
      arn: arn:aws:ec2::123456789012:ipam-pool/ipam-pool-0abc123

  principals:
    - name: dev-account
      type: account
      id: "999999999999"
```

### Stage 2: Growing (OU-Based Sharing)

As your organization grows, share resources with entire Organizational Units.

```yaml
apiVersion: aws.hops.ops.com.ai/v1alpha1
kind: RAMShare
metadata:
  name: ipam-pools-shared
spec:
  region: us-east-1
  name: ipam-pools-platform
  allowExternalPrincipals: false

  resources:
    - name: ipv4-us-east-1
      arn: arn:aws:ec2::123456789012:ipam-pool/ipam-pool-0abc123def456789a
    - name: ipv4-us-west-2
      arn: arn:aws:ec2::123456789012:ipam-pool/ipam-pool-0def456abc789012b

  principals:
    - name: teams-ou
      type: organizationalUnit
      arn: arn:aws:organizations::123456789012:ou/o-abc123def4/ou-root-teams

  tags:
    environment: platform
```

### Stage 3: Enterprise Scale (Multiple Resource Types)

Full enterprise pattern with centralized networking resources.

**Transit Gateway sharing:**
```yaml
apiVersion: aws.hops.ops.com.ai/v1alpha1
kind: RAMShare
metadata:
  name: central-transit-gateway
spec:
  region: us-east-1
  name: central-transit-gateway
  allowExternalPrincipals: false

  resources:
    - name: tgw
      arn: arn:aws:ec2:us-east-1:222222222222:transit-gateway/tgw-0abc123def456789a

  principals:
    - name: workloads-ou
      type: organizationalUnit
      arn: arn:aws:organizations::111111111111:ou/o-abc123def4/ou-root-workloads
```

**Private CA sharing:**
```yaml
apiVersion: aws.hops.ops.com.ai/v1alpha1
kind: RAMShare
metadata:
  name: internal-pki
spec:
  region: us-east-1
  name: internal-certificate-authority
  allowExternalPrincipals: false

  resources:
    - name: root-ca
      arn: arn:aws:acm-pca:us-east-1:333333333333:certificate-authority/abc123-def456

  principals:
    - name: entire-org
      type: organization
      arn: arn:aws:organizations::111111111111:organization/o-abc123def4
```

## Using RAMShare

The RAMShare XRD outputs status that can be used for monitoring:

```yaml
status:
  ready: true
  shareArn: arn:aws:ram:us-east-1:123456789012:resource-share/abc-123
  shareId: abc-123-def-456
  resources:
    - name: ipv4-us-east-1
      arn: arn:aws:ec2::123456789012:ipam-pool/ipam-pool-0abc123
      ready: true
  principals:
    - name: teams-ou
      principal: arn:aws:organizations::123456789012:ou/o-abc123def4/ou-root-teams
      ready: true
```

## Shareable Resource Types

RAM supports 60+ resource types including:

| Category | Resources |
|----------|-----------|
| **Networking** | IPAM Pools, Transit Gateways, Subnets, Prefix Lists, Resolver Rules |
| **Security** | Private CA, Network Firewall Policies, Route 53 Firewall Rules |
| **Compute** | Capacity Reservations, Dedicated Hosts, EC2 Image Builder |
| **Data** | Aurora Clusters, Glue Catalogs, SageMaker Models |

## Status

| Field | Description |
|-------|-------------|
| `ready` | True when all resources are shared and principals associated |
| `shareArn` | ARN of the RAM resource share |
| `shareId` | ID of the RAM resource share |
| `resources` | Status of each resource association |
| `principals` | Status of each principal association |

## Composed Resources

- `ResourceShare` - The RAM share itself
- `ResourceAssociation` - One per resource ARN being shared
- `PrincipalAssociation` - One per principal (account, OU, or organization)

## Development

```bash
make render          # Render all examples
make validate        # Validate all examples
make test            # Run KCL tests
make render:minimal  # Render single example
```

## License

Apache-2.0
