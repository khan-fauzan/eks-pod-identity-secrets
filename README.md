# Multi-Tenant EKS with AWS Secrets Manager Implementation Guide

## Note
This repo is for training purposes only and this code should not be used for production workloads

## Overview
This guide demonstrates how to securely integrate AWS Secrets Manager with Amazon EKS in a multi-tenant environment, ensuring proper isolation and access controls.

## Architecture Components
- **AWS Secrets Manager**: Centralized secret storage
- **AWS Secrets Store CSI Driver**: Bridge between Kubernetes and Secrets Manager
- **Amazon EKS Pod Identity**: Secure authentication (replaces IRSA)
- **Kubernetes Namespaces**: Tenant isolation
- **Service Accounts**: Per-tenant identity management

## Step 1: Environment Setup and Prerequisites

### 1.1 Install Required Tools
```bash
# Install AWS CLI (if not already installed)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 1.2 Verify EKS Cluster Access
```bash
# Update kubeconfig
aws eks update-kubeconfig --region <your-region> --name <your-cluster-name>

# Verify connection
kubectl get nodes
```

## Step 2: Create Multi-Tenant Namespace Structure

### 2.1 Create Tenant Namespaces
```bash
# Create namespaces for different tenants
kubectl create namespace tenant-a
kubectl create namespace tenant-b
kubectl create namespace tenant-c

# Label namespaces for better organization
kubectl label namespace tenant-a tenant=tenant-a
kubectl label namespace tenant-b tenant=tenant-b
kubectl label namespace tenant-c tenant=tenant-c
```

### 2.2 Apply Network Policies (Optional but Recommended)
```yaml
# networkpolicy-tenant-isolation.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-isolation
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: tenant-a
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tenant: tenant-a
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

```bash
# Apply network policies for each tenant
kubectl apply -f networkpolicy-tenant-isolation.yaml
# Repeat for tenant-b and tenant-c namespaces
```

## Step 3: Install AWS Secrets Store CSI Driver

### 3.1 Add the Secrets Store CSI Driver
```bash
# Add the helm repository
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update

# Install the CSI driver
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true \
  --set enableSecretRotation=true
```

### 3.2 Install AWS Provider for Secrets Store CSI Driver
```bash
# Apply the AWS provider
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

### 3.3 Verify Installation
```bash
# Check if pods are running
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=csi-secrets-store-provider-aws
```

## Step 4: Create AWS Secrets Manager Secrets

### 4.1 Create Secrets for Each Tenant
```bash
# Create secrets for tenant-a
aws secretsmanager create-secret \
  --name "tenant-a/database-credentials" \
  --description "Database credentials for tenant A" \
  --secret-string '{"username":"tenant_a_user","password":"secure_password_123"}' \
  --region <your-region>

aws secretsmanager create-secret \
  --name "tenant-a/api-keys" \
  --description "API keys for tenant A" \
  --secret-string '{"api_key":"tenant-a-api-key-xyz","webhook_secret":"webhook-secret-abc"}' \
  --region <your-region>

# Create secrets for tenant-b
aws secretsmanager create-secret \
  --name "tenant-b/database-credentials" \
  --description "Database credentials for tenant B" \
  --secret-string '{"username":"tenant_b_user","password":"secure_password_456"}' \
  --region <your-region>

# Create secrets for tenant-c
aws secretsmanager create-secret \
  --name "tenant-c/database-credentials" \
  --description "Database credentials for tenant C" \
  --secret-string '{"username":"tenant_c_user","password":"secure_password_789"}' \
  --region <your-region>
```

### 4.2 Tag Secrets for Better Organization
```bash
# Tag secrets with tenant information
aws secretsmanager tag-resource \
  --secret-id "tenant-a/database-credentials" \
  --tags Key=Tenant,Value=tenant-a Key=Environment,Value=production

aws secretsmanager tag-resource \
  --secret-id "tenant-b/database-credentials" \
  --tags Key=Tenant,Value=tenant-b Key=Environment,Value=production
```

## Step 5: Enable and Configure EKS Pod Identity

### 5.1 Enable Pod Identity Agent on EKS Cluster
```bash
# Set environment variables
export CLUSTER_NAME=<your-cluster-name>
export REGION=<your-region>

