
---
# **EKS Cluster Autoscaler – Overview and Implementation Guide**
---

## **What is EKS Cluster Autoscaler?**  
EKS Cluster Autoscaler is a Kubernetes component that automatically adjusts the number of worker nodes in an **Amazon EKS** cluster based on workload demands. It ensures that sufficient compute resources are available while also optimizing costs by removing unused nodes when they are no longer needed.

### **Why Use EKS Cluster Autoscaler?**
- **Scales Up Nodes**: Adds new worker nodes when the cluster runs out of resources.
- **Scales Down Nodes**: Removes underutilized nodes to save costs.
- **Improves Performance**: Ensures applications have enough compute capacity.
- **Optimizes Costs**: Removes idle resources automatically.

---

## **Step-by-Step Implementation of EKS Cluster Autoscaler**

### **Step 1: Describe the Node Group in EKS**
To check the existing node group details, run:
```sh
aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name <node-group-name>
```
This command provides information about the current node group, including its scaling configuration, IAM roles, and status.

---

### **Step 2: Attach IAM Policy for Cluster Autoscaler**  
EKS worker nodes need proper IAM permissions to manage Auto Scaling Groups (ASG). Attach the **AmazonEKSClusterAutoscalerPolicy** to the IAM role of your worker nodes. 

if not exist create names policy "AmazonEKSClusterAutoscalerPolicy"
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": "*"
    }
  ]
}
```
## Step 2.1: Create an IAM OIDC Provider for Your Cluster

Set your cluster name in an environment variable:

```bash
cluster_name=ekswithavinash
```

Extract the OIDC ID from your cluster:

```bash
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id
```

Check if an IAM OIDC provider already exists for your cluster:

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

- **If output is returned:** You already have an IAM OIDC provider and can skip the next step.
- **If no output is returned:** Create an IAM OIDC provider for your cluster:

```bash
eksctl utils associate-iam-oidc-provider --cluster ekswithavinash --approve
```

---
### 2.2 – Download and Create the IAM Policy

Download the IAM policy required for the AWS Load Balancer Controller:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

Create the IAM policy:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 2.3 – Create the IAM Service Account
Create the IAM service account for the AWS Load Balancer Controller using `eksctl`. This command attaches the policy to the service account and (if it exists) overrides the existing service account:

```bash
eksctl create iamserviceaccount \
    --cluster=ekswithavinash \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<<account-id>>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ap-south-1 \
    --approve
```
###2.4 check service account 

```bash

kubectl get serviceaccounts -n kube-system

```

---



---

###Helm installation 
If you're using **Ubuntu** (EC2 instance or your local Ubuntu machine), these are the standard Helm installation steps.

---

# Step 1: Download the Helm installation script

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```

### Why?

This downloads the official Helm installation script maintained by the Helm project.

---

# Step 2: Give execute permission

```bash
chmod 700 get_helm.sh
```

### Why?

By default, the downloaded file isn't executable. This command allows you to run it.

---

# Step 3: Install Helm

```bash
./get_helm.sh
```

### Why?

The script:

* Downloads the latest stable Helm binary.
* Places it in `/usr/local/bin/helm`.
* Makes it executable.

---

# Step 4: Verify the installation

```bash
helm version
```

Example output:

```text
version.BuildInfo{
Version:"v3.18.4",
GitCommit:"...",
GitTreeState:"clean",
GoVersion:"go1.24.x"
}
```

---

### **Step 3: Install Cluster Autoscaler using Helm**  
Deploy Cluster Autoscaler using Helm, which simplifies the installation process.

#### **Run the following commands:**
```sh
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=<cluster-name> \
  --set awsRegion=<region-code> \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false
```

**Explanation:**
- **`autoDiscovery.clusterName=<cluster-name>`**: Enables automatic node discovery for the specified EKS cluster.
- **`awsRegion=<region-code>`**: Set your AWS region, e.g., `ap-south-1` (Mumbai).
- **`balance-similar-node-groups=true`**: Distributes workloads evenly across node groups.
- **`skip-nodes-with-system-pods=false`**: Ensures system pods do not block node termination.


#### **Verify Cluster Autoscaler Deployment**
Check if the Cluster Autoscaler pod is running:
```sh
kubectl get pods -n kube-system | grep cluster-autoscaler
```

---

### **Step 4: Deploy `php-apache` Application and Increase Replicas**
Now, deploy a sample application (`php-apache`) and increase the replica count to test autoscaling.

#### **Scale Up the Application**
```sh
kubectl scale deployment php-apache --replicas=50
```
- This command increases the number of running pods from the default count to **50**.
- If the current worker nodes **do not have enough resources**, Cluster Autoscaler will scale up and add more nodes.

---

### **Step 5: Verify Logs and Autoscaler Behavior**
Monitor the logs of the deployment and ensure that autoscaling is happening.

#### **Check Application Logs**
```sh
kubectl logs -f deployment/php-apache
```

#### **Monitor Cluster Autoscaler Logs**
```sh
kubectl logs -f -n kube-system deployment/cluster-autoscaler
```
- If scaling is working correctly, you will see messages like **"Expanding node group"** in the logs.
- If it does not scale up, check for errors in the **Cluster Autoscaler** logs.

---
