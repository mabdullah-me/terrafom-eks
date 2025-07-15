
# 🚀 Provision an EKS Cluster

This repository is a companion to the [Provision an EKS Cluster tutorial](https://developer.hashicorp.com/terraform/tutorials/kubernetes/eks).  
It contains Terraform configuration files to provision an Amazon EKS cluster on AWS.

---

## 🧰 Prerequisites

Ensure the following tools are installed:

- [Terraform](https://developer.hashicorp.com/terraform/downloads)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [eksctl](https://eksctl.io/)

Also, configure your AWS CLI with credentials:

```bash
aws configure
````

---

## 📦 Deploy the EKS Cluster

Clone the repository and run the following commands:

```bash
git clone <your-repo-url>
cd <repo-directory>

terraform init
terraform plan
terraform apply
```

---

## 🔧 Update kubeconfig for EKS cluster

Once the cluster is provisioned, run the following to update your kubeconfig:

```bash
aws eks --region <region> update-kubeconfig --name <cluster-name>
```

---

## 📁 Setup EFS Storage Class

To enable dynamic provisioning for EFS, follow these steps:

### 📚 References

* [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-efs)
* [EFS CSI Dynamic Provisioning](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/dynamic_provisioning/README.md)
* [AWS EFS with EKS](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)

### 📝 Steps

1. **Download the StorageClass manifest:**

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
```

2. **Get the filesystem ID:**

```bash
aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text
```

Example output:

```
fs-0d4a5eaa988b19bca
```

3. **Edit the manifest:**

Update `fileSystemId` in `storageclass.yaml` with the actual ID obtained above.

4. **Apply the manifest:**

```bash
kubectl apply -f k8s-manifests/storageclass.yaml
```

5. **Make EFS StorageClass the default:**

```bash
# Mark existing default StorageClass (e.g. gp2) as non-default
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Set EFS StorageClass as default
kubectl patch storageclass efs-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

6. **Test the EFS provisioner:**

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/pod.yaml
kubectl apply -f pod.yaml
```

---

## 🌐 AWS Load Balancer Controller

> *This project was formerly known as the AWS ALB Ingress Controller.*

### 📚 Documentation

* [Project Homepage](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)
* [Installation Guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/)

### 📝 Steps

1. **Download and create IAM policy:**

```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

2. **Create IAM Role and Kubernetes Service Account:**

Replace placeholders with your values:

```bash
eksctl create iamserviceaccount \
  --cluster=<CLUSTER_NAME> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-1 \
  --approve
```

3. **Add the EKS Helm chart repo and install the controller:**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=education-eks-PiLc7s9m \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

## ✅ Verification & Next Steps

* Test LoadBalancer service or Ingress controller
* Deploy applications and expose using `Ingress` resources
* Monitor your cluster using CloudWatch or Prometheus/Grafana
* Secure IAM Roles for workloads (IRSA)
* Scale using Cluster Autoscaler or Karpenter

---

## 📚 Additional References

* [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
* [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
* [AWS Load Balancer Controller GitHub](https://github.com/kubernetes-sigs/aws-load-balancer-controller)
* [AWS EFS CSI Driver GitHub](https://github.com/kubernetes-sigs/aws-efs-csi-driver)

