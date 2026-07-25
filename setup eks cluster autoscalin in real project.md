Absolutely. Let's go through the **entire production process** from the beginning, including where your cluster failed and why. This will help you understand not just the commands, but the architecture behind Cluster Autoscaler on EKS.

---

# Step 1: Create an EKS Cluster

Example:

```bash
eksctl create cluster \
  --name srinivas \
  --region us-east-1 \
  --nodes 2 \
  --node-type t3.medium
```

This creates:

* EKS Control Plane
* Managed Node Group
* EC2 Worker Nodes
* Auto Scaling Group (ASG)
* VPC, Subnets, Security Groups (if not provided)

Architecture:

```text
AWS
│
├── EKS Cluster (Control Plane)
│
└── Managed Node Group
      │
      ├── EC2 Node 1
      ├── EC2 Node 2
      └── Auto Scaling Group (ASG)
```

---

# Step 2: Install Metrics Server

HPA needs Metrics Server to collect CPU and memory usage.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify:

```bash
kubectl get pods -n kube-system
```

You should see:

```text
metrics-server Running
```

---

# Step 3: Install Cluster Autoscaler

Cluster Autoscaler is **not installed automatically**.

It runs as a Deployment inside Kubernetes.

It watches for:

* Pending Pods
* Empty Nodes

Then communicates with AWS Auto Scaling Groups.

---

# Step 4: Create IAM Policy (hint: in your aws account iam policies list if you have this policy "AmazonEKSClusterAutoscalerPolicy" delete that then follow these step )  )

create a file :

```bash
vi iam-policy.json
---

paste:
```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:DescribeInstances",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeAvailabilityZones"
      ],
      "Resource": "*",
      "Effect": "Allow"
    }
  ]
}

```
Then create the policy:
```bash
aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://iam-policy.json
```

the expected out put :

```bash
{
    "Policy": {
        "PolicyName": "AmazonEKSClusterAutoscalerPolicy",
        "PolicyId": "ANPA2RMER5XH5WOLBGHVO",
        "Arn": "arn:aws:iam::724516859343:policy/AmazonEKSClusterAutoscalerPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2026-07-25T12:40:09+00:00",
        "UpdateDate": "2026-07-25T12:40:09+00:00"
    }
}
```
copy the ARN  here and use in the next command

# Step 5: Associate OIDC Provider

IRSA requires an OIDC provider.

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster srinivas \
  --region us-east-1 \
  --approve
```

Without this, ServiceAccounts cannot assume IAM roles.

---

# Step 6: Create an IAM ServiceAccount

This is where your issue occurred.

Create the ServiceAccount: ( repleace ARN paste here that copied ARN)

```bash
eksctl create iamserviceaccount \
  --cluster=srinivas \
  --region=us-east-1 \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AmazonEKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve
```

This creates:

```text
IAM Policy
      │
      ▼
IAM Role
      │
      ▼
ServiceAccount
```

Verify:

```bash
kubectl describe sa cluster-autoscaler -n kube-system
```

Expected:

```text
Annotations:
eks.amazonaws.com/role-arn:
arn:aws:iam::<ACCOUNT_ID>:role/...
```

---

# Step 7: Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh
```

---

# Step 8: Add the Autoscaler Helm Repository

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler

helm repo update
```

---

# Step 9: Install Cluster Autoscaler with the Correct ServiceAccount

Do **not** let Helm create a new ServiceAccount.

```bash
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=srinivas \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler
```

# Step 10: Check the Pod

```bash
kubectl get pods -n kube-system
```

Expected:

```text
cluster-autoscaler-xxxxxxxx Running
```
Check the Auto Scaling Group:
```bash
aws autoscaling describe-auto-scaling-groups \
  --region us-east-1 \
  --query "AutoScalingGroups[*].[AutoScalingGroupName,DesiredCapacity,MinSize,MaxSize]"
```
expected out:
```bash
[
    [
        "eks-ng-0b5be796-7acfcc99-80c5-f40a-b640-54bae4aa5d7b",
        2,
        2,
        2
    ]
]

```

Auto scaling working process:

| Component          | Responsibility                                                |
| ------------------ | ------------------------------------------------------------- |
| Metrics Server     | Collects CPU and memory metrics                               |
| HPA                | Increases or decreases the number of Pods                     |
| Cluster Autoscaler | Adds or removes worker nodes                                  |
| Auto Scaling Group | Launches or terminates EC2 instances                          |
| IRSA               | Allows the Cluster Autoscaler pod to securely access AWS APIs |
| Helm               | Installs and manages the Cluster Autoscaler chart             |

