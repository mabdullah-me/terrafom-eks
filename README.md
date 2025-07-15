#  Provision an EKS Cluster

This repo is a companion repo to the [Provision an EKS Cluster tutorial](https://developer.hashicorp.com/terraform/tutorials/kubernetes/eks), containing
Terraform configuration files to provision an EKS cluster on AWS.



clone terrafor code and run the below commands
terrafrom init
terraform plan
terraform apply


# Update kubeconfig for EKS cluster
aws eks --region us-east-1 update-kubeconfig --name education-eks-PiLc7s9m

# Setup EFS storage class
https://kubernetes.io/docs/concepts/storage/storage-classes/#aws-efs
dynamic provisioning
https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/dynamic_provisioning/README.md
aws efs
https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html

- Get storageclass manifest
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml

- Get the filesystem name
aws efs describe-file-systems --query "FileSystems[*].FileSystemId" --output text

Output e.g: fs-0d4a5eaa988b19bca

edit storage.yml and update fileSystemId parameter

kubectl apply -f k8s-manifests/storageclass.yaml

- Make EFS filesystem as Default
https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/
kubectl get storageclass
Mark the default StorageClass as non-default
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
Mark efs-sc StorageClass as default
kubectl patch storageclass efs-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

- To verify efs storage class run the test pod
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/pod.yaml

# AWS Load Balancer Controller ( This project was formerly known as "AWS ALB Ingress Controller" )
https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/
https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/


- Download and create an IAM policy for the LBC 
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

- Create an IAM role and Kubernetes ServiceAccount for the LBC. Use the ARN from the previous step.
eksctl create iamserviceaccount \
--cluster=<cluster-name> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region <region-code> \
--approve

- Add the EKS chart repo to Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller