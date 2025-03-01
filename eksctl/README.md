
# Explanation of EKS Local Cluster Configuration on AWS Outposts


```
# An example ClusterConfig that creates a fully-private cluster on AWS Outposts.
# Since the VPC will be created by eksctl, it will lack connectivity to the API server because eksctl does not
# associate the VPC with the local gateway. Therefore, the command must be run with `--without-nodegroup`, as in
# `eksctl create cluster -f examples/37-outposts.yaml --without-nodegroup`, and the nodegroups can be created after
# ensuring connectivity to the API server.
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eks-local
  region: us-west-2
  version: "1.30"
vpc:
  subnets:
    private:
      us-west-2a:
        id: subnet-0e1cdc2fd18f3db39

privateCluster:
  enabled: true
  skipEndpointCreation: true

nodeGroups:
  - name: eks-local-ng
    # Optional, defaults to the smallest available instance type on the Outpost.
    instanceType: m5.large
    privateNetworking: true

outpost:
  # Required.
  controlPlaneOutpostARN: "arn:aws:outposts:us-west-2:576319625758:outpost/op-0268f76782a30c66a"
  # Optional, defaults to the smallest available instance type on the Outpost.
  controlPlaneInstanceType: m5.large
```


This `eksctl` configuration file defines a fully-private Amazon EKS Local cluster on AWS Outposts. Here's a breakdown of its key elements:

## Core Configuration
- Creates a cluster named "local-eks" in the us-west-2 region running Kubernetes 1.30
- Uses an existing private subnet (subnet-<id>) in us-west-2a - change this accordingly

## Private Cluster Settings
- `privateCluster.enabled: true` - Makes the cluster fully private with no public endpoint
- `skipEndpointCreation: true` - Prevents creating a public endpoint for the control plane

## Outpost Configuration
- Deploys the control plane on a specific AWS Outpost (ARN specified)
- Uses m5.large instances for the control plane nodes - change this accordingly based on the capacity available
- Configurable: uses the smallest available instance type on the Outpost if not specified

## Node Group Configuration
- Defines one node group named "local-eks-ng"
- Uses m5.large instances for worker nodes - change this accordingly based on the capacity available
- Sets `privateNetworking: true` to ensure nodes have only private IP addresses

## Important Deployment Note
- Must be deployed with `--without-nodegroup` flag as noted in the comments
- The reason: eksctl doesn't associate the VPC with the local gateway, which is needed for API connectivity
- Nodegroups must be created after ensuring connectivity to the API server

This configuration creates a fully isolated EKS cluster with both control plane and worker nodes running on the specified AWS Outpost, with no external internet connectivity.



# Creating Node Groups for Amazon EKS Local Clusters on AWS Outposts

Creating a node group for an Amazon EKS Local cluster on AWS Outposts requires specific considerations based on your chosen architectural pattern. The process differs depending on whether you'll manage your cluster from a bastion host on the Outpost itself or from your on-premises environment.

## Option 1: Using a Bastion Host on the AWS Outpost

If you plan to use an EC2 instance on the Outpost rack as your bastion host:

1. **IAM Configuration for Outpost Bastion**:
   - Attach an IAM role to the bastion instance with the following permissions:
     - `eks:DescribeCluster`
     - `eks:ListClusters`
     - `ec2:DescribeInstances`
     - `ec2:DescribeSubnets`
     - `ec2:DescribeSecurityGroups`
     - `iam:PassRole` (for the node group role)
   - Ensure the bastion role has permissions to assume the cluster creator role if needed

2. **Network Configuration**:
   - Place the bastion in a subnet with routing to the EKS control plane endpoints
   - Configure security groups to allow SSH access from your management network
   - Allow outbound access to the EKS API endpoints (port 443)

3. **Node Group Creation Process**:
   ```bash
   # Install eksctl and kubectl on the bastion
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
   sudo mv /tmp/eksctl /usr/local/bin
   
   # Get cluster details
   aws eks update-kubeconfig --name your-cluster-name --region your-region
   
   # Create the node group
   eksctl create nodegroup \
     --cluster your-cluster-name \
     --region your-region \
     --name your-nodegroup-name \
     --node-type m5.large \
     --nodes 3 \
     --nodes-min 1 \
     --nodes-max 4 \
     --managed=false \
     --node-ami-family AmazonLinux2 \
     --subnet-ids subnet-xxxxx \
     --instance-prefix your-node-prefix \
     --ssh-access \
     --ssh-public-key your-key \
     --asg-access \
     --external-dns-access \
     --full-ecr-access \
     --outpost-arn arn:aws:outposts:region:account:outpost/op-xxxxxxxxx
   ```

## Option 2: Using an On-Premises VM as Bastion

If you plan to use a VM in your on-premises environment as your bastion:

1. **IAM Role Assumption**:
   - Locate the cluster creator IAM role from the CloudFormation stack outputs
   - Configure your on-premises environment to assume this role:
     - Set up AWS credentials with permissions to assume the role
     - Create an AWS profile for role assumption

