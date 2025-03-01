# Components Created by the EKS Cluster CloudFormation Template

This CloudFormation template creates an Amazon EKS Local cluster on AWS Outpost. Here's a comprehensive list of all components it creates:

## IAM Resources

1. **ServiceRole**
   - Type: `AWS::IAM::Role`
   - Purpose: IAM role for the EKS control plane
   - Policy: AmazonEKSLocalOutpostClusterPolicy
   - Trust Relationship: EC2 service principal (mapped based on AWS partition)

## Security Groups

2. **ControlPlaneSecurityGroup**
   - Type: `AWS::EC2::SecurityGroup`
   - Purpose: Controls communication between control plane and worker nodes
   - VPC: Uses the provided VpcId parameter

## Security Group Rules

3. **IngressDefaultClusterToNodeSG**
   - Type: `AWS::EC2::SecurityGroupIngress`
   - Purpose: Allows all traffic from cluster security group to node security group
   - Protocol: All (-1)
   - Port Range: 0-65535
   - Source: EKS cluster security group
   - Destination: Shared node security group (from parameter)

4. **IngressNodeToDefaultClusterSG**
   - Type: `AWS::EC2::SecurityGroupIngress`
   - Purpose: Allows all traffic from node security group to cluster security group
   - Protocol: All (-1)
   - Port Range: 0-65535
   - Source: Shared node security group (from parameter)
   - Destination: EKS cluster security group

## EKS Resources

5. **ControlPlane**
   - Type: `AWS::EKS::Cluster`
   - Purpose: The EKS Local cluster on Outpost
   - Key Properties:
     - Authentication Mode: CONFIG_MAP
     - Bootstrap Admin Permissions: Enabled
     - Kubernetes Version: From parameter (1.28-1.31)
     - Outpost Configuration:
       - Instance Type: From parameter (m5.large, c5.large, etc.)
       - Outpost ARN: From parameter
     - VPC Configuration:
       - Private Endpoint: Enabled
       - Public Endpoint: Disabled
       - Security Groups: ControlPlaneSecurityGroup
       - Subnets: PrivateSubnetId from parameter

## CloudFormation Outputs

6. **ARN**
   - Value: Cluster ARN
   - Exported as `${StackName}::ARN`

7. **CertificateAuthorityData**
   - Value: Cluster certificate authority data
   - Not exported

8. **ClusterFullyPrivate**
   - Value: true
   - Exported as `${StackName}::ClusterFullyPrivate`

9. **ClusterSecurityGroupId**
   - Value: Security group ID created by EKS
   - Exported as `${StackName}::ClusterSecurityGroupId`

10. **ClusterStackName**
    - Value: Current stack name
    - Not exported

11. **Endpoint**
    - Value: Cluster API server endpoint
    - Exported as `${StackName}::Endpoint`

12. **FeatureNATMode**
    - Value: Single
    - Not exported

13. **SecurityGroup**
    - Value: ControlPlaneSecurityGroup ID
    - Exported as `${StackName}::SecurityGroup`

14. **ServiceRoleARN**
    - Value: IAM role ARN for cluster
    - Exported as `${StackName}::ServiceRoleARN`

## Additional Configurations

15. **ServicePrincipalPartitionMap**
    - Type: CloudFormation Mapping
    - Purpose: Maps service principals across different AWS partitions
    - Includes mappings for different AWS regions (standard, China, GovCloud, etc.)

This template creates a fully private EKS Local cluster on AWS Outpost with no public internet access, using the networking infrastructure that must be provided as parameters.
