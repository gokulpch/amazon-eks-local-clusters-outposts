# CloudFormation Templates for Amazon EKS Local Clusters on AWS Outposts

This repository contains AWS CloudFromation (native AWS IaaC) templates for creating Amazon EKS Local Clusters on AWS Outposts

Two CFN Stacks:
1. Creates Networking required for creating fully-private Amazon EKS Local Clusters on AWS Outposts
2. Create a Amazon EKS - Local Clusters on AWS Outposts using the networking environment created above (users should choose the VPC and Subnet from the above stack as parameters when creationg a cluster)

## Components created by Networking CFN Template (network-infra.yaml)

A comprehensive list of all components created by the CloudFormation template:

1. **VPC**
   - Type: `AWS::EC2::VPC`
   - CIDR: 192.168.0.0/16
   - Properties: EnableDnsHostnames, EnableDnsSupport

2. **Cluster Shared Node Security Group**
   - Type: `AWS::EC2::SecurityGroup`
   - Purpose: Communication between all nodes in the cluster
   - VPC: References the VPC created in this stack

3. **Private Subnet**
   - Type: `AWS::EC2::Subnet`
   - Name: SubnetPrivateUSWEST2A
   - CIDR: 192.168.32.0/19
   - Availability Zone: us-west-2a
   - OutpostArn: arn:aws:outposts:us-west-2:576319625758:outpost/op-0268f76782a30c66a
   - VPC: References the VPC created in this stack

4. **Private Route Table**
   - Type: `AWS::EC2::RouteTable`
   - Name: PrivateRouteTableUSWEST2A
   - VPC: References the VPC created in this stack

5. **Route Table Association**
   - Type: `AWS::EC2::SubnetRouteTableAssociation`
   - Associates the Private Route Table with the Private Subnet

6. **VPC Endpoint for EC2**
   - Type: `AWS::EC2::VPCEndpoint`
   - Service Name: com.amazonaws.[Region].ec2
   - Type: Interface
   - Private DNS Enabled: true
   - Security Group: References the Cluster Shared Node Security Group
   - Subnet: References the Private Subnet
   - VPC: References the VPC created in this stack

7. **VPC Endpoint for EC2 Messages**
   - Type: `AWS::EC2::VPCEndpoint`
   - Service Name: com.amazonaws.[Region].ec2messages
   - Type: Interface
   - Private DNS Enabled: true
   - Security Group: References the Cluster Shared Node Security Group
   - Subnet: References the Private Subnet
   - VPC: References the VPC created in this stack

8. **VPC Endpoint for ECR API**
   - Type: `AWS::EC2::VPCEndpoint`
   - Service Name: com.amazonaws.[Region].ecr.api
   - Type: Interface
   - Private DNS Enabled: true
   - Security Group: References the Cluster Shared Node Security Group
   - Subnet: References the Private Subnet
   - VPC: References the VPC created in this stack

9. **VPC Endpoint for ECR DKR**
   - Type: `AWS::EC2::VPCEndpoint`
   - Service Name: com.amazonaws.[Region].ecr.dkr
   - Type: Interface
   - Private DNS Enabled: true
   - Security Group: References the Cluster Shared Node Security Group
   - Subnet: References the Private Subnet
   - VPC: References the VPC created in this stack

10. **VPC Endpoint for S3**
    - Type: `AWS::EC2::VPCEndpoint`
    - Service Name: com.amazonaws.[Region].s3
    - Type: Gateway
    - Route Table: References the Private Route Table
    - VPC: References the VPC created in this stack

11. **VPC Endpoint for Secrets Manager**
    - Type: `AWS::EC2::VPCEndpoint`
    - Service Name: com.amazonaws.[Region].secretsmanager
    - Type: Interface
    - Private DNS Enabled: true
    - Security Group: References the Cluster Shared Node Security Group
    - Subnet: References the Private Subnet
    - VPC: References the VPC created in this stack

12. **VPC Endpoint for SSM**
    - Type: `AWS::EC2::VPCEndpoint`
    - Service Name: com.amazonaws.[Region].ssm
    - Type: Interface
    - Private DNS Enabled: true
    - Security Group: References the Cluster Shared Node Security Group
    - Subnet: References the Private Subnet
    - VPC: References the VPC created in this stack

