# Exercise 7 - AWS Load Balancer Controller & ALB Ingress on Amazon EKS

## Objective

Deploy a sample application on Amazon EKS and expose it to the internet using the AWS Load Balancer Controller (ALB Ingress).

---

# Architecture

```
                Internet
                    │
                    ▼
          AWS Application Load Balancer
                    │
              Kubernetes Ingress
                    │
             ClusterIP Service
                    │
         -------------------------
         |                       |
         ▼                       ▼
     Pod (NodeJS)          Pod (NodeJS)
                    │
                    ▼
               Amazon EKS
```

---

# Prerequisites

- AWS CLI
- kubectl
- eksctl
- Helm
- Existing Amazon EKS Cluster
- Metrics Server
- OIDC Provider Enabled

---

# Verify Cluster

```bash
kubectl get nodes -o wide

kubectl get deployment -A

kubectl get pods -A
```

---

# Verify OIDC

```bash
aws eks describe-cluster \
  --name devops-secrets \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer"
```

Expected Output

```
https://oidc.eks.us-east-1.amazonaws.com/id/xxxxxxxx
```

---

# Install AWS Load Balancer Controller

Add Helm Repository

```bash
helm repo add eks https://aws.github.io/eks-charts

helm repo update
```

Install Controller

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=devops-secrets \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set region=us-east-1 \
--set vpcId=<YOUR_VPC_ID>
```

---

# Verify Controller

```bash
kubectl get pods -n kube-system

kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

---

# Deploy Application

```bash
kubectl apply -f k8s/demo-app/deployment.yaml

kubectl apply -f k8s/demo-app/service.yaml

kubectl apply -f k8s/demo-app/ingress.yaml
```

---

# Verify Resources

Pods

```bash
kubectl get pods
```

Service

```bash
kubectl get svc
```

Ingress

```bash
kubectl get ingress
```

---

# Common Troubleshooting

## Check Ingress

```bash
kubectl describe ingress demo-webapp
```

---

## Check Controller Logs

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller -f
```

---

## Verify Service Account

```bash
kubectl get sa aws-load-balancer-controller \
-n kube-system -o yaml
```

---

## Verify Deployment Uses Service Account

```bash
kubectl get deployment aws-load-balancer-controller \
-n kube-system \
-o yaml
```

---

# Verify Cluster VPC

```bash
aws eks describe-cluster \
--name devops-secrets \
--region us-east-1 \
--query "cluster.resourcesVpcConfig"
```

---

# Verify Subnets

```bash
aws ec2 describe-subnets \
--subnet-ids \
subnet-xxxxxxxx \
subnet-yyyyyyyy
```

---

# Required Public Subnet Tags

Both public subnets must contain:

```
kubernetes.io/role/elb = 1

kubernetes.io/cluster/devops-secrets = shared
```

If missing:

```bash
aws ec2 create-tags \
--resources subnet-xxxxxxxx \
--tags Key=kubernetes.io/role/elb,Value=1

aws ec2 create-tags \
--resources subnet-xxxxxxxx \
--tags Key=kubernetes.io/cluster/devops-secrets,Value=shared
```

---

# Restart Controller

```bash
kubectl rollout restart deployment aws-load-balancer-controller \
-n kube-system

kubectl rollout status deployment aws-load-balancer-controller \
-n kube-system
```

---

# Recreate Ingress

Delete

```bash
kubectl delete ingress demo-webapp
```

Apply

```bash
kubectl apply -f k8s/demo-app/ingress.yaml
```

Watch

```bash
kubectl get ingress -w
```

Expected

```
NAME          CLASS   HOSTS   ADDRESS
demo-webapp   alb     *       k8s-demo-xxxxxxxx.us-east-1.elb.amazonaws.com
```

---

# Test Application

Browser

```
http://<ALB-DNS>
```

or

```bash
curl http://<ALB-DNS>
```

---

# Useful Debug Commands

Nodes

```bash
kubectl get nodes
```

Pods

```bash
kubectl get pods
```

Services

```bash
kubectl get svc
```

Ingress

```bash
kubectl get ingress
```

Describe Ingress

```bash
kubectl describe ingress demo-webapp
```

Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

Controller Logs

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller -f
```

---

# Issues Faced

### Issue 1

```
couldn't auto-discover subnets
```

### Cause

Missing subnet tags.

### Fix

Added:

```
kubernetes.io/role/elb=1
kubernetes.io/cluster/devops-secrets=shared
```

---

### Issue 2

```
AccessDenied:
elasticloadbalancing:DescribeLoadBalancers
```

### Cause

AWS Load Balancer Controller was using the Node IAM Role instead of an IAM Role for Service Accounts (IRSA).

### Fix

- Configured IRSA.
- Annotated the Service Account with the IAM Role ARN.
- Restarted the AWS Load Balancer Controller.

---

# Skills Learned

- Amazon EKS Networking
- AWS Load Balancer Controller
- Kubernetes Ingress
- Application Load Balancer (ALB)
- IAM Roles for Service Accounts (IRSA)
- Helm
- Kubernetes Services
- Kubernetes Networking
- AWS Subnet Tagging
- Troubleshooting ALB Provisioning
- Kubernetes Debugging

---

# Result

Successfully deployed a public application on Amazon EKS using:

- Amazon EKS
- AWS Load Balancer Controller
- Application Load Balancer
- Kubernetes Ingress
- ClusterIP Service
- IRSA Authentication
- Helm

Application became publicly accessible through the generated ALB DNS name.