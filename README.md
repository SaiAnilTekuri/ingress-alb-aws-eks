# Deploying an Application on AWS EKS Fargate with AWS Load Balancer Controller

This guide provides step-by-step instructions to set up an AWS EKS cluster with Fargate, deploy an application, and configure an AWS Load Balancer Controller for ingress management.

## Prerequisites
Ensure that you have the following tools installed on your local machine:
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [eksctl](https://eksctl.io/)
- [Helm](https://helm.sh/docs/intro/install/)

---

## Steps to Set Up AWS EKS with Fargate and Deploy an Application

### 1. Install Required Tools
Ensure AWS CLI, `kubectl`, `eksctl`, and `helm` are installed.

```sh
aws --version
kubectl version --client
eksctl version
helm version
```

### 2. Create an AWS EKS Cluster
Create an EKS cluster using `eksctl`. This will provision an EKS cluster with Fargate as the compute engine.

```sh
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

### 3. Update kubeconfig
Update your local kubeconfig file to interact with the newly created cluster.

```sh
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

### 4. Create a Fargate Profile for a Custom Namespace
Create a Fargate profile to ensure workloads in the specified namespace run on AWS Fargate.

```sh
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

### 5. Deploy the Application
Deploy the sample 2048 game application along with its services and ingress definitions.

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

### 6. Associate AWS OIDC Provider
Associate an IAM OpenID Connect (OIDC) provider with the EKS cluster to enable IAM role-based authentication for Kubernetes workloads.

```sh
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

### 7. Download the IAM Policy File
Download the IAM policy file required for the AWS Load Balancer Controller.

```sh
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### 8. Create an IAM Policy
Create an IAM policy from the downloaded policy file.

```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 9. Create an IAM Service Account
Create an IAM service account and associate it with an IAM role and the IAM policy created in the previous step.

```sh
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<amazon-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 10. Install AWS Load Balancer Controller
Install the AWS Load Balancer Controller using Helm. This controller creates AWS Load Balancers for Kubernetes services.

```sh
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<vpc-id>
```

---

## Verification
After the setup, verify the deployment using the following commands:

```sh
kubectl get pods -n kube-system
kubectl get deployments -n game-2048
kubectl get services -n game-2048
kubectl get ingress -n game-2048
```

You should see the application running and an external load balancer created.

---

## Cleanup
To delete all created resources, run:

```sh
eksctl delete cluster --name demo-cluster --region us-east-1
```

---

## References
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)