13. **VPC Endpoint for SSM Messages**
    - Type: `AWS::EC2::VPCEndpoint`
    - Service Name: com.amazonaws.[Region].ssmmessages
    - Type: Interface
    - Private DNS Enabled: true
    - Security Group: References the Cluster Shared Node Security Group
    - Subnet: References the Private Subnet
    - VPC: References the VPC created in this stack

14. **VPC Endpoint for STS**
    - Type: `AWS::EC2::VPCEndpoint`
    - Service Name: com.amazonaws.[Region].sts
    - Type: Interface
    - Private DNS Enabled: true
    - Security Group: References the Cluster Shared Node Security Group
    - Subnet: References the Private Subnet
    - VPC: References the VPC created in this stack

15. **Security Group Ingress Rule for Private Subnet**
    - Type: `AWS::EC2::SecurityGroupIngress`
    - Purpose: Allow private subnets to communicate with VPC endpoints
    - Protocol: TCP
    - Port Range: 443
    - CIDR: References the Private Subnet CIDR
    - Security Group: References the Cluster Shared Node Security Group

16. **Security Group Ingress Rule for Inter-Node Communication**
    - Type: `AWS::EC2::SecurityGroupIngress`
    - Purpose: Allow nodes to communicate with each other
    - Protocol: All (-1)
    - Port Range: 0-65535
    - Source: Self-referencing the Cluster Shared Node Security Group
    - Security Group: References the Cluster Shared Node Security Group

17. **CloudFormation Outputs**
    - VPC: Exports the VPC ID
    - SharedNodeSecurityGroup: Exports the Security Group ID
    - SubnetsPrivate: Exports the Private Subnet ID
    - VpcCidr: Exports the VPC CIDR block
    - PrivateSubnetCidr: Exports the Private Subnet CIDR block

All of these components work together to create a secure, private networking infrastructure for an EKS cluster running on AWS Outpost with private connectivity to AWS services.

# AWS CLI Commands to Create Each Component in the CloudFormation Template [if not using the stack above]

**Note**: To run these commands successfully, you'll need to:

1. Replace placeholder IDs (vpc-xxxxxxxxxxxxxxxxx, subnet-xxxxxxxxxxxxxxxxx, etc.) with actual IDs returned from previous commands
2. Set appropriate AWS credentials and region through environment variables or AWS CLI configuration
3. Run each command in the correct order to maintain dependencies
4. Ensure you have the necessary permissions to create each resource

While this approach works, managing these resources with CloudFormation is generally recommended as it handles dependencies and rollbacks automatically.

Here are the AWS CLI commands to create each component defined in the CloudFormation template individually:

## 1. Create VPC

```bash
aws ec2 create-vpc \
  --cidr-block 192.168.0.0/16 \
  --enable-dns-support \
  --enable-dns-hostnames \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=EKSNetworkingVPC}]' \
  --region us-west-2
```

## 2. Create Cluster Shared Node Security Group

```bash
# First, you need the VPC ID from the previous command
vpc_id="vpc-xxxxxxxxxxxxxxxxx"

aws ec2 create-security-group \
  --group-name ClusterSharedNodeSecurityGroup \
  --description "Communication between all nodes in the cluster" \
  --vpc-id $vpc_id \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=ClusterSharedNodeSecurityGroup}]' \
  --region us-west-2
```

## 3. Create Private Subnet in Outpost

```bash
aws ec2 create-subnet \
  --vpc-id $vpc_id \
  --cidr-block 192.168.32.0/19 \
  --availability-zone us-west-2a \
  --outpost-arn arn:aws:outposts:us-west-2:576319625758:outpost/op-0268f76782a30c66a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=kubernetes.io/role/internal-elb,Value=1},{Key=Name,Value=SubnetPrivateUSWEST2A}]' \
  --region us-west-2
```

## 4. Create Private Route Table

```bash
aws ec2 create-route-table \
  --vpc-id $vpc_id \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=PrivateRouteTableUSWEST2A}]' \
  --region us-west-2
```