2. **Role Assumption Configuration**:
   ```bash
   # Configure AWS CLI profile for role assumption
   aws configure set role_arn arn:aws:iam::123456789012:role/ClusterCreatorRole --profile eks-admin
   aws configure set source_profile default --profile eks-admin
   aws configure set region your-region --profile eks-admin
   
   # Alternatively, use temporary credentials
   aws sts assume-role \
     --role-arn arn:aws:iam::123456789012:role/ClusterCreatorRole \
     --role-session-name EKSAdminSession \
     --duration-seconds 3600
   
   # Export the temporary credentials
   export AWS_ACCESS_KEY_ID=...
   export AWS_SECRET_ACCESS_KEY=...
   export AWS_SESSION_TOKEN=...
   ```

3. **Local Network Requirements**:
   - Ensure network connectivity from on-premises to the Outpost Local Gateway
   - Configure routing to allow access to the EKS API endpoint through the Outpost
   - Set up DNS resolution for the EKS API endpoint

4. **Node Group Creation**:
   ```bash
   # Update kubeconfig with role assumption
   aws eks update-kubeconfig \
     --name your-cluster-name \
     --region your-region \
     --profile eks-admin
   
   # Create the node group
   eksctl create nodegroup \
     --cluster your-cluster-name \
     --region your-region \
     --profile eks-admin \
     --name your-nodegroup-name \
     --node-type m5.large \
     --nodes 3 \
     --nodes-min 1 \
     --nodes-max 4 \
     --managed=false \
     --node-ami-family AmazonLinux2 \
     --subnet-ids subnet-xxxxx \
     --outpost-arn arn:aws:outposts:region:account:outpost/op-xxxxxxxxx
   ```

## Key Considerations for Both Approaches

1. **Node AMI Selection**:
   - Use EKS-optimized AMIs that are pre-cached on the Outpost
   - Specify the correct architecture (x86_64 or arm64) based on your Outpost configuration

2. **Outpost Capacity Planning**:
   - Consider the physical capacity limits of your Outpost when sizing node groups
   - Set appropriate min/max values for node autoscaling

3. **Subnet Configuration**:
   - Ensure the subnet has sufficient IP addresses for your planned node count
   - Verify that the subnet is properly associated with the Outpost ARN

4. **Private Registry Access**:
   - Configure node groups to access container images through VPC endpoints
   - Consider using a local container registry mirror if internet access is limited

5. **Monitoring and Logging**:
   - Set up CloudWatch agent on nodes to send logs and metrics via VPC endpoints
   - Configure node group with appropriate instance profile for logging permissions

By carefully selecting the approach that aligns with your operational model and following these guidelines, you can successfully create and manage node groups for your Amazon EKS Local cluster on AWS Outpost.


You're right, and that's an important clarification. Let me revise the guidance for accessing the EKS cluster when using an on-premises VM as a bastion:

# Creating Node Groups for Amazon EKS Local Clusters on AWS Outposts

## Option 2: Using an On-Premises VM as Bastion

When using a VM in your on-premises environment as your bastion, the cluster creator role is not typically shown directly in the CloudFormation outputs. Instead:

1. **Identify the Correct IAM Role**:
   - The ServiceRoleARN is available in the CloudFormation outputs, but this is the EKS service role, not the cluster creator role
   - The cluster creator role is the IAM entity (user/role) that was used to execute the CloudFormation template or create the cluster
   - You need to use the same IAM identity that created the cluster, or:
     - Find the IAM user/role in the AWS IAM console that has the required permissions
     - Check AWS CloudTrail logs for the `eks:CreateCluster` action to identify which IAM entity created the cluster

2. **Access Configuration**:
   ```bash
   # If you know the IAM role that created the cluster
   aws configure set role_arn arn:aws:iam::123456789012:role/KnownCreatorRole --profile eks-admin
   aws configure set source_profile default --profile eks-admin
   aws configure set region your-region --profile eks-admin
   
   # Update kubeconfig to use this profile
   aws eks update-kubeconfig \
     --name your-cluster-name \
     --region your-region \
     --profile eks-admin
   ```

3. **Alternative: Add Your IAM Entity to aws-auth ConfigMap**:
   - If you cannot use the original creator role, you need to add your IAM entity to the aws-auth ConfigMap
   - This requires one-time access using the creator role or by someone who already has cluster-admin access
   - Connect to the cluster using an entity that already has access, then:

   ```bash
   # Edit the aws-auth ConfigMap
   kubectl edit configmap aws-auth -n kube-system
   
   # Add your IAM role or user to the mapRoles or mapUsers section
   # Add this under mapRoles:
   - rolearn: arn:aws:iam::123456789012:role/YourAdminRole
     username: admin
     groups:
       - system:masters
   
   # Alternatively, use eksctl to manage auth
   eksctl create iamidentitymapping \
     --cluster your-cluster-name \
     --region your-region \
     --arn arn:aws:iam::123456789012:role/YourAdminRole \
     --username admin \
     --group system:masters
   ```

4. **Proceed with Node Group Creation**:
   ```bash
   # After gaining access, create the node group
   eksctl create nodegroup \
     --cluster your-cluster-name \
     --region your-region \
     --name your-nodegroup-name \
     --node-type m5.large \
     --nodes 3 \
     --nodes-min 1 \
     --nodes-max 4 \
     --managed=false \
     --node-ami-family AmazonLinux2 \
     --subnet-ids subnet-xxxxx \
     --outpost-arn arn:aws:outposts:region:account:outpost/op-xxxxxxxxx
   ```

This approach acknowledges that the cluster creator role isn't directly accessible from CloudFormation outputs and provides practical alternatives for gaining the necessary access to create node groups.
