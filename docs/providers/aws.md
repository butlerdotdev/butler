# AWS Provider Guide

> **Status: In Progress.** Cloud provider support is in active development. The CRD types, bootstrap controller integration, and CLI commands are implemented. The provider controller (`butler-provider-aws`) is not yet released. This guide describes the target architecture.

This guide covers setting up Butler with Amazon Web Services as the infrastructure provider for management cluster bootstrap.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [AWS Setup](#aws-setup)
- [Butler Configuration](#butler-configuration)
- [Network Architecture](#network-architecture)
- [Load Balancer Resources](#load-balancer-resources)
- [Resource Recommendations](#resource-recommendations)
- [Troubleshooting](#troubleshooting)

---

## Overview

Butler uses a thin provider controller (`butler-provider-aws`) to provision EC2 instances running Talos Linux. The controller creates instances in the specified VPC/subnet and reports their IPs back to the bootstrap controller.

For control plane HA, the bootstrap controller creates a LoadBalancerRequest, and the AWS provider provisions a Network Load Balancer (NLB) with a TCP target group and listener on port 6443.

```mermaid
flowchart LR
    CLI["butleradm"] --> KIND["KIND Cluster"]
    KIND --> Bootstrap["butler-bootstrap"]
    KIND --> AWSProvider["butler-provider-aws"]
    Bootstrap --> MR["MachineRequest CRs"]
    Bootstrap --> LBR["LoadBalancerRequest CR"]
    AWSProvider --> EC2["EC2 Instances"]
    AWSProvider --> NLB["Network Load Balancer"]
```

### Key Components

| Component | Purpose |
|-----------|---------|
| butler-provider-aws | Provisions EC2 instances and NLB resources |
| CAPI AWS Provider (CAPA) | Manages tenant cluster worker instances after bootstrap |

---

## Prerequisites

### AWS Account

- An AWS account with access to the target region
- EC2 and Elastic Load Balancing APIs enabled

### IAM User or Role

An IAM user (or role) with permissions for:
- EC2: instances, security groups, key pairs, AMIs, EBS volumes
- Elastic Load Balancing v2: NLBs, target groups, listeners
- VPC: subnets, route tables (read-only)

A suitable managed policy combination:
- `AmazonEC2FullAccess`
- `ElasticLoadBalancingFullAccess`

Or use a custom IAM policy scoped to the required resources.

### Networking

- A VPC with at least one public subnet (instances need reachability from the bootstrap machine)
- A security group allowing required ports (see [Network Architecture](#network-architecture))
- Sufficient Elastic IP quota if using static public IPs

### Talos AMI

A Talos Linux AMI must be available in the target region. Import from the official Talos releases or use the Talos Image Factory to produce a custom AMI.

---

## AWS Setup

### 1. Create IAM User

```bash
# Create IAM user
aws iam create-user --user-name butler-bootstrap

# Attach required policies
aws iam attach-user-policy --user-name butler-bootstrap \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam attach-user-policy --user-name butler-bootstrap \
  --policy-arn arn:aws:iam::aws:policy/ElasticLoadBalancingFullAccess

# Create access key
aws iam create-access-key --user-name butler-bootstrap
```

Save the `AccessKeyId` and `SecretAccessKey` from the output.

### 2. Create Security Group

```bash
# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name butler-bootstrap \
  --description "Butler bootstrap cluster" \
  --vpc-id vpc-XXXXX \
  --query 'GroupId' --output text)

# Allow inter-node traffic (self-referencing)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 6443 --source-group $SG_ID
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 50000-50001 --source-group $SG_ID
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 2379-2380 --source-group $SG_ID
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 10250 --source-group $SG_ID

# Allow kube-apiserver access from anywhere (for LB and external access)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 6443 --cidr 0.0.0.0/0

# Allow SSH (optional, for debugging)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 22 --cidr YOUR_IP/32
```

### 3. Import Talos AMI (if needed)

```bash
# Download Talos AWS image
wget https://factory.talos.dev/image/SCHEMATIC_ID/vX.Y.Z/aws-amd64.raw.xz

# Upload to S3 and import as AMI
aws s3 cp talos-vX.Y.Z.raw.xz s3://BUCKET/talos-vX.Y.Z.raw.xz

aws ec2 import-image \
  --disk-containers "Description=Talos vX.Y.Z,Format=raw,Url=s3://BUCKET/talos-vX.Y.Z.raw.xz"
```

---

## Butler Configuration

### ProviderConfig

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: butler-system
type: Opaque
stringData:
  access-key-id: "AKIAIOSFODNN7EXAMPLE"
  secret-access-key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
---
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ProviderConfig
metadata:
  name: aws-prod
  namespace: butler-system
spec:
  provider: aws
  credentialsRef:
    name: aws-credentials
    namespace: butler-system
  network:
    mode: cloud
  aws:
    region: us-east-1
    vpcID: vpc-0abcdef1234567890
    subnetIDs:
      - subnet-0abcdef1234567890
    securityGroupIDs:
      - sg-0abcdef1234567890
```

### Bootstrap Config

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: ClusterBootstrap
metadata:
  name: butler-mgmt
  namespace: butler-system
spec:
  provider: aws
  providerRef:
    name: aws-prod
    namespace: butler-system

  cluster:
    name: butler-mgmt
    topology: ha
    controlPlane:
      replicas: 3
      cpu: 4
      memoryMB: 16384
      diskGB: 100
    workers:
      replicas: 3
      cpu: 8
      memoryMB: 32768
      diskGB: 200

  network:
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/12

  talos:
    version: v1.9.2
    schematic: ce4c980550dd2ab1b17bbf2b08801c7eb59418eafe8f279833297925d67c7515

  addons:
    cni:
      type: cilium
    storage:
      type: longhorn
    controlPlaneProvider:
      type: steward
    capi:
      enabled: true
      infrastructureProviders:
        - name: aws

  controlPlaneExposure:
    mode: LoadBalancer
```

### Run Bootstrap

```bash
butleradm bootstrap aws --config bootstrap.yaml
```

---

## Network Architecture

```mermaid
flowchart TB
    subgraph AWS["AWS Account"]
        subgraph VPC["VPC"]
            subgraph Subnet["Public Subnet (us-east-1a)"]
                CP0["CP-0<br/>10.0.1.10"]
                CP1["CP-1<br/>10.0.1.11"]
                CP2["CP-2<br/>10.0.1.12"]
                W0["Worker-0<br/>10.0.1.20"]
                W1["Worker-1<br/>10.0.1.21"]
                W2["Worker-2<br/>10.0.1.22"]
            end
            SG["Security Group"]
        end
        NLB["Network Load Balancer<br/>butler-mgmt-nlb-xxx.elb.amazonaws.com:6443"]
    end

    Internet["External Access"] --> NLB
    NLB --> CP0
    NLB --> CP1
    NLB --> CP2
    SG -.-> Subnet
```

### Security Group Rules

| Direction | Protocol/Port | Source/Destination | Purpose |
|-----------|--------------|-------------------|---------|
| Inbound | TCP 6443 | 0.0.0.0/0 | Kubernetes API (external + NLB health checks) |
| Inbound | TCP 50000-50001 | Self | Talos API (apid + trustd) |
| Inbound | TCP 2379-2380 | Self | etcd client and peer |
| Inbound | TCP 10250 | Self | kubelet API |

NLB health checks originate from within the VPC, so the `0.0.0.0/0` rule on port 6443 covers both external access and health check traffic.

---

## Load Balancer Resources

When the bootstrap controller creates a LoadBalancerRequest with `provider: aws`, the AWS provider controller creates:

| AWS Resource | Purpose |
|--------------|---------|
| Network Load Balancer | Regional L4 load balancer in the VPC |
| Target group (TCP 6443) | Groups the control plane EC2 instances |
| Listener (TCP 6443) | Routes traffic from NLB to the target group |

The NLB uses TCP passthrough. The NLB DNS name (e.g., `butler-mgmt-nlb-xxx.elb.us-east-1.amazonaws.com`) is reported as the LoadBalancerRequest endpoint and used as the control plane address.

Target registration uses instance IDs (`LoadBalancerTarget.instanceID`).

---

## Resource Recommendations

| Profile | CP Nodes | CP Instance Type | Worker Nodes | Worker Instance Type | Estimated Monthly Cost |
|---------|----------|-----------------|-------------|---------------------|----------------------|
| Development | 1 (single-node) | m6i.xlarge (4 vCPU, 16 GB) | 0 | -- | ~$150 |
| Staging | 3 | m6i.xlarge (4 vCPU, 16 GB) | 2 | m6i.2xlarge (8 vCPU, 32 GB) | ~$950 |
| Production | 3 | m6i.xlarge (4 vCPU, 16 GB) | 3+ | m6i.2xlarge (8 vCPU, 32 GB) | ~$1,300+ |

NLB pricing: NLB charges per hour plus per LCU (Load Balancer Capacity Unit). For a management cluster with low traffic, expect approximately $20/month.

---

## Troubleshooting

### Security Group Rules Missing

**Symptom**: Talos bootstrap times out. Nodes cannot communicate on required ports.

**Verify**:
```bash
aws ec2 describe-security-groups --group-ids sg-XXXXX \
  --query "SecurityGroups[0].IpPermissions"
```

### IAM Permissions Insufficient

**Symptom**: Provider controller logs show `UnauthorizedOperation` errors.

**Verify**:
```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT:user/butler-bootstrap \
  --action-names ec2:RunInstances elasticloadbalancing:CreateLoadBalancer
```

### NLB Target Health Failures

**Symptom**: LoadBalancerRequest stuck in `Creating` phase. Target group shows unhealthy targets.

**Verify**:
```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:REGION:ACCOUNT:targetgroup/butler-mgmt/XXXXX
```

**Common causes:**
- Security group does not allow TCP 6443 from VPC CIDR
- kube-apiserver not yet listening (bootstrap still in progress)
- Instance not registered in the correct target group

### Subnet Not Public

**Symptom**: Instances created but not reachable from the bootstrap machine.

Instances need a route to the internet (via IGW) for the KIND cluster to reach the Talos API during bootstrap. Use a public subnet with auto-assign public IP, or set up a bastion/VPN.

---

## See Also

- [Bootstrap Flow](../architecture/bootstrap-flow.md) - end-to-end bootstrap sequence
- [ClusterBootstrap CRD](../reference/crds/clusterbootstrap.md) - full spec reference
- [LoadBalancerRequest CRD](../reference/crds/loadbalancerrequest.md) - cloud LB provisioning
- [ProviderConfig CRD](../reference/crds/providerconfig.md) - provider credentials
