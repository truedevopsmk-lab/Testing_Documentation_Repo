---

# **EKS Cluster Setup using EKSCTL**

## **1. Overview**

This page documents the setup of an Amazon EKS Cluster using **eksctl**, based on our PRE DEV Managed Environment. This includes VPC reuse, subnet configuration, NAT/IGW setup, and the full *eksctl* cluster configuration along with validation steps.

---

# **2. Prerequisites**

### **VPC Details**

We reused an existing VPC instead of creating one through eksctl.

| Property   | Value                   |
| ---------- | ----------------------- |
| VPC ID     | `vpc-08a80834636025c17` |
| CIDR Block | `10.143.64.0/21`        |

---

# **3. VPC & Networking Setup**

Since we did not create the VPC automatically using eksctl, we mapped all subnets manually. 
_(If there are no subnets available, please create them following the steps below in next section)_

### **Subnets Used**

Below are the subnets created for the PRE-1 EKS Cluster setup, following the /25(for Private) and /27(for Public) CIDR patterns.

| Type    | AZ         | CIDR Block       | Name                                     | Subnet ID                |
| ------- | ---------- | ---------------- | ---------------------------------------- | ------------------------ |
| Private | us-west-2a | 10.143.70.0/25   | subnet-pre-1-uswt2-eks-cluster-private-a | subnet-0636b5c63f50101ba |
| Private | us-west-2b | 10.143.70.128/25 | subnet-pre-1-uswt2-eks-cluster-private-b | subnet-0abc54d600fbe43c9 |
| Public  | us-west-2a | 10.143.71.0/27   | subnet-pre-1-uswt2-eks-cluster-public-a  | subnet-0f0c4b6f86cc48ab8 |
| Public  | us-west-2b | 10.143.71.32/27  | subnet-pre-1-uswt2-eks-cluster-public-b  | subnet-07ab322a4c1cd1351 |


These subnets already had the **Internet Gateway**, **NAT Gateway**, and route table associations configured to support EKS node provisioning..

---

# **4. Creating Subnets From Scratch (Optional)**

If these need to be replicated, below are the exact CLI commands used.

### **Subnet Creation**

If you need to recreate these PRE-1 style subnets, the following CLI commands help replicate the exact structure.

**Subnet Creation**

```bash
# Private Subnet - us-west-2a (/25)
aws ec2 create-subnet \
  --vpc-id vpc-08a80834636025c17 \
  --cidr-block 10.143.70.0/25 \
  --availability-zone us-west-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-pre-1-uswt2-eks-cluster-private-a}]'

# Private Subnet - us-west-2b (/25)
aws ec2 create-subnet \
  --vpc-id vpc-08a80834636025c17 \
  --cidr-block 10.143.70.128/25 \
  --availability-zone us-west-2b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-pre-1-uswt2-eks-cluster-private-b}]'

# Public Subnet - us-west-2a (/27)
aws ec2 create-subnet \
  --vpc-id vpc-08a80834636025c17 \
  --cidr-block 10.143.71.0/27 \
  --availability-zone us-west-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-pre-1-uswt2-eks-cluster-public-a}]'

# Public Subnet - us-west-2b (/27)
aws ec2 create-subnet \
  --vpc-id vpc-08a80834636025c17 \
  --cidr-block 10.143.71.32/27 \
  --availability-zone us-west-2b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-pre-1-uswt2-eks-cluster-public-b}]'

```

---

# **5. IGW & NAT Setup**

### **Attach Internet Gateway**

```bash
aws ec2 attach-internet-gateway --vpc-id vpc-0abc123xyz --internet-gateway-id igw-0def456uvw
aws ec2 create-tags --resources igw-0def456uvw --tags Key=Name,Value=PRE-Group-IGW
```

### **Create NAT Gateway**

Allocate EIP:

```bash
aws ec2 allocate-address
```

Create NAT Gateway:

```bash
aws ec2 create-nat-gateway \
  --subnet-id <public-subnet-1-id> \
  --allocation-id eipalloc-0ghij789 \
  --connectivity-type public
```

Tag:

```bash
aws ec2 create-tags --resources nat-0123abc456def789 --tags Key=Name,Value=PRE-Group-NAT
```

Wait for NAT to become **Available**.

---

# **6. eksctl Cluster Configuration**

We created a custom file:

```
PRE-1-uswt2-eks-cluster.yaml
```

### **Cluster YAML**