# Enable Pod Identity Agent add-on
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name eks-pod-identity-agent \
  --region $REGION

# Wait for add-on to be active
aws eks describe-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name eks-pod-identity-agent \
  --region $REGION \
  --query "addon.status"
```

### 5.2 Create IAM Policies for Each Tenant
```bash
# Create policy for tenant-a
cat > tenant-a-secrets-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": [
                "arn:aws:secretsmanager:<your-region>:<your-account-id>:secret:tenant-a/*"
            ]
        }
    ]
}
EOF

aws iam create-policy \
  --policy-name TenantASecretsPolicy \
  --policy-document file://tenant-a-secrets-policy.json

# Create policy for tenant-b
cat > tenant-b-secrets-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": [
                "arn:aws:secretsmanager:<your-region>:<your-account-id>:secret:tenant-b/*"
            ]
        }
    ]
}
EOF

aws iam create-policy \
  --policy-name TenantBSecretsPolicy \
  --policy-document file://tenant-b-secrets-policy.json

# Create policy for tenant-c
cat > tenant-c-secrets-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": [
                "arn:aws:secretsmanager:<your-region>:<your-account-id>:secret:tenant-c/*"
            ]
        }
    ]
}
EOF

aws iam create-policy \
  --policy-name TenantCSecretsPolicy \
  --policy-document file://tenant-c-secrets-policy.json
```

### 5.3 Create IAM Roles for Pod Identity
```bash
# Create trust policy for Pod Identity
cat > pod-identity-trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "pods.eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
EOF

# Create IAM role for tenant-a
aws iam create-role \
  --role-name TenantASecretsRole \
  --assume-role-policy-document file://pod-identity-trust-policy.json \
  --description "Pod Identity role for Tenant A secrets access"

# Attach policy to tenant-a role
aws iam attach-role-policy \
  --role-name TenantASecretsRole \
  --policy-arn arn:aws:iam::<your-account-id>:policy/TenantASecretsPolicy

# Create IAM role for tenant-b
aws iam create-role \
  --role-name TenantBSecretsRole \
  --assume-role-policy-document file://pod-identity-trust-policy.json \
  --description "Pod Identity role for Tenant B secrets access"

# Attach policy to tenant-b role
aws iam attach-role-policy \
  --role-name TenantBSecretsRole \
  --policy-arn arn:aws:iam::<your-account-id>:policy/TenantBSecretsPolicy

# Create IAM role for tenant-c
aws iam create-role \
  --role-name TenantCSecretsRole \
  --assume-role-policy-document file://pod-identity-trust-policy.json \
  --description "Pod Identity role for Tenant C secrets access"

# Attach policy to tenant-c role
aws iam attach-role-policy \
  --role-name TenantCSecretsRole \
  --policy-arn arn:aws:iam::<your-account-id>:policy/TenantCSecretsPolicy
```

## Step 6: Create Kubernetes Service Accounts and Pod Identity Associations

### 6.1 Create Service Accounts
```yaml
# tenant-a-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-a-service-account
  namespace: tenant-a
  labels:
    app.kubernetes.io/name: tenant-a-service-account
---
# tenant-b-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-b-service-account
  namespace: tenant-b
  labels:
    app.kubernetes.io/name: tenant-b-service-account
---
# tenant-c-service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-c-service-account
  namespace: tenant-c
  labels:
    app.kubernetes.io/name: tenant-c-service-account
```

```bash
# Apply service accounts
kubectl apply -f tenant-a-service-account.yaml
kubectl apply -f tenant-b-service-account.yaml
kubectl apply -f tenant-c-service-account.yaml
```

### 6.2 Create Pod Identity Associations
```bash
# Get your AWS account ID
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create Pod Identity association for tenant-a
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace tenant-a \
  --service-account tenant-a-service-account \
  --role-arn arn:aws:iam::$ACCOUNT_ID:role/TenantASecretsRole \
  --region $REGION

# Create Pod Identity association for tenant-b
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace tenant-b \
  --service-account tenant-b-service-account \
  --role-arn arn:aws:iam::$ACCOUNT_ID:role/TenantBSecretsRole \
  --region $REGION

