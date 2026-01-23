---
title: "terraform for aws(1)"
date: 2026-01-23T10:05:01+09:00
draft: false          
type: "post"         
categories:
  - Tech Notes
tags:
  - Terraform
  - AWS
author: "coherence"  
aliases:
  - /archives/1
---

## Environment
- AWS Provider  
- Terraform v1.14.3
- Terraformer v0.8.30  

## Introduction
For a long time, my workflow was based on the AWS Management Console. While it's intuitive, it is hard to replicate and prone to human error. Recently, I decided to automate my entire environment—including Server—using Terraform. This post covers what I have learned and how I transitioned to a 'one-click' automated deployment.

## Technical Implementation (The Code)
```bash
# Login in aws account and access in with your access key and password made from aws IAM account
aws configure

# Terraform launch
terraform init

# Give you some recommend configurations
terraform plan

# Deploy in your local environment
terraform apply

# Cancel your deployment in your local environment
terraform destory

# Login in your terraform cloud and you need to input a token that you can get from your terraform cloud on browser, and also you can make several workplace to work individually
terraform login

# Check that if you connected to the aws cli
cat ~/.aws/credentials

# Check your iam identity
aws sts get-caller-identity
```

## About backend
### Local Backend
- Storage location: Your local computer folder, named terraform.tfstate.
- Advantages:
Simple setup with no extra configuration needed, ideal for beginners experimenting on their own machines.
- Disadvantages:
• No collaboration: Your colleagues can't access your local file, so when they run the code, it will assume the cloud is empty. 
• Insecure: If your computer crashes or you accidentally delete the folder, you lose control over cloud resources (requiring manual deletion via the console).
• Sensitive data: This file stores plaintext configuration details, posing a leakage risk when stored locally.

### Remote Backend
- Storage location: your Terraform cloud plateform(free up to 5 users) or S3
- Advantages:
• Team Collaboration: Anyone with permissions can read the same state file, enabling collaborative work.
• High Security: With S3 versioning enabled, accidentally deleted states can be recovered.
• Locking Mechanism: When paired with AWS DynamoDB, it prevents two users from simultaneously running `apply` commands and causing environment chaos (similar to conflicts when multiple people edit the same Word document).

```hcl
# To contruct a remote backend environment(terraform cloud)
terraform {
  backend "remote" {
    organization = "develop-directive"

    workspaces {
        name = "terraform-course"
    }
  }
}

# To contruct a remote backend environment(S3)
# Using S3 to store DynamoDB state and share it with colleagues
# Contrcut a bucket
resource "aws_s3_bucket" "terraform_state" {
  bucket = "develop-directive-bucket"
}

# Dedicated version control (standalone resource)
resource "aws_s3_bucket_versioning" "enabled" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Specifically configured encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "default" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
    name       = "terraform-state-locking"
    billing_mode    = "PAY_PER_REQUEST"
    hash_key   = "LockID"
    attribute {
        name = "LockID"
        type = "S"
    }
}
```

## Convert AWS resources into configuration files
```bash
# Use the terraformer not officially but have efficiency
terraformer import aws -r vpc,subnet,sg,ec2_instance --regions=ap-northeast-1

# Or you want to convert by id
```hcl
resource "aws_instance" "my_server" {
  # Leave this blank for now. Terraform will prompt you to complete it after import.
}
```
```bash
terraform import aws_instance.my_server i-1234567890abcdef0
terraform plan

# Or you want to convert by tags
terraformer import aws -r ec2_instance --filter="Name=tags.Team;Value=DevOps" --filter="Name=tags.Project;Value=MyProject"

# If you have so many instance ids
$instances = @(
  "i-0123456789abcdef0",
  "i-0123456789abcdef1",
  "i-0123456789abcdef2"
)

foreach ($id in $instances) {
  terraform import "aws_instance.instance_$id" $id
}
```

## Error
To enable one-click import of terraformer configurations, they must match Terraform configurations and be AMD64-compatible; otherwise, errors will occur. Additionally, this folder is required to store the terraform-provider-aws_v4.51.0_x5.exe file.
```bash
open \.terraform.d/plugins/windows_amd64: The system cannot find the path specified.

C:\Users\ZE38833\.terraform.d\plugins\windows_amd64
```

## Additional words
Switching to Terraform changed my perspective on cloud management. It’s no longer about 'where to click,' but about 'how to architect.'

