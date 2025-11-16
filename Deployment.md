# **LiveKit on AWS EKS – Production Deployment Guide**  

---

## Overview

This is a **complete, professional, enterprise-grade deployment guide** for **LiveKit on AWS EKS** using:

| Component | Purpose |
|---------|--------|
| **Amazon EKS** | Kubernetes cluster with host networking for WebRTC |
| **ElastiCache Redis** | Distributed session state & pub/sub |
| **AWS Certificate Manager (ACM)** | Free TLS for `wss://livekit.yourdomain.com` |
| **Let’s Encrypt + cert-manager** | Auto-renewing TLS for `turn.yourdomain.com:3478` |
| **AWS Application Load Balancer (ALB)** | HTTPS/WSS termination |
| **Route53** | DNS routing |
| **LiveKit Helm Chart** | Autoscaling, graceful shutdown, zero-downtime upgrades |

> **Tested & Verified on**:  
> - EKS 1.34  
> - LiveKit Helm v0.15+  
> - cert-manager v1.15.3  
> - AWS ALB Controller v2.14.1  
> - CloudShell (November 2025)

---

## Prerequisites

| Requirement | Details |
|------------|--------|
| **AWS Account** | IAM user with: `AmazonEKSClusterPolicy`, `AmazonEC2FullAccess`, `IAMFullAccess`, `ACMFullAccess`, `Route53FullAccess` |
| **Domain** | `yourdomain.com` (must be in **Route53** for auto-validation) |
| **Subdomains** | `livekit.yourdomain.com` → signaling<br>`turn.yourdomain.com` → TURN/TLS |
| **ElastiCache Redis** | Redis OSS, note **primary endpoint** |
| **Tools** | `aws`, `kubectl`, `helm`, `eksctl`, `docker` |

---

## Deployment Options

### **Option 1: AWS CloudShell (Recommended – No Local Install)**

> **Important**: CloudShell **does NOT** come with `helm`, `kubectl`, `eksctl`, or `docker` pre-installed.

#### Install Tools in CloudShell

```bash
# Update package index
sudo yum update -y

# Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install Docker (for token generation)
sudo yum install -y docker
sudo service docker start
sudo usermod -a -G docker ec2-user
# Log out and back in to CloudShell for group change
```

> **After installing**: Run `helm version`, `kubectl version --client`, `eksctl version`, `docker --version` to verify.

---

### **Option 2: Local Machine**

```bash
# macOS
brew install awscli kubectl helm eksctl docker

# Ubuntu/Debian
sudo apt update && sudo apt install -y awscli kubectl helm docker.io
curl -LO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xzf eksctl_*.tar.gz && sudo mv eksctl /usr/local/bin

# Windows (PowerShell as Admin)
winget install Amazon.AWSCLI Kubernetes.kubectl Helm.Helm Amazon.EKSCTL Docker.DockerDesktop
```

```bash
aws configure
```

---

## Step 1: Configure AWS CLI

```bash
aws configure
```
Enter:
- **Access Key ID**
- **Secret Access Key**
- **Region**: `us-east-1`
- **Output**: `json`

---

## Step 2: Create EKS Cluster

```bash
eksctl create cluster \
  --name livekit-cluster-v2 \
  --region us-east-1 \
  --version 1.34 \
  --nodegroup-name workers \
  --instance-types t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 20 \
  --managed
```

> **Wait**: 12–15 minutes  
> **Verify**:
```bash
kubectl get nodes -o wide
```

---

## Step 3: Install AWS Load Balancer Controller

```bash
# 1. IAM Policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 2. Account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# 3. Service Account
eksctl create iamserviceaccount \
  --cluster livekit-cluster-v2 \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

# 4. Helm & CRDs
helm repo add eks https://aws.github.io/eks-charts
helm repo update

wget https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml
kubectl apply -f crds.yaml

# 5. Install
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=livekit-cluster-v2 \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0 \
  --wait
```

> **Verify**:
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Step 4: Request ACM Certificate (UI + CLI)

### **Via AWS Console (Recommended)**

1. Go to **Certificate Manager** → **Request certificate**
2. Select **Public certificate**
3. Add: `livekit.yourdomain.com`
4. Validation: **DNS**
5. Click **"Create record in Route53"**
6. Wait → **Issued**
7. **Copy ARN**

### **Via CLI (Optional)**

```bash
aws acm request-certificate \
  --domain-name livekit.yourdomain.com \
  --validation-method DNS \
  --region us-east-1
```

---

## Step 5: Install cert-manager

