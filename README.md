Got it, Jarvis will convert this into a **clean, production-ready GitHub Wiki page** with proper GitHub-flavored Markdown, headings, tables, code blocks, and consistent formatting.
You can paste this **directly into a GitHub Wiki page** (Home.md or a new page).

---

# **EKS Cluster Setup using EKSCTL**

## **1. Overview**

This page documents the setup of an Amazon EKS Cluster using **eksctl**, based on our PRE DEV Managed Environment. This includes VPC reuse, subnet configuration, NAT/IGW setup, and the full eksctl cluster configuration along with validation steps.

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

### **Subnets Used**

| Type    | AZ         | Subnet ID                | Name                          |
| ------- | ---------- | ------------------------ | ----------------------------- |
| Private | us-west-2a | subnet-0b3364a5802574367 | subnet-pre-mkr-temp-private-a |
| Private | us-west-2b | subnet-0939805aeee14d4d3 | subnet-pre-mkr-temp-private-b |
| Public  | us-west-2a | subnet-0fa98a0dc964d561a | subnet-pre-mkr-temp-public-a  |
| Public  | us-west-2b | subnet-09e7e7daf298189be | subnet-pre-mkr-temp-public-b  |

These subnets already had the **Internet Gateway**, **NAT Gateway**, and route table associations configured.

---

# **4. Creating Subnets From Scratch (Optional)**

If these need to be replicated, below are the exact CLI commands used.

### **Subnet Creation**

```bash
# Private A
aws ec2 create-subnet \
  --vpc-id vpc-08a80834636025c17 \
  --cidr-block 10.143.66.0/26 \
  --availability-zone us-west-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-pre-mkr-temp-private-a}]'

# Private B
aws ec2 create-subnet \
  --vpc-id vpc-08a80834636025c17 \
  --cidr-block 10.143.66.64/26 \
  --availability-zone us-west-2b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-pre-mkr-temp-private-b}]'

# Public A
aws ec2 create-subnet \
  --vpc-id vpc-08a80834636025c17 \
  --cidr-block 10.143.67.0/26 \
  --availability-zone us-west-2a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-pre-mkr-temp-public-a}]'

# Public B
aws ec2 create-subnet \
  --vpc-id vpc-08a80834636025c17 \
  --cidr-block 10.143.67.64/26 \
  --availability-zone us-west-2b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=subnet-pre-mkr-temp-public-b}]'
```

---

# **5. IGW & NAT Setup**

### **Attach Internet Gateway**

```bash
aws ec2 attach-internet-gateway --vpc-id vpc-0abc123xyz --internet-gateway-id igw-0def456uvw
aws ec2 create-tags --resources igw-0def456uvw --tags Key=Name,Value=PRE-MKR-IGW
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
aws ec2 create-tags --resources nat-0123abc456def789 --tags Key=Name,Value=PRE-MKR-NAT
```

Wait for NAT to become **Available**.

---

# **6. eksctl Cluster Configuration**

We created a custom file:

```
pre-mkr-temp-cluster.yaml
```

### **Cluster YAML**

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: PRE-MKR-TEMP-Cluster
  region: us-west-2
  version: "1.31"

vpc:
  id: vpc-08a80834636025c17
  subnets:
    private:
      us-west-2a: { id: subnet-0b3364a5802574367 }
      us-west-2b: { id: subnet-0939805aeee14d4d3 }
    public:
      us-west-2a: { id: subnet-0fa98a0dc964d561a }
      us-west-2b: { id: subnet-09e7e7daf298189be }

managedNodeGroups:
  - name: ng-pre-mkr-temp-app
    ami: ami-0991f82bace1d9d8c
    amiFamily: AmazonLinux2023
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    privateNetworking: true
    ssh:
      allow: true
      publicKeyName: pre-mkr-nodegroup-sshkey
    securityGroups:
      attachIDs:
        - sg-03afdf513c0c04772
    labels:
      role: app
    tags:
      Name: pre-mkr-temp-app-nodes

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator"]

iam:
  withOIDC: true
```

---

# **7. AMI Selection**

We used a FICO-specific, custom AL2023 EKS-Optimized AMI for v1.31.

* **AMI Name:** `eks_1.31_nodegroup_al2023_baseline_1.80.0`
* **AMI ID:** `ami-0991f82bace1d9d8c`
* **Created On:** 2025-10-06

---

# **8. SSH Key Setup**

We reused the existing SSH keypair:

**Key:** `pre-mkr-nodegroup-sshkey`

This allows SSH access to worker nodes when required.

---

# **9. Create the Cluster**

```bash
eksctl create cluster -f pre-mkr-temp-cluster.yaml
```

---

# **10. Validation Steps**

### **Check CloudFormation Stacks**

```bash
aws cloudformation list-stacks --stack-status-filter CREATE_IN_PROGRESS CREATE_COMPLETE DELETE_FAILED ROLLBACK_IN_PROGRESS ROLLBACK_COMPLETE
```

### **Check Cluster Status**

```bash
aws eks describe-cluster --name PRE-MKR-TEMP-Cluster --region us-west-2 --query "cluster.status"
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
eksctl get addons --cluster PRE-MKR-TEMP-Cluster
```

### **OIDC Details**

```bash
aws eks describe-cluster --name PRE-MKR-TEMP-Cluster --query "cluster.identity.oidc.issuer"
```

### **CloudWatch Logs**

Navigate to:

```
CloudWatch → Logs → /aws/eks/PRE-MKR-TEMP-Cluster/cluster
```

---

# **11. Summary**

* Successfully created **PRE-MKR-TEMP-Cluster**
* Attached **2 worker nodes** using a custom AL2023 AMI
* Reused the existing VPC & subnet layout
* Enabled CloudWatch logging & OIDC integration
* Verified SSH and SSM access to nodes

---

