
# 🚀 EKS Infrastructure Provisioning with Terraform + Jenkins CI/CD

## 🌍 Real-World Scenario

You're a DevOps Engineer at a fast-growing startup. Your team needs a scalable Kubernetes cluster to deploy a microservices-based application. You’ll automate the provisioning of an AWS EKS cluster using **Terraform**, set up **remote state management** with **S3 + DynamoDB**, and **automate deployments** via **Jenkins**.

---

## 📁 Project Structure

```
eks-automation/
├── terraform/
│   ├── main.tf                 # EKS Infra (VPC, Cluster, NodeGroup)
│   ├── backend.tf              # Remote backend (S3 + DynamoDB)
│   ├── outputs.tf              # Output useful values
│   ├── provider.tf             # AWS provider and region
│   ├── variables.tf            # Input variables
├── app/
│   ├── deployment.yaml         # Kubernetes Deployment
│   └── service.yaml            # Kubernetes Service
├── scripts/
│   └── create-backend.tf       # Provision S3 & DynamoDB
├── .gitignore
├── Jenkinsfile
└── README.md
```
---

## setup-files.py
```python
import os

# Base directory
base_dir = "/home/lilia/VIDEOS/eks-automation"

# File structure with file contents
file_structure = {
    "terraform/main.tf": '''# main.tf - EKS Infra (VPC, Cluster, NodeGroup)
# You can define your EKS module or resource block here
''',
    "terraform/backend.tf": '''# backend.tf - Remote backend configuration
terraform {
  backend "s3" {
    bucket         = "lili-terraform-state"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamo_table   = "terraform-state"
    encrypt        = true
  }
}
''',
    "terraform/outputs.tf": '''# outputs.tf - Output useful values
output "cluster_name" {
  value = module.eks.cluster_name
}
''',
    "terraform/provider.tf": '''# provider.tf - AWS provider and region
provider "aws" {
  region = "us-east-1"
}
''',
    "terraform/variables.tf": '''# variables.tf - Input variables
variable "region" {
  default = "us-east-1"
}
''',
    "app/deployment.yaml": '''# Kubernetes Deployment for sample app
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: laly9999/sample-app:1
        ports:
        - containerPort: 80
''',
    "app/service.yaml": '''# Kubernetes Service
apiVersion: v1
kind: Service
metadata:
  name: sample-service
spec:
  type: LoadBalancer
  selector:
    app: sample-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
''',
    "scripts/create-backend.tf": '''# create-backend.tf - S3 + DynamoDB for remote state
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "lili-terraform-state"
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_state" {
  name         = "terraform-state"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}

output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
}

output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_state.name
}
''',
    ".gitignore": '''# .gitignore
.terraform/
terraform.tfstate
terraform.tfstate.backup
*.tfvars
*.log
''',
    "Jenkinsfile": '''pipeline {
  agent any

  environment {
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
    REGION                = "us-east-1"
  }

  stages {
    stage('Terraform Init') {
      steps {
        sh 'cd terraform && terraform init'
      }
    }

    stage('Terraform Plan') {
      steps {
        sh 'cd terraform && terraform plan'
      }
    }

    stage('Terraform Apply') {
      steps {
        input "Approve apply?"
        sh 'cd terraform && terraform apply -auto-approve'
      }
    }

    stage('K8s Deploy') {
      steps {
        sh '''
        aws eks update-kubeconfig --region us-east-1 --name eks_cluster
        kubectl apply -f app/deployment.yaml
        kubectl apply -f app/service.yaml
        '''
      }
    }
  }
}
'''
}

# Create the directories and files
for relative_path, content in file_structure.items():
    file_path = os.path.join(base_dir, relative_path)
    os.makedirs(os.path.dirname(file_path), exist_ok=True)
    with open(file_path, "w") as f:
        f.write(content)

"All files created successfully in /home/lilia/VIDEOS/eks-automation"


```

---

## 🧠 Step-by-Step Instructions

### 1️⃣ Provision EKS Infrastructure (Locally First)

```bash
cd terraform/
terraform fmt
terraform init
terraform validate
terraform plan
terraform apply -auto-approve
```

---

### 2️⃣ Configure `kubectl` for EKS

```bash
aws eks update-kubeconfig --region us-east-1 --name eks_cluster
kubectl get nodes
```

---

### 3️⃣ Deploy Sample Application

```bash
kubectl apply -f ../app/deployment.yaml
kubectl apply -f ../app/service.yaml
kubectl get pods
kubectl get svc
```

- Visit the `EXTERNAL-IP` of the service in your browser.

---

## ☁️ Remote Backend Setup (S3 + DynamoDB)

### A. Backend Script Example (`scripts/create-backend.tf`)

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "lili-terraform-state"
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_state" {
  name         = "terraform-state"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}

output "s3_bucket_arn" {
  value       = aws_s3_bucket.terraform_state.arn
}

output "dynamodb_table_name" {
  value       = aws_dynamodb_table.terraform_state.name
}
```

---

### B. Remote Backend Block in `provider.tf`

```hcl
terraform {
  backend "s3" {
    bucket         = "lili-terraform-state"
    key            = "global/s3/terraform.tfstate"
    region         = "us-east-1"
    dynamo_table   = "terraform-state"
    encrypt        = true
  }
}
```

### C. Migrate to Remote Backend

```bash
terraform init -migrate-state
```

### D. Confirm Remote State

```bash
terraform state list
cat terraform.tfstate  # Should be minimal or empty
```

- Check the state file in S3 console.

---

### E. State Locking Test

```bash
terraform plan      # Then press Ctrl+C
terraform plan      # You’ll see locking error

# Unlock manually:
terraform force-unlock <LOCK_ID>
```

---

## 🔄 Optional: Switch Back to Local State

```hcl
# Comment the backend block in provider.tf
terraform init -migrate-state
```

---

## 🤖 Automate with Jenkins

### Jenkinsfile Example

```groovy
pipeline {
  agent any

  environment {
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
    REGION                = "us-east-1"
  }

  stages {
    stage('Terraform Init') {
      steps {
        sh 'cd terraform && terraform init'
      }
    }

    stage('Terraform Plan') {
      steps {
        sh 'cd terraform && terraform plan'
      }
    }

    stage('Terraform Apply') {
      steps {
        input "Approve apply?"
        sh 'cd terraform && terraform apply -auto-approve'
      }
    }

    stage('K8s Deploy') {
      steps {
        sh '''
        aws eks update-kubeconfig --region us-east-1 --name eks_cluster
        kubectl apply -f app/deployment.yaml
        kubectl apply -f app/service.yaml
        '''
      }
    }
  }
}
```

---

## ✅ Recap: Why Remote State?

| Feature         | Local State                     | Remote State (S3 + DynamoDB)           |
|----------------|----------------------------------|----------------------------------------|
| Accessibility  | Only on developer’s machine      | Shared and centralized                 |
| Locking        | ❌ Not supported                 | ✅ Prevents race conditions            |
| Backups        | ❌ Risk of loss                  | ✅ Versioned in S3                     |
| Scalability    | Poor for teams                  | Excellent for collaboration            |

---

## 🔚 Final Step

```bash
kubectl get svc
```

- Copy the `EXTERNAL-IP` and open it in a browser to access your deployed app.

---

