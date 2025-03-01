


You're right, and that's an important clarification. Let me revise the guidance for accessing the EKS cluster when using an on-premises VM as a bastion:

# Creating Node Groups for Amazon EKS Local Clusters on AWS Outposts (Revised)

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