```yaml
# ============================================================
# EKS Cluster + Nodegroup Config (One-shot setup)
# Name: PRE-4-uswt2-eks-clusterv2
# File: PRE-4-uswt2-eks-cluster.yaml
# Region: us-west-2
# Version: Kubernetes 1.32 (AL2023 compatible)
# 
# FICO EKS Nodegroup AL2023 Baseline - Built on ami-022a2d9a04badc5e0 [07:49:43 06 Oct 2025]
# AMI used: ami-063a81f8743fece51 | x86_64
#
# Includes: Temporary public access for node bootstrap and can be reverted right after nodegroup creation.
  # Enable temporary public endpoint for node bootstrap
  # We can disable it doing - aws eks update-cluster-config --resources-vpc-config endpointPublicAccess=false,endpointPrivateAccess=true
# ============================================================

apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: PRE-4-uswt2-eks-clusterv2
  region: us-west-2
  version: "1.32"

# ------------------------------------------------------------
# VPC CONFIGURATION
# ------------------------------------------------------------
vpc:
  id: vpc-08a80834636025c17              ## Existing VPC ID
  subnets:                               ## Custom subnets mapped by AZ
    private:
      us-west-2a:
        id: subnet-0636b5c63f50101ba      # 10.143.70.0/25
      us-west-2b:
        id: subnet-0abc54d600fbe43c9      # 10.143.70.128/25
    public:
      us-west-2a:
        id: subnet-0f0c4b6f86cc48ab8      # 10.143.71.0/27
      us-west-2b:
        id: subnet-07ab322a4c1cd1351      # 10.143.71.32/27

  clusterEndpoints:
    publicAccess: true
    privateAccess: true

# ------------------------------------------------------------
# NODEGROUP CONFIGURATION
# ------------------------------------------------------------
managedNodeGroups:
  - name: ng-pre-1-eks-cluster        ## Node group name
    ami: ami-063a81f8743fece51        ## Custom AMI (EKS 1.32 AL2023) - Screenshot below on how to find this.
    amiFamily: AmazonLinux2023        ## Must specify when using custom AMI (AmazonLinux2023, AmazonLinux2, Bottlerocket,)
    instanceType: t3.medium           ## Node's EC2 instance type
    desiredCapacity: 2                ## Count of Desired no.of nodes
    minSize: 1
    maxSize: 3
    privateNetworking: true           ## Use private subnets for nodes
    ssh:
      allow: true
      publicKeyName: pre-1-sshkey     ## Your SSH key for node access
    securityGroups:
      attachIDs:
        - sg-03afdf513c0c04772        ## Security group for EKS worker nodes — allows communication with control plane and node-to-node traffic.
    labels:
      role: app
    tags:
      Name: pre-1-app-nodes

    # Recommended IAM Addon Policies
    iam:
      withAddonPolicies:
        awsLoadBalancerController: true
        autoScaler: true
        cloudWatch: true
        ebs: true

# ------------------------------------------------------------
# LOGGING CONFIGURATION
# ------------------------------------------------------------
cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator

# ------------------------------------------------------------
# IAM CONFIGURATION
# ------------------------------------------------------------
iam:
  withOIDC: true

```

---

# **7. AMI Selection**

We used a FICO-specific, custom AL2023 EKS-Optimized AMI for v1.31.

FICO EKS Nodegroup AL2023 Baseline
* **AMI Name:** `eks_1.32_nodegroup_al2023_baseline_1.80.0`
* **AMI ID:** `ami-063a81f8743fece51`
* **Created On:** 2025-10-06

---

# **8. SSH Key Setup**

We reused the existing SSH keypair:

**Key:** `pre-1-sshkey`

This allows SSH access to worker nodes when required.

---

# **9. Create the Cluster**

```bash
eksctl create cluster -f PRE-4-uswt2-eks-cluster.yaml
```

---

# **10. Validation Steps**

### **Check CloudFormation Stacks**

```bash
aws cloudformation list-stacks --stack-status-filter CREATE_IN_PROGRESS CREATE_COMPLETE DELETE_FAILED ROLLBACK_IN_PROGRESS ROLLBACK_COMPLETE
```

### **Check Cluster Status**

```bash
aws eks describe-cluster --name PRE-4-uswt2-eks-cluster.yaml --region us-west-2 --query "cluster.status"
```

### **Check Nodes**

```bash
kubectl get nodes -o wide
```

### **Cluster Health**

```bash
kubectl get componentstatuses
```

### **Add-ons**

```bash
eksctl get addons --cluster PRE-4-uswt2-eks-cluster.yaml
```

### **OIDC Details**

```bash
aws eks describe-cluster --name PRE-1-uswt2-eks-cluster --query "cluster.identity.oidc.issuer"
```

### **CloudWatch Logs**

Navigate to:

```
CloudWatch → Logs → /aws/eks/PRE-1-uswt2-eks-cluster/cluster
```

---

# **11. Summary**

* Successfully created **PRE-1-uswt2-eks-cluster**
* Attached **2 worker nodes** using a custom AL2023 AMI _(supporting v1.32 Cluster/Crossplane)_
* Reused the existing VPC & subnet layout
* Enabled CloudWatch logging & OIDC integration
* Verified SSH and SSM access to nodes

---

