# Exercise 8 - External Secrets Operator with AWS Secrets Manager on Amazon EKS

## Objective

Integrate Amazon EKS with AWS Secrets Manager using External Secrets Operator (ESO) and IAM Roles for Service Accounts (IRSA) so that Kubernetes secrets are automatically synchronized from AWS Secrets Manager.

---

# Architecture

```
AWS Secrets Manager
        │
        ▼
IAM Policy
        │
        ▼
IAM Role (IRSA)
        │
        ▼
Service Account
        │
        ▼
External Secrets Operator
        │
        ▼
ClusterSecretStore
        │
        ▼
ExternalSecret
        │
        ▼
Kubernetes Secret
        │
        ▼
Application
```

---

# Prerequisites

- AWS CLI configured
- kubectl installed
- eksctl installed
- Helm installed
- Amazon EKS Cluster running
- OIDC Provider associated with cluster

---

# Step 1 - Associate IAM OIDC Provider

```powershell
eksctl utils associate-iam-oidc-provider `
  --cluster devops `
  --region us-east-1 `
  --approve
```

Verify

```powershell
aws eks describe-cluster `
  --name devops `
  --region us-east-1 `
  --query "cluster.identity.oidc.issuer"
```

---

# Step 2 - Create Secret in AWS Secrets Manager

Create a file named `secret.json`

```json
{
    "username": "admin",
    "password": "supersecret123"
}
```

Create the secret

```powershell
aws secretsmanager create-secret `
  --name demo-secret `
  --secret-string file://secret.json
```

Verify

```powershell
aws secretsmanager get-secret-value `
  --secret-id demo-secret
```

---

# Step 3 - Create IAM Policy

Create the policy

```powershell
aws iam create-policy `
  --policy-name ExternalSecretsPolicy `
  --policy-document file://iam/secrets-policy.json
```

If policy already exists

```powershell
aws iam list-policies --scope Local
```

---

# Step 4 - Install External Secrets Operator

Add Helm Repository

```powershell
helm repo add external-secrets https://charts.external-secrets.io

helm repo update
```

Create Namespace

```powershell
kubectl create namespace external-secrets
```

Install Operator

```powershell
helm install external-secrets external-secrets/external-secrets `
    -n external-secrets
```

Verify

```powershell
kubectl get pods -n external-secrets
```

---

# Step 5 - Create IAM Service Account (IRSA)

Attach IAM Policy to Kubernetes Service Account

```powershell
eksctl create iamserviceaccount `
    --cluster devops `
    --namespace external-secrets `
    --name external-secrets `
    --attach-policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/ExternalSecretsPolicy `
    --override-existing-serviceaccounts `
    --approve
```

Verify

```powershell
kubectl get sa external-secrets -n external-secrets -o yaml
```

Expected Annotation

```yaml
eks.amazonaws.com/role-arn:
```

---

# Step 6 - Restart External Secrets Pods

```powershell
kubectl rollout restart deployment external-secrets `
    -n external-secrets
```

Verify

```powershell
kubectl get pods -n external-secrets
```

---

# Step 7 - Create ClusterSecretStore

Apply

```powershell
kubectl apply -f k8s/cluster-secret-store.yaml
```

Verify

```powershell
kubectl describe clustersecretstore aws-secretsmanager
```

Expected

```
Status:
Ready = True
```

---

# Step 8 - Create External Secret

Apply

```powershell
kubectl apply -f k8s/external-secret.yaml
```

Verify

```powershell
kubectl get externalsecret
```

Expected

```
READY
True
```

---

# Step 9 - Verify Kubernetes Secret

```powershell
kubectl get secret
```

Describe

```powershell
kubectl describe secret demo-secret
```

View Secret

```powershell
kubectl get secret demo-secret -o yaml
```

Decode Secret

```powershell
kubectl get secret demo-secret -o jsonpath="{.data.username}" | base64 -d

kubectl get secret demo-secret -o jsonpath="{.data.password}" | base64 -d
```

---

# Files Used

```
iam/
└── secrets-policy.json

k8s/
├── cluster-secret-store.yaml
└── external-secret.yaml

secret.json
```

---

# Troubleshooting

## ClusterSecretStore not Ready

Verify Service Account

```powershell
kubectl get sa external-secrets -n external-secrets -o yaml
```

Verify IRSA

```powershell
eksctl get iamserviceaccount --cluster devops
```

---

## External Secret shows SecretSyncedError

Verify AWS Secret

```powershell
aws secretsmanager get-secret-value `
    --secret-id demo-secret
```

Ensure the secret is valid JSON

Correct

```json
{
  "username":"admin",
  "password":"supersecret123"
}
```

Incorrect

```
{username:admin,password:supersecret123}
```

---

## Verify Everything

External Secrets

```powershell
kubectl get externalsecret
```

Cluster Secret Store

```powershell
kubectl get clustersecretstore
```

Secrets

```powershell
kubectl get secret
```

Pods

```powershell
kubectl get pods -n external-secrets
```

---

# Learning Outcomes

- Learned IAM Roles for Service Accounts (IRSA)
- Integrated AWS Secrets Manager with Amazon EKS
- Installed External Secrets Operator using Helm
- Created ClusterSecretStore
- Created ExternalSecret
- Automatically synchronized AWS Secrets Manager secrets into Kubernetes Secrets
- Verified secure secret management without storing secrets inside Git repositories

---

# Result

Successfully synchronized secrets from AWS Secrets Manager to Kubernetes using External Secrets Operator.

```
AWS Secrets Manager
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
        │
        ▼
Application
```