
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
Yes. Let's do it **the correct production way**.

---

# Step 1: Make sure the OIDC provider is associated

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster srinivas \
  --region us-east-1 \
  --approve
```

**Why?**

IRSA (IAM Roles for Service Accounts) requires an OIDC provider. If it's already associated, `eksctl` will tell you.

---

# Step 2: Download the IAM policy for Cluster Autoscaler

```bash
curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/iam-policy.json
```

This downloads:

```text
iam-policy.json
```

---

# Step 3: Create the IAM Policy

```bash
aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://iam-policy.json
```

After it succeeds, you'll get output similar to:

```text
arn:aws:iam::724516859343:policy/AmazonEKSClusterAutoscalerPolicy
```

This ARN is needed in the next step.

If you already created the policy, verify it:

```bash
aws iam list-policies --scope Local
```

---

# Step 4: Create the IAM ServiceAccount

```bash
eksctl create iamserviceaccount \
  --cluster=srinivas \
  --region=us-east-1 \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::724516859343:policy/AmazonEKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### What this command creates

```text
IAM Policy
      │
      ▼
IAM Role
      │
      ▼
ServiceAccount (cluster-autoscaler)
      │
      ▼
IRSA Annotation
```

---

# Step 5: Verify the ServiceAccount

Run:

```bash
kubectl describe sa cluster-autoscaler -n kube-system
```

You should see:

```text
Annotations:
eks.amazonaws.com/role-arn:
arn:aws:iam::724516859343:role/eksctl-srinivas-addon-iamserviceaccount-...
```

If you see this annotation, the ServiceAccount is configured correctly.

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
