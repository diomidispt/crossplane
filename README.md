# crossplane-demo

Provisions AWS infrastructure (S3 + EC2) from a local Kind cluster using Crossplane.
Follows a **platform-team / app-team** model: the platform team owns `bootstrap/` and `apis/`,
while each environment team owns their `environments/<env>/` folder.

## What Was Built

This repo was created to see Crossplane in action — the goal was to watch real AWS resources appear
and disappear by simply applying and deleting Kubernetes claims. We spun up multiple S3 buckets and
EC2 instances, then deleted them, and observed Crossplane reconcile everything through to AWS.

**Phase 1 — Storage**: defined an `ObjectStorage` abstraction and created several S3 buckets across
dev and prod to confirm the full lifecycle (create → observe in AWS → delete).

**Phase 2 — Compute**: added a `VirtualMachine` abstraction (EC2) and a standalone `SecurityGroup`.
The SG is decoupled from the VM so one security group can be shared across multiple instances.
We then launched and deleted EC2 instances the same way — pure YAML, no AWS console.

---

## How It Works

```
Claim  →  XR (composite)  →  Composition  →  Managed Resources  →  AWS
```

| Claim Kind       | Composite         | AWS Resources Created         |
|------------------|-------------------|-------------------------------|
| `SecurityGroup`  | `XSecurityGroup`  | SG + SSH IngressRule          |
| `ObjectStorage`  | `XObjectStorage`  | S3 Bucket                     |
| `VirtualMachine` | `XVirtualMachine` | EC2 Instance                  |

`SecurityGroup` is its own independent claim — one SG can be shared across many `VirtualMachine` claims.

---

## Repository Structure

```
crossplane-demo/
├── bootstrap/                        # Platform team: install once per cluster
│   ├── providers.yaml                #   AWS provider packages (s3 v2.5.3, ec2 v2.5.3)
│   ├── functions.yaml                #   function-patch-and-transform v0.10.4
│   └── provider-config.yaml          #   AWS credential binding
│
├── apis/                             # Platform team: XRDs + Compositions
│   ├── networking/
│   │   ├── security-group/
│   │   │   ├── definition.yaml       #   XSecurityGroup XRD
│   │   │   └── composition.yaml      #   → SG + IngressRule
│   │   └── vpc/                      #   (placeholder — VPC + Subnets + IGW)
│   ├── storage/
│   │   ├── bucket/
│   │   │   ├── definition.yaml       #   XObjectStorage XRD
│   │   │   └── composition.yaml      #   → S3 Bucket
│   │   └── database/                 #   (placeholder — RDS)
│   └── compute/
│       ├── instance/
│       │   ├── definition.yaml       #   XVirtualMachine XRD
│       │   └── composition.yaml      #   → EC2 Instance
│       └── cluster/                  #   (placeholder — EKS)
│
├── environments/                     # App/ops team: claims per environment
│   ├── dev/
│   │   ├── networking/security-group.yaml   # dev-sg  (pre-filled demo values*)
│   │   ├── storage/bucket.yaml              # dev-bucket
│   │   └── compute/instance.yaml           # dev-instance (pre-filled demo values*)
│   └── prod/
│       ├── networking/security-group.yaml   # prod-sg  (REPLACE_ placeholders)
│       ├── storage/bucket.yaml              # prod-bucket
│       └── compute/instance.yaml           # prod-instance (REPLACE_ placeholders)
│
└── secrets/
    └── aws-credentials.yaml.example  # Template — never commit real credentials
```

> **\* dev/ pre-filled values**: `environments/dev/` files contain VPC, subnet, AMI, and SG IDs
> from the original demo AWS account (eu-central-1). Replace them with values from your own
> account before applying (see [Gather AWS values](#gather-aws-values-one-time-per-environment)).

---

## Full Setup — From Zero

### 1. Install CLI tools (macOS)

```bash
brew install kind          # local Kubernetes clusters via Docker
brew install kubectl       # Kubernetes CLI
brew install helm          # Kubernetes package manager
brew install awscli        # AWS CLI (for looking up AMI IDs, VPC IDs, etc.)

# Crossplane CLI (optional — useful for `crossplane beta trace`)
brew install crossplane
```

### 2. Create a Kind cluster

```bash
kind create cluster --name crossplane-demo

# Verify the cluster is up
kubectl get nodes
# NAME                            STATUS   ROLES           AGE   VERSION
# crossplane-demo-control-plane   Ready    control-plane   ...   v1.35.0
```

### 3. Install Crossplane via Helm

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace \
  --version 2.2.1

# Wait for Crossplane to be ready
kubectl wait --for=condition=Available deployment/crossplane \
  -n crossplane-system --timeout=2m

kubectl get pods -n crossplane-system
# NAME                                       READY   STATUS
# crossplane-...                             1/1     Running
# crossplane-rbac-manager-...               1/1     Running
```

### 4. Install AWS Providers and the patch-and-transform Function

```bash
kubectl apply -f bootstrap/providers.yaml
kubectl apply -f bootstrap/functions.yaml

# Providers pull their images — takes ~2-3 min
kubectl wait --for=condition=Healthy provider/provider-aws-s3 --timeout=5m
kubectl wait --for=condition=Healthy provider/provider-aws-ec2 --timeout=5m
kubectl wait --for=condition=Healthy function/function-patch-and-transform --timeout=5m

kubectl get providers
# NAME                          INSTALLED   HEALTHY
# provider-aws-ec2              True        True
# provider-aws-s3               True        True
# upbound-provider-family-aws   True        True     ← auto-installed as dependency
```

### 5. Create the AWS Credentials Secret

The secret must live in `crossplane-system` and use the AWS credentials file format.

```bash
cp secrets/aws-credentials.yaml.example secrets/aws-credentials.yaml
```

Edit `secrets/aws-credentials.yaml` and fill in your credentials:
```yaml
stringData:
  credentials: |
    [default]
    aws_access_key_id = YOUR_ACCESS_KEY_ID
    aws_secret_access_key = YOUR_SECRET_ACCESS_KEY
```

Then apply (this file is gitignored — it will never be committed):
```bash
kubectl apply -f secrets/aws-credentials.yaml
```

Alternatively, create it directly without a file:
```bash
kubectl create secret generic aws-credentials \
  -n crossplane-system \
  --from-literal=credentials="$(printf '[default]\naws_access_key_id = YOURKEY\naws_secret_access_key = YOURSECRET')"
```

### 6. Apply the ProviderConfig

```bash
kubectl apply -f bootstrap/provider-config.yaml
```

This tells Crossplane where to find the credentials for every AWS managed resource.

### 7. Install the Platform APIs (XRDs + Compositions)

```bash
kubectl apply -f apis/networking/security-group/
kubectl apply -f apis/storage/bucket/
kubectl apply -f apis/compute/instance/
```

Verify all XRDs are Established:
```bash
kubectl get xrd
# NAME                                  ESTABLISHED   OFFERED
# xsecuritygroups.demo.crossplane.io    True          True
# xobjectstorages.demo.crossplane.io    True          True
# xvirtualmachines.demo.crossplane.io   True          True
```

---

## Deploying an Environment

### Gather AWS values (one-time per environment)

```bash
REGION=eu-central-1

# Default VPC
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text --region $REGION)

# First subnet in that VPC
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text --region $REGION)

# Latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text --region $REGION)

echo "VPC: $VPC_ID  Subnet: $SUBNET_ID  AMI: $AMI_ID"
```

Fill these into the relevant `environments/<env>/` YAML files before applying.

### Apply dev

Networking and storage can be applied in parallel.
The instance **must** come after the security group (it needs the SG ID).

```bash
# 1. Create the SG and bucket
kubectl apply -f environments/dev/networking/
kubectl apply -f environments/dev/storage/

# 2. Wait for the SG, then capture its AWS ID
kubectl wait --for=condition=Ready securitygroup/dev-sg --timeout=3m
SG_ID=$(kubectl get securitygroup dev-sg \
  -o jsonpath='{.status.securityGroupId}')
echo "SG ID: $SG_ID"

# 3. Fill in the SG ID in the instance claim, then apply
#    (or edit environments/dev/compute/instance.yaml manually)
sed -i '' "s/REPLACE_WITH_DEV_SG_ID/$SG_ID/" \
  environments/dev/compute/instance.yaml
kubectl apply -f environments/dev/compute/
```

### Observe

```bash
# All AWS-level managed resources
kubectl get managed

# User-facing claims
kubectl get securitygroup,objectstorage,virtualmachine -A

# Get specific output values
kubectl get securitygroup dev-sg \
  -o jsonpath='{.status.securityGroupId}'

kubectl get virtualmachine dev-instance \
  -o jsonpath='{.status.publicIp}'

# Full composition trace (requires Crossplane CLI)
crossplane beta trace virtualmachine dev-instance
```

### Cleanup

```bash
# Delete claims — Crossplane cascades deletion to AWS resources
kubectl delete -f environments/dev/compute/ \
               -f environments/dev/storage/ \
               -f environments/dev/networking/

# Confirm all AWS resources are gone
kubectl get managed
```

---

## Key Concepts

| Concept | What it is | Location |
|---------|------------|----------|
| Provider | Installs AWS CRDs + controller | `bootstrap/providers.yaml` |
| Function | Logic engine for Compositions | `bootstrap/functions.yaml` |
| ProviderConfig | Binds AWS credentials to providers | `bootstrap/provider-config.yaml` |
| XRD | Defines the platform's public API (the Claim schema) | `apis/*/definition.yaml` |
| Composition | Maps a Claim to real AWS managed resources, uses PatchSets for DRY patches | `apis/*/composition.yaml` |
| Claim | What the app/ops team creates — high-level, no AWS details | `environments/<env>/` |
| Managed Resource | The actual AWS object managed by Crossplane | Created automatically |

### Crossplane abstraction layers

```
┌─────────────────────────────────────────────┐
│  environments/dev/compute/instance.yaml     │  ← app team writes this
│  kind: VirtualMachine                       │
│  spec: { instanceType, ami, subnetId, ... } │
└────────────────────┬────────────────────────┘
                     │ Crossplane binds claim → composite
┌────────────────────▼────────────────────────┐
│  XVirtualMachine (composite resource)       │  ← managed by Crossplane
└────────────────────┬────────────────────────┘
                     │ Composition + function-patch-and-transform
┌────────────────────▼────────────────────────┐
│  Instance (ec2.aws.upbound.io/v1beta2)      │  ← Crossplane creates in AWS
└────────────────────┬────────────────────────┘
                     │ AWS API call
┌────────────────────▼────────────────────────┐
│  Real EC2 Instance in eu-central-1          │
└─────────────────────────────────────────────┘
```

### Versions pinned in this repo

| Component | Version |
|-----------|---------|
| Crossplane (Helm) | 2.2.1 |
| provider-aws-s3 | v2.5.3 |
| provider-aws-ec2 | v2.5.3 |
| function-patch-and-transform | v0.10.4 |