## 5. Associate Route Table with Private Subnet

```bash
# Use the IDs from previous commands
subnet_id="subnet-xxxxxxxxxxxxxxxxx"
route_table_id="rtb-xxxxxxxxxxxxxxxxx"

aws ec2 associate-route-table \
  --route-table-id $route_table_id \
  --subnet-id $subnet_id \
  --region us-west-2
```

## 6. Create VPC Endpoint for EC2

```bash
# Use Security Group ID from previous command
sg_id="sg-xxxxxxxxxxxxxxxxx"

aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-west-2.ec2 \
  --subnet-ids $subnet_id \
  --security-group-ids $sg_id \
  --private-dns-enabled \
  --region us-west-2
```

## 7. Create VPC Endpoint for EC2 Messages

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-west-2.ec2messages \
  --subnet-ids $subnet_id \
  --security-group-ids $sg_id \
  --private-dns-enabled \
  --region us-west-2
```

## 8. Create VPC Endpoint for ECR API

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-west-2.ecr.api \
  --subnet-ids $subnet_id \
  --security-group-ids $sg_id \
  --private-dns-enabled \
  --region us-west-2
```

## 9. Create VPC Endpoint for ECR DKR

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-west-2.ecr.dkr \
  --subnet-ids $subnet_id \
  --security-group-ids $sg_id \
  --private-dns-enabled \
  --region us-west-2
```

## 10. Create S3 Gateway VPC Endpoint

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.us-west-2.s3 \
  --route-table-ids $route_table_id \
  --region us-west-2
```

## 11. Create VPC Endpoint for Secrets Manager

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-west-2.secretsmanager \
  --subnet-ids $subnet_id \
  --security-group-ids $sg_id \
  --private-dns-enabled \
  --region us-west-2
```

## 12. Create VPC Endpoint for SSM

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-west-2.ssm \
  --subnet-ids $subnet_id \
  --security-group-ids $sg_id \
  --private-dns-enabled \
  --region us-west-2
```

## 13. Create VPC Endpoint for SSM Messages

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-west-2.ssmmessages \
  --subnet-ids $subnet_id \
  --security-group-ids $sg_id \
  --private-dns-enabled \
  --region us-west-2
```

## 14. Create VPC Endpoint for STS

```bash
aws ec2 create-vpc-endpoint \
  --vpc-id $vpc_id \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-west-2.sts \
  --subnet-ids $subnet_id \
  --security-group-ids $sg_id \
  --private-dns-enabled \
  --region us-west-2
```

## 15. Create Security Group Ingress Rule for Private Subnet

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $sg_id \
  --protocol tcp \
  --port 443 \
  --cidr 192.168.32.0/19 \
  --description "Allow private subnets to communicate with VPC endpoints" \
  --region us-west-2
```

## 16. Create Security Group Ingress Rule for Inter-Node Communication

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $sg_id \
  --protocol all \
  --port 0-65535 \
  --source-group $sg_id \
  --description "Allow nodes to communicate with each other (all ports)" \
  --region us-west-2
```

## 17. Create CloudFormation Exports (Stack Outputs)

```bash
# For CloudFormation exports, you'd typically need to create the entire stack
# However, to achieve similar functionality, you can use Systems Manager Parameter Store

aws ssm put-parameter \
  --name "/EKSNetworking/VPC" \
  --value $vpc_id \
  --type String \
  --region us-west-2

aws ssm put-parameter \
  --name "/EKSNetworking/SharedNodeSecurityGroup" \
  --value $sg_id \
  --type String \
  --region us-west-2

aws ssm put-parameter \
  --name "/EKSNetworking/SubnetsPrivate" \
  --value $subnet_id \
  --type String \
  --region us-west-2

aws ssm put-parameter \
  --name "/EKSNetworking/VpcCidr" \
  --value "192.168.0.0/16" \
  --type String \
  --region us-west-2

aws ssm put-parameter \
  --name "/EKSNetworking/PrivateSubnetCidr" \
  --value "192.168.32.0/19" \
  --type String \
  --region us-west-2
```