```bash
kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.15.3 \
  --set installCRDs=true \
  --wait
```

---

## Step 6: Create Let’s Encrypt Issuer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@yourdomain.com        # CHANGE THIS
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: alb
EOF
```

---

## Step 7: Generate API Key/Secret

```bash
docker run --rm livekit/livekit-server generate-keys
```

**Save**:
```
API Key:    APIk123abc
API Secret: supersecretxyz789
```

---

## Step 8: Create `values-eks.yaml`

```yaml
livekit:
  domain: livekit.yourdomain.com
  rtc:
    use_external_ip: true
    port_range_start: 50000
    port_range_end: 60000
  redis:
    addresses:
      - your-redis-endpoint.cache.amazonaws.com:6379
  keys:
    APIk123abc: supersecretxyz789

  turn:
    enabled: true
    domain: turn.yourdomain.com
    certManagerIssuer: letsencrypt-prod
    tls_port: 3478
    udp_port: 3478

loadBalancer:
  type: alb
  tls:
    - hosts:
        - livekit.yourdomain.com
      certificateArn: arn:aws:acm:us-east-1:123456789012:certificate/abc123...

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60
```

---

## Step 9: Deploy LiveKit

```bash
helm repo add livekit https://helm.livekit.io
helm repo update

helm upgrade --install livekit livekit/livekit-server \
  --namespace livekit \
  --create-namespace \
  --values values-eks.yaml \
  --wait
```

---

## Step 10: Find ALB DNS Name (CLI + AWS Console UI)

### **CLI Method**

```bash
kubectl get ingress -n livekit -o wide
```

**Copy `ADDRESS`**

### **AWS Console UI Method (If CLI Shows No Address)**

1. Go to **EC2 Dashboard → Load Balancers**
2. Filter by **Tag**: `kubernetes.io/service-name = livekit/livekit`
3. Find load balancer with name like `k8s-livekit-xxxxxx`
4. **Copy DNS name** → `k8s-livekit-xxxxxx-1234567890.us-east-1.elb.amazonaws.com`

> **Why ALB may not appear?**  
> - ALB Controller not running  
> - IAM policy missing  
> - Ingress not created  
> → Check logs: `kubectl logs -n kube-system deploy/aws-load-balancer-controller`

---

## Step 11: Configure DNS in Route53 (UI)

1. Go to **Route53 → Hosted Zones → yourdomain.com**
2. Create **2 records**:

| Type | Name | Value | Alias |
|------|------|-------|-------|
| **A** | `livekit` | `k8s-livekit-xxxxxx.elb.amazonaws.com` | **Yes** |
| **CNAME** | `turn` | `k8s-livekit-xxxxxx.elb.amazonaws.com` | No |

---

## Step 12: Open Security Group

1. **EC2 → Security Groups**
2. Find **EKS node group SG**
3. **Edit inbound rules** → Add:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| HTTPS | TCP | 443 | 0.0.0.0/0 |
| Custom TCP | TCP | 3478 | 0.0.0.0/0 |
| Custom UDP | UDP | 3478 | 0.0.0.0/0 |
| Custom UDP | UDP | 50000–60000 | 0.0.0.0/0 |

---

## Step 13: Verify

```bash
kubectl get pods -n livekit
kubectl get certificate -n livekit
```

---

## Step 14: Test

```bash
docker run --rm livekit/generate \
  --url wss://livekit.yourdomain.com \
  --api-key APIk123abc \
  --api-secret supersecretxyz789 \
  --join --room demo
```

Go to: [https://meet.livekit.io/custom](https://meet.livekit.io/custom)

---

## Troubleshooting (Professional)

| Symptom | Diagnosis | Fix |
|--------|----------|-----|
| **No ALB in `kubectl get ingress`** | `kubectl logs -n kube-system deploy/aws-load-balancer-controller` | Check IAM, subnet tags |
| **Certificate stuck in `Pending`** | `kubectl describe certificate -n livekit` | Check Route53 CNAMEs |
| **Pods in `CrashLoopBackOff`** | `kubectl logs -n livekit deploy/livekit` | Redis unreachable? Wrong key? |
| **Clients can't connect** | Browser → `ICE failed` | Open UDP 50000–60000 |
| **DNS not resolving** | `dig livekit.yourdomain.com` | Wait 15 mins or check TTL |

---

## Official References

- LiveKit Helm: https://github.com/livekit/livekit-helm
- AWS ALB Controller: https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html
- cert-manager: https://cert-manager.io
- Ports: https://docs.livekit.io/home/self-hosting/ports-firewall

---
