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

Create the ServiceAccount:

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
  --set serviceAccount.create=false \
  --set serviceAccount.name=cluster-autoscaler
```

This tells Helm to use the IRSA-enabled ServiceAccount you created in Step 6.

---

# What Happened in Your Cluster?

You installed the chart like this:

```bash
helm install cluster-autoscaler ...
```

without specifying an existing ServiceAccount. 

Helm therefore created its own ServiceAccount:

```text
cluster-autoscaler-aws-cluster-autoscaler
```

That ServiceAccount had **no IAM role annotation**, so the pod fell back to using the EC2 worker node's IAM role.

---

# Step 10: Check the Pod

```bash
kubectl get pods -n kube-system
```

Expected:

```text
cluster-autoscaler-xxxxxxxx Running
```

In your case it showed:

```text
CrashLoopBackOff
```

because it couldn't call the AWS Auto Scaling API. 

---

# Step 11: Check the Logs

```bash
kubectl logs -n kube-system <cluster-autoscaler-pod>
```

You previously saw an error similar to:

```text
AccessDenied:
autoscaling:DescribeAutoScalingGroups
```

That was the key clue that the pod was **not using the correct IAM role**.

---

# Step 12: Verify the Deployment Uses the Correct ServiceAccount

```bash
kubectl get deployment cluster-autoscaler-aws-cluster-autoscaler \
  -n kube-system -o yaml | grep serviceAccountName
```

Expected:

```text
serviceAccountName: cluster-autoscaler
```

If it instead shows:

```text
serviceAccountName: cluster-autoscaler-aws-cluster-autoscaler
```

then Helm is still using its own ServiceAccount.

---

# Step 13: Test Scaling

Deploy an application:

```bash
kubectl apply -f deployment.yaml
```

Create an HPA:

```bash
kubectl apply -f hpa.yaml
```

Generate CPU load:

```bash
kubectl run load-generator \
  --image=busybox \
  -- /bin/sh -c "while true; do wget -q -O- http://<service-name>; done"
```

Flow:

```text
CPU Usage ↑
      │
      ▼
Metrics Server
      │
      ▼
HPA
      │
      ▼
More Pods
      │
      ▼
Pods Pending?
      │
      ▼
Cluster Autoscaler
      │
      ▼
AWS Auto Scaling Group
      │
      ▼
Launch New EC2 Instance
      │
      ▼
Node Joins Cluster
      │
      ▼
Pending Pods Scheduled
```

---

# Components and Their Responsibilities

| Component          | Responsibility                                                |
| ------------------ | ------------------------------------------------------------- |
| Metrics Server     | Collects CPU and memory metrics                               |
| HPA                | Increases or decreases the number of Pods                     |
| Cluster Autoscaler | Adds or removes worker nodes                                  |
| Auto Scaling Group | Launches or terminates EC2 instances                          |
| IRSA               | Allows the Cluster Autoscaler pod to securely access AWS APIs |
| Helm               | Installs and manages the Cluster Autoscaler chart             |

---

## Why Your Cluster Autoscaler Crashed

The root cause was:

1. You correctly created an IRSA-enabled ServiceAccount for the **AWS Load Balancer Controller**. 
2. You installed the Cluster Autoscaler Helm chart without configuring it to use an IRSA-enabled ServiceAccount. 
3. Helm created its own ServiceAccount.
4. That ServiceAccount had no IAM role annotation.
5. The pod used the EC2 node IAM role instead.
6. The node IAM role did not have the required Auto Scaling permissions.
7. AWS returned `AccessDenied`, and the Cluster Autoscaler failed to start correctly.

Once the Deployment is configured to use the correct IRSA-enabled ServiceAccount with the `AmazonEKSClusterAutoscalerPolicy`, the Cluster Autoscaler should be able to communicate with the Auto Scaling Group and run normally.
