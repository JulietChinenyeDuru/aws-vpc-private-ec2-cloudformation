# AWS VPC Private EC2 CloudFormation Project

This project creates:
- A VPC (10.0.0.0/16)
- Two private subnets in different Availability Zones
- One EC2 instance in the first private subnet
- No internet access (no NAT or IGW)
- EC2 instance accessible only via AWS Systems Manager Session Manager

## Usage

1. Update the AMI ID in the template if needed.
2. Deploy the CloudFormation stack via AWS Console or CLI.
3. Use SSM Session Manager to connect to the EC2 instance.

## Parameters

- **AvailabilityZone1**: First AZ for Subnet 1
- **AvailabilityZone2**: Second AZ for Subnet 2
- **InstanceType**: EC2 instance type (default: t3.micro)
