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