# Create Pod Identity association for tenant-c
aws eks create-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --namespace tenant-c \
  --service-account tenant-c-service-account \
  --role-arn arn:aws:iam::$ACCOUNT_ID:role/TenantCSecretsRole \
  --region $REGION
```

### 6.3 Verify Pod Identity Associations
```bash
# List all Pod Identity associations for the cluster
aws eks list-pod-identity-associations \
  --cluster-name $CLUSTER_NAME \
  --region $REGION

# Get details for specific association (use association-id from list command)
aws eks describe-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --association-id <association-id> \
  --region $REGION
```

## Step 7: Create SecretProviderClass Resources

### 7.1 Create SecretProviderClass for Tenant A
```yaml
# tenant-a-secret-provider-class.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: tenant-a-secrets
  namespace: tenant-a
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "tenant-a/database-credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: "username"
            objectAlias: "db-username"
          - path: "password"
            objectAlias: "db-password"
      - objectName: "tenant-a/api-keys"
        objectType: "secretsmanager"
        jmesPath:
          - path: "api_key"
            objectAlias: "api-key"
          - path: "webhook_secret"
            objectAlias: "webhook-secret"
  secretObjects:
  - secretName: tenant-a-database-secret
    type: Opaque
    data:
    - objectName: db-username
      key: username
    - objectName: db-password
      key: password
  - secretName: tenant-a-api-secret
    type: Opaque
    data:
    - objectName: api-key
      key: api_key
    - objectName: webhook-secret
      key: webhook_secret
```

```bash
kubectl apply -f tenant-a-secret-provider-class.yaml
```

## Step 8: Deploy Sample Applications

### 8.1 Create Deployment for Tenant A
```yaml
# tenant-a-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tenant-a-app
  namespace: tenant-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tenant-a-app
  template:
    metadata:
      labels:
        app: tenant-a-app
    spec:
      serviceAccountName: tenant-a-service-account
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: tenant-a-database-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: tenant-a-database-secret
              key: password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: tenant-a-api-secret
              key: api_key
        volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets-store"
          readOnly: true
      volumes:
      - name: secrets-store
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "tenant-a-secrets"
```

```bash
kubectl apply -f tenant-a-deployment.yaml
```

## Step 9: Verification and Testing

### 9.1 Verify Secret Mounting
```bash
# Check if pods are running
kubectl get pods -n tenant-a

# Check mounted secrets
kubectl exec -n tenant-a deployment/tenant-a-app -- ls -la /mnt/secrets-store

# Verify environment variables
kubectl exec -n tenant-a deployment/tenant-a-app -- env | grep -E "(DB_|API_)"

# Check Kubernetes secrets were created
kubectl get secrets -n tenant-a
```

### 9.2 Test Secret Access and Pod Identity
```bash
# Check if pods are running
kubectl get pods -n tenant-a

# Check mounted secrets
kubectl exec -n tenant-a deployment/tenant-a-app -- ls -la /mnt/secrets-store

# Verify environment variables
kubectl exec -n tenant-a deployment/tenant-a-app -- env | grep -E "(DB_|API_)"

# Check Kubernetes secrets were created
kubectl get secrets -n tenant-a

# Test Pod Identity - check AWS identity assumed by pod
kubectl exec -n tenant-a deployment/tenant-a-app -- aws sts get-caller-identity

# Verify secret content (be careful in production)
kubectl get secret tenant-a-database-secret -n tenant-a -o yaml

# Test tenant isolation - tenant-a should NOT be able to access tenant-b secrets
kubectl exec -n tenant-a deployment/tenant-a-app -- aws secretsmanager get-secret-value --secret-id tenant-b/database-credentials --region $REGION 2>&1 || echo "Access denied - this is expected for proper tenant isolation"

# Test that tenant-b can access their own secrets (if tenant-b is deployed)
kubectl exec -n tenant-b deployment/tenant-b-app -- aws secretsmanager get-secret-value --secret-id tenant-b/database-credentials --region $REGION 2>/dev/null && echo "Tenant B can access their secrets - this is expected"
```

## Step 10: Monitoring and Logging

### 10.1 Set up CloudWatch Logging
```bash
# Enable container insights (if not already enabled)
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/$CLUSTER_NAME/;s/{{region_name}}/$REGION/" | kubectl apply -f -
```

### 10.2 Monitor Secret Access
```bash
# Create CloudWatch log group for secrets access
aws logs create-log-group --log-group-name /aws/eks/secrets-manager-access

