# AWS Secure Private VPC — EC2 via SSM Session Manager

> **Infrastructure-as-Code** | AWS CloudFormation | Zero-Trust Network Design | No Bastion Host

[![CloudFormation](https://img.shields.io/badge/IaC-AWS%20CloudFormation-orange?logo=amazon-aws)](https://aws.amazon.com/cloudformation/)
[![Security](https://img.shields.io/badge/Access-SSM%20Session%20Manager-blue?logo=amazon-aws)](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)


---

## Overview

This project provisions a **fully private, internet-free AWS network environment** using a single CloudFormation template. An EC2 instance is deployed with **no public IP address, no SSH port open, and no internet gateway or NAT gateway** — yet remains fully manageable via **AWS Systems Manager Session Manager**.

This approach reflects modern **zero-trust security** principles: rather than exposing a bastion host or relying on key-pair SSH access, all instance management is routed through private VPC interface endpoints, reducing the attack surface to near-zero.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS VPC (10.0.0.0/16)                │
│                                                             │
│  ┌──────────────────────┐   ┌──────────────────────────┐   │
│  │  Private Subnet 1    │   │  Private Subnet 2        │   │
│  │  (AZ1) 10.0.1.0/24  │   │  (AZ2) 10.0.2.0/24      │   │
│  │                      │   │                          │   │
│  │  ┌────────────────┐  │   │  (reserved for HA        │   │
│  │  │  EC2 Instance  │  │   │   workloads / scaling)   │   │
│  │  │  t3.micro      │  │   │                          │   │
│  │  │  No public IP  │  │   └──────────────────────────┘   │
│  │  └───────┬────────┘  │                                  │
│  │          │ HTTPS     │   ┌──────────────────────────┐   │
│  │  ┌───────▼────────┐  │   │  VPC Interface Endpoints │   │
│  │  │  SSM Agent     ├──┼───►  com.amazonaws.*.ssm     │   │
│  │  └────────────────┘  │   │  com.amazonaws.*.ssmmsg  │   │
│  └──────────────────────┘   │  com.amazonaws.*.ec2msg  │   │
│                             └──────────┬─────────────── ┘   │
│  No Internet Gateway                  │                     │
│  No NAT Gateway          ┌────────────▼──────────────┐     │
│  No public subnets        │    AWS SSM Service         │     │
│                           │    (AWS-managed plane)     │     │
└───────────────────────────┴───────────────────────────┴─────┘

  Engineer connects via:
  aws ssm start-session --target <instance-id>
  (No SSH keys, no open ports, no bastion host required)
```

---

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| No Internet Gateway | Eliminates internet-facing exposure entirely |
| No NAT Gateway | Reduces cost and attack surface; SSM endpoints handle all outbound |
| SSM over SSH | No port 22, no key management, full audit trail via CloudTrail |
| VPC Interface Endpoints | All SSM traffic stays within the AWS private network |
| IAM Instance Profile | Least-privilege: only `AmazonSSMManagedInstanceCore` policy attached |
| Dynamic AMI via SSM Parameter | Always resolves to latest Amazon Linux 2 AMI — no hardcoded IDs |
| Multi-AZ private subnets | High-availability foundation for future workload expansion |
| Parameterised CIDR blocks | Reusable across environments without template modification |

---

## Resources Provisioned

| Resource | Type | Purpose |
|----------|------|---------|
| VPC | `AWS::EC2::VPC` | Isolated network (10.0.0.0/16) |
| Private Subnet 1 | `AWS::EC2::Subnet` | EC2 host subnet (AZ1) |
| Private Subnet 2 | `AWS::EC2::Subnet` | HA / expansion subnet (AZ2) |
| Route Table | `AWS::EC2::RouteTable` | Private routing (no 0.0.0.0/0 route) |
| EC2 Security Group | `AWS::EC2::SecurityGroup` | No inbound rules; HTTPS egress only |
| Endpoint Security Group | `AWS::EC2::SecurityGroup` | Restricts VPC endpoint access to VPC CIDR |
| SSM VPC Endpoint | `AWS::EC2::VPCEndpoint` | Private path to SSM service |
| SSMMessages VPC Endpoint | `AWS::EC2::VPCEndpoint` | Session Manager communication |
| EC2Messages VPC Endpoint | `AWS::EC2::VPCEndpoint` | EC2 ↔ SSM message relay |
| IAM Role | `AWS::IAM::Role` | EC2 instance identity for SSM |
| IAM Instance Profile | `AWS::IAM::InstanceProfile` | Attaches IAM role to EC2 |
| EC2 Instance | `AWS::EC2::Instance` | Private compute, SSM-managed |

---

## Prerequisites

- AWS CLI installed and configured (`aws configure`)
- Sufficient IAM permissions to create VPC, EC2, IAM, and CloudFormation resources
- AWS Session Manager Plugin installed locally for terminal access:
  ```bash
  # macOS
  brew install --cask session-manager-plugin

  # Linux
  curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
  sudo dpkg -i session-manager-plugin.deb
  ```

---

## Deployment

### Option 1 — AWS Console

1. Navigate to **CloudFormation → Create Stack → With new resources**
2. Upload `template.yaml`
3. Fill in parameters (AZs, CIDR blocks, instance type)
4. Acknowledge IAM resource creation
5. Click **Create Stack**

### Option 2 — AWS CLI

```bash
aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name private-vpc-ssm-demo \
  --parameter-overrides \
      ProjectName=private-vpc-demo \
      Environment=dev \
      AvailabilityZone1=eu-west-1a \
      AvailabilityZone2=eu-west-1b \
      InstanceType=t3.micro \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-west-1
```

### Option 3 — From Jupyter Notebook (boto3)

See [`vpc-ec2-ssm-deployment.ipynb`](vpc-ec2-ssm-deployment.ipynb) for a step-by-step deployment walkthrough using the AWS Python SDK (boto3), including stack creation, status polling, and output retrieval.

---

## Connecting to the EC2 Instance

Once the stack is deployed, connect without SSH or a bastion host:

```bash
# Retrieve instance ID from stack outputs
INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name private-vpc-ssm-demo \
  --query "Stacks[0].Outputs[?OutputKey=='EC2InstanceId'].OutputValue" \
  --output text)

# Start a Session Manager session
aws ssm start-session --target $INSTANCE_ID --region eu-west-1
```

You will get a shell directly on the private EC2 instance — no SSH keys, no open ports.

---

## Parameters Reference

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ProjectName` | `private-vpc-demo` | Prefix applied to all resource names |
| `Environment` | `dev` | Deployment environment (`dev`, `staging`, `prod`) |
| `VpcCidr` | `10.0.0.0/16` | VPC IP address range |
| `PrivateSubnet1Cidr` | `10.0.1.0/24` | Subnet 1 CIDR (AZ1) |
| `PrivateSubnet2Cidr` | `10.0.2.0/24` | Subnet 2 CIDR (AZ2) |
| `AvailabilityZone1` | *(required)* | First AZ (e.g. `eu-west-1a`) |
| `AvailabilityZone2` | *(required)* | Second AZ (must differ from AZ1) |
| `InstanceType` | `t3.micro` | EC2 instance size |
| `AmiId` | Latest Amazon Linux 2 | Resolves dynamically via SSM Parameter Store |

---

## Stack Outputs

After deployment, the stack exports:

| Output | Description |
|--------|-------------|
| `VPCId` | VPC resource ID |
| `PrivateSubnet1Id` | Subnet 1 ID (importable by other stacks) |
| `PrivateSubnet2Id` | Subnet 2 ID |
| `EC2InstanceId` | Instance ID for SSM connection |
| `SSMConnectCommand` | Ready-to-run CLI command for session access |
| `EC2SSMRoleArn` | IAM Role ARN |

---

## Teardown

To avoid ongoing charges, delete the stack when finished:

```bash
aws cloudformation delete-stack \
  --stack-name private-vpc-ssm-demo \
  --region eu-west-1
```

---

## Security Posture Summary

This project intentionally eliminates common cloud security risks:

- **No SSH exposure** — port 22 is never opened; no key pairs required
- **No public IP** — instance cannot be reached from the internet
- **No internet gateway** — the VPC has no outbound internet path
- **No NAT gateway** — removes a common lateral movement vector
- **Full audit trail** — all SSM sessions are logged via AWS CloudTrail
- **Least-privilege IAM** — only the minimum managed policy for SSM is attached
- **Private DNS resolution** — VPC endpoints use private DNS to prevent traffic leaving AWS network

---

## Skills Demonstrated

- AWS CloudFormation (Infrastructure-as-Code)
- VPC networking (subnets, route tables, security groups)
- AWS Systems Manager Session Manager
- VPC Interface Endpoints (PrivateLink)
- IAM roles and instance profiles
- Zero-trust access design
- Multi-AZ high-availability architecture
- AWS CLI and boto3 (Python SDK)

---

## Author

**Juliet Chinenye Duru**
DevOps / Cloud Engineer | Agentic AI Engineer | NLP Researcher
[GitHub](https://github.com/JulietChinenyeDuru) · [ORCID: 0009-0002-0530-8082](https://orcid.org/0009-0002-0530-8082)

---


