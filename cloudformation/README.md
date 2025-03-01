# CloudFormation Templates for Amazon EKS Local Clusters on AWS Outposts

This repository contains AWS CloudFromation (native AWS IaaC) templates for creating Amazon EKS Local Clusters on AWS Outposts

Two CFN Stacks:
1. Creates Networking required for creating fully-private Amazon EKS Local Clusters on AWS Outposts
2. Create a Amazon EKS - Local Clusters on AWS Outposts using the networking environment created above (users should choose the VPC and Subnet from the above stack as parameters when creationg a cluster)