# Set up custom metrics (optional)
```

## Step 11: Security Best Practices

### 11.1 Enable Secret Rotation
```bash
# Enable automatic rotation for secrets
aws secretsmanager update-secret \
  --secret-id "tenant-a/database-credentials" \
  --rotation-rules AutomaticallyAfterDays=30
```

### 11.2 Implement Resource Quotas
```yaml
# tenant-a-resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "10"
    secrets: "10"
```

### 11.3 Set up RBAC
```yaml
# tenant-a-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: tenant-a
  name: tenant-a-role
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps", "pods"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-a-rolebinding
  namespace: tenant-a
subjects:
- kind: ServiceAccount
  name: tenant-a-service-account
  namespace: tenant-a
roleRef:
  kind: Role
  name: tenant-a-role
  apiGroup: rbac.authorization.k8s.io
```

## Step 12: Troubleshooting

### 12.1 Common Issues and Solutions

**Issue: Pod fails to mount secrets**
```bash
# Check CSI driver pods
kubectl get pods -n kube-system | grep secrets-store

# Check logs
kubectl logs -n kube-system -l app=secrets-store-csi-driver

# Verify IAM permissions
aws sts get-caller-identity
```

**Issue: Pod fails to assume IAM role with Pod Identity**
```bash
# Check Pod Identity agent is running
kubectl get pods -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent

# Verify Pod Identity associations
aws eks list-pod-identity-associations --cluster-name $CLUSTER_NAME --region $REGION

# Check pod identity agent logs
kubectl logs -n kube-system -l app.kubernetes.io/name=eks-pod-identity-agent

# Verify service account exists and is correctly configured
kubectl describe sa tenant-a-service-account -n tenant-a
```

**Issue: Service account cannot assume role**
```bash
# Check if Pod Identity association exists
aws eks describe-pod-identity-association \
  --cluster-name $CLUSTER_NAME \
  --association-id <association-id> \
  --region $REGION

# Verify IAM role trust policy allows pods.eks.amazonaws.com
aws iam get-role --role-name TenantASecretsRole --query 'Role.AssumeRolePolicyDocument'

# Test role assumption manually
aws sts assume-role \
  --role-arn arn:aws:iam::$ACCOUNT_ID:role/TenantASecretsRole \
  --role-session-name test-session
```

**Issue: Secrets not accessible**
```bash
# Test AWS CLI access with Pod Identity
kubectl exec -n tenant-a deployment/tenant-a-app -- aws sts get-caller-identity
kubectl exec -n tenant-a deployment/tenant-a-app -- aws secretsmanager get-secret-value --secret-id tenant-a/database-credentials --region $REGION

# Check secret provider class
kubectl describe secretproviderclass tenant-a-secrets -n tenant-a

# Verify Pod Identity agent is working
kubectl get events -n tenant-a --field-selector reason=PodIdentityAssociation
```

## Conclusion

This implementation provides:
- **Tenant Isolation**: Each tenant has separate namespaces, IAM roles, and secret access
- **Security**: RBAC, network policies, and least-privilege access
- **Simplified Authentication**: Pod Identity eliminates the need for OIDC provider configuration
- **Scalability**: Easy to add new tenants by replicating the pattern
- **Monitoring**: CloudWatch integration for audit and monitoring
- **Automation**: Consistent deployment patterns across tenants

## Key Advantages of Pod Identity over IRSA:
- **Simpler Setup**: No need to configure OIDC providers or complex trust policies
- **Better Security**: Built-in credential rotation and shorter-lived tokens
- **Easier Management**: Centralized association management through EKS APIs
- **Improved Reliability**: Reduces dependency on external identity providers

For production environments, consider implementing:
- GitOps workflows for secret management
- External secret rotation mechanisms
- Advanced monitoring and alerting
- Backup and disaster recovery procedures
- Regular security audits and compliance checks