---

=============================================================================================================
now how to check that either autosscaling working or not 

Excellent! 🎉

Your Cluster Autoscaler is now **running successfully**.

From your output:

```text
cluster-autoscaler-aws-cluster-autoscaler-5ff75784bd-hq9dg   1/1 Running
```

and

```text
serviceAccountName: cluster-autoscaler
```

This confirms:

* ✅ Cluster Autoscaler is running.
* ✅ It is using the correct `cluster-autoscaler` ServiceAccount.
* ✅ IRSA is configured correctly.
* ✅ The previous `AccessDenied` issue has been resolved.

Now let's verify that **Cluster Autoscaler actually works**.

---

# How Cluster Autoscaler Works

There are **two scenarios** to test:

### Scale Up

When there are **Pending Pods** because the cluster doesn't have enough resources, Cluster Autoscaler should **add a new EC2 node**.

### Scale Down

When a node is underutilized and its Pods can be moved elsewhere, Cluster Autoscaler should **remove the extra EC2 node** after a delay.

We'll test **Scale Up** first.

---

# Step 1: Check the Current Number of Nodes

```bash
kubectl get nodes
```

Example:

```text
NAME                           STATUS   ROLES    AGE
ip-192-168-10-10.ec2.internal  Ready    <none>   30m
ip-192-168-20-15.ec2.internal  Ready    <none>   30m
```

Suppose you have **2 nodes**.

---

# Step 2: Check the Auto Scaling Group

Find your node group:

```bash
aws autoscaling describe-auto-scaling-groups \
  --region us-east-1 \
  --query "AutoScalingGroups[*].[AutoScalingGroupName,DesiredCapacity,MinSize,MaxSize]"
```

Example:

```text
my-nodegroup-asg    2    2    5
```

This means:

* Min = 2
* Desired = 2
* Max = 5

Cluster Autoscaler can scale up to **5 nodes**.

---

# Step 3: Create a Large Deployment

Create a file named `inflate.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 20
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      containers:
      - name: inflate
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: "1000m"
            memory: "1Gi"
```

Apply it:

```bash
kubectl apply -f inflate.yaml
```

---
then check posds are under pending state why because there is only two worker nodes present :
```bash
kubectl get pods
```
```text
inflate-xxxxx   Pending
inflate-yyyyy   Pending
inflate-zzzzz   Running
```
Some Pods will remain **Pending** because there isn't enough CPU or memory on the current nodes.


# Step 4: Watch the Pods

```bash
kubectl get pods -w
```
you can e only 2 worker nodes only 

---


# Step 7: Verify in AWS

Open the AWS Console:

**EC2 → Auto Scaling Groups**
here now click on the autoscaling group.

You should see:

```text
Desired Capacity
2 → 2
```
click on "edit "
 then
 increase the number at 
 ```text
 Max desired capacity
 ```
as 10 .
then you can see the number of worker nodes will be increased .


Then go to:

**EC2 → Instances**

A new EC2 instance should appear and transition to the `running` state.

---

# Step 8: Check Kubernetes Again

Run:

```bash
kubectl get nodes
```

You should now see:

```text

# Complete Flow

```text
Create Deployment
        │
        ▼
Pods Created
        │
        ▼
Enough Resources?
        │
   Yes ─────────► Pods Run
        │
        No
        ▼
Pods Stay Pending
        │
        ▼
Cluster Autoscaler Detects Pending Pods
        │
        ▼
Calls AWS Auto Scaling Group
        │
        ▼
ASG Launches New EC2 Instance
        │
        ▼
New Node Joins the Cluster
        │
        ▼
Scheduler Places Pending Pods
        │
        ▼
All Pods Become Running
```

---

## Production Note

In real production environments, Cluster Autoscaler is commonly used together with the **Horizontal Pod Autoscaler (HPA)**:

1. HPA increases the number of Pods when CPU or memory usage is high.
2. If those new Pods cannot be scheduled because the cluster lacks capacity, they become `Pending`.
3. Cluster Autoscaler detects the Pending Pods and adds worker nodes.
4. When traffic decreases, HPA reduces the number of Pods.
5. After nodes become underutilized for a period, Cluster Autoscaler removes the extra nodes.

This combination provides both **pod-level scaling** and **node-level scaling**, which is the standard autoscaling architecture for production EKS clusters.

