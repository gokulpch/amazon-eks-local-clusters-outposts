# amazon-eks-local-clusters-outposts
IaaC for creating Amazon EKS Local Clusters on AWS Outposts


# EKS Local Clusters on AWS Outposts: Control Plane Network Requirements

When deploying Amazon EKS local clusters on AWS Outposts, it's essential to understand the specific network and subnet requirements for the control plane. The control plane consists of three instances that must be properly distributed across subnets to ensure high availability and fault tolerance. Below is a detailed table outlining the possible configurations:

| **Number of Subnets** | **IP Addresses Required per Subnet** | **Total IP Addresses Required** | **Control Plane Instance Distribution** | **Notes** |
|:----------------------|:-------------------------------------|:--------------------------------|:----------------------------------------|:----------|
| **1 Subnet**          | 3                                    | 3                               | All 3 instances in a single subnet      | Not recommended due to lack of redundancy |
| **2 Subnets**         | 2 per subnet                         | 4                               | 2 instances in one subnet, 1 in the other | Provides some redundancy, but not optimal |
| **3 Subnets**         | 1 per subnet                         | 3                               | 1 instance per subnet                   | Recommended for optimal high availability |

## Key Considerations:

* **Subnet Placement:** All subnets used for the control plane must reside on the same logical Outpost and have proper routes and security group permissions to communicate with each other. 

* **Routing Configuration:** Subnets should have routes to the Outpost's local gateway (LGW) to facilitate access to the Kubernetes API server over your local network. 

* **IP Address Availability:** Ensure that each subnet has the required number of available IP addresses to accommodate the control plane instances.

By adhering to these configurations, you can ensure a robust and highly available control plane for your Amazon EKS local clusters on AWS Outposts.


# EKS Worker Nodes on AWS Outposts: Network Requirements

When deploying Amazon EKS worker nodes on AWS Outposts, it's essential to understand the network and subnet requirements to ensure optimal performance and high availability. Below is a detailed table outlining various deployment scenarios and their corresponding requirements:

| **Deployment Scenario** | **Subnet Configuration** | **IP Addresses Required per Subnet** | **Total IP Addresses Required** | **Node Distribution** | **Notes** |
|:------------------------|:-------------------------|:-------------------------------------|:--------------------------------|:----------------------|:----------|
| **Single Subnet Deployment** | One private subnet on the Outpost | Sufficient for desired number of nodes and pods | Depends on node and pod count | All nodes within a single subnet | Not recommended due to lack of redundancy and potential single point of failure. |
| **Multiple Subnet Deployment** | Multiple private subnets on the same Outpost | Sufficient for desired number of nodes and pods in each subnet | Depends on node and pod count | Nodes distributed across multiple subnets | Enhances fault tolerance by distributing nodes; ensure subnets have proper routes and security group permissions to communicate. |
| **Cross-Outpost Deployment** | Subnets across multiple Outposts | Sufficient for desired number of nodes and pods in each subnet | Depends on node and pod count | Nodes distributed across subnets on different Outposts | Requires intra-VPC communication setup; enhances resilience by spreading nodes across physical locations. |

## Key Considerations:

* **Subnet Placement:** All subnets used for worker nodes must reside on the same logical Outpost and have proper routes and security group permissions to communicate with each other.

* **Routing Configuration:** Subnets should have routes to the Outpost's local gateway (LGW) to facilitate access to the Kubernetes API server over your local network.

* **IP Address Availability:** Ensure that each subnet has a sufficient number of available IP addresses to accommodate the desired number of nodes and pods.

* **High Availability:** Distributing nodes across multiple subnets, and preferably across multiple Outposts, enhances fault tolerance and availability.

By adhering to these configurations, you can ensure a robust and highly available deployment of Amazon EKS worker nodes on AWS Outposts.


# Subnet Configurations for EKS Local Clusters on AWS Outposts

When deploying Amazon EKS local clusters on AWS Outposts, you have flexibility in configuring subnets for the control plane and worker nodes. You can place both the control plane and worker nodes within the same subnet or separate them into distinct subnets, depending on your architectural requirements. Below is a table summarizing these scenarios:

| **Scenario** | **Subnet Configuration** | **Description** | **Considerations** |
|:-------------|:-------------------------|:----------------|:-------------------|
| **Single Subnet for Both Control Plane and Worker Nodes** | * One private subnet on the Outpost<br>* Both control plane and worker nodes reside in this subnet | Simplifies network configuration by consolidating resources into a single subnet. | * Ensure the subnet has sufficient IP addresses to accommodate all control plane instances (3 IPs) and the desired number of worker nodes.<br>* May introduce a single point of failure; if the subnet encounters issues, both control plane and worker nodes are affected. |
| **Separate Subnets for Control Plane and Worker Nodes** | * One or more private subnets for the control plane<br>* One or more private subnets for worker nodes | Segregates control plane and worker nodes, enhancing fault isolation and potentially improving security. | * Requires careful planning to ensure each subnet has adequate IP addresses for the respective components.<br>* Increased complexity in network configuration and management. |

## Key Considerations:

* **IP Address Availability:** Each subnet must have sufficient IP addresses. The control plane requires a total of 3 IP addresses, which can be distributed across one, two, or three subnets. Worker nodes require additional IP addresses based on the number of nodes and pods.

* **Subnet Placement:** All subnets used must reside on the same logical Outpost and have proper routes and security group permissions to communicate with each other.

* **Routing Configuration:** Subnets should have routes to the Outpost's local gateway (LGW) to facilitate access to the Kubernetes API server over your local network.

* **High Availability:** Distributing control plane instances across multiple subnets enhances fault tolerance. For optimal high availability, it's recommended to use three subnets, each hosting one control plane instance.

By carefully selecting your subnet configuration based on these considerations, you can design an Amazon EKS local cluster on AWS Outposts that aligns with your availability, scalability, and management requirements.


# Managing EKS Cluster Access with IAM and RBAC

When creating an Amazon EKS cluster, the IAM entity (user or role) that initiates the creation is granted system:masters permissions within the Kubernetes RBAC system, effectively making it the cluster administrator. To delegate specific, non-administrative permissions to other IAM users or roles, follow the steps outlined below:

## 1. **Modify the `aws-auth` ConfigMap:**

The aws-auth ConfigMap in your EKS cluster maps IAM identities to Kubernetes roles or users. To grant additional IAM users or roles specific permissions:

* **Access the ConfigMap:**
   * Use kubectl to edit the aws-auth ConfigMap: 

```
kubectl edit configmap aws-auth -n kube-system
```

* **Add Mappings:**
   * Within the ConfigMap, add entries under mapUsers or mapRoles to associate IAM users or roles with Kubernetes usernames and groups.
   * **Example for an IAM Role:** 

```
mapRoles: |
  - rolearn: arn:aws:iam::123456789012:role/YourRoleName
    username: your-k8s-username
    groups:
      - your-k8s-group
```

   * **Example for an IAM User:** 

```
mapUsers: |
  - userarn: arn:aws:iam::123456789012:user/YourUserName
    username: your-k8s-username
    groups:
      - your-k8s-group
```

## 2. **Define Kubernetes RBAC Roles and RoleBindings:**

After mapping IAM identities, create Kubernetes Roles and RoleBindings to specify the permissions for these users or roles:

* **Create a Role:**
   * Define a Role that specifies allowed actions within a particular namespace. 

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: your-namespace
  name: your-role-name
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

* **Create a RoleBinding:**
   * Bind the Role to the Kubernetes user or group associated with the IAM identity. 

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: your-namespace
  name: your-rolebinding-name
subjects:
  - kind: User
    name: your-k8s-username
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: your-role-name
  apiGroup: rbac.authorization.k8s.io
```

## **Considerations for AWS Outposts:**

AWS Outposts supports service-linked roles, which are predefined by AWS services to perform specific actions on your behalf. However, creating custom service roles is not supported on AWS Outposts. Therefore, when managing EKS clusters on Outposts, ensure that you utilize the appropriate service-linked roles provided by AWS. For more information, refer to the AWS Outposts User Guide.

By carefully configuring the aws-auth ConfigMap and defining precise Kubernetes RBAC roles and bindings, you can grant specific permissions to IAM users or roles without assigning full administrative access to your EKS cluster.
