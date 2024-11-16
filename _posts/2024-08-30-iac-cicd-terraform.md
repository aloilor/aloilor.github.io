---
title: "IaC and CI/CD: Terraform"
tags:
  - AWS
  - CI/CD
  - IaC
  - Terraform
---

Hey there! In this post I'm going to talk about how I set up my Terraform pipeline for automated IaC, using GitHub Actions. 

# Terraform over org-formation-cli for AWS Organizations
First of all, I realized I made a mistake setting up org-formation-cli with GitHub Actions to create my own CI/CD pipeline for managing AWS Organizations, since I found out that Terraform does it even better. I'll switch to Terraform to manage Organizations too for simplicity.

# Terraform best practices
These two videos ([1](https://www.youtube.com/watch?v=l5k1ai_GBDE&t=416s), [2](https://www.youtube.com/watch?v=gxPykhPxRW0)) and this [blog post](https://buildkite.com/blog/best-practices-for-terraform-ci-cd) helped me understand how Terraform actually works and the best practices I'm going to follow through my project. I'm writing them down here as a reminder to myself: <br>

1. Manipulate state only through TF commands
2. Setup a shared remote state and use state locking (e.g use AWS S3)
3. Backup state files (e.g. S3 versioning)
4. 1 state per environment (dev, test, prod)
5. Host TF scripts in Git repo
6. CI for TF Code (write Tests too)
7. Apply TF ONLY through CD pipeline

As a rule of thumb I would also like to store separate repos containing Terraform files - i.e. one for pure Infrastructure stuff and one for Dev AND Infra (kind of) stuff (mainly deployment on AWS of the app code). In other words, I'll be using two repos, one for the app code and one for all of the Infrastructure stuff with Terraform. 

# Terraform configuration
I'm not going to explain every step needed to setup Terraform, since there already are lots of good resources on how to do that ([e.g.](https://www.youtube.com/watch?v=7xngnjfIlK4)), but in the following section I will add my main configuration file, to show you what a super basic configuration should look like: 
```tf 
terraform {
  backend "s3" {
    bucket         = "<YOUR_BUCKET_NAME>"
    key            = "</path/to/your/S3/key>"
    region         = "<YOUR_REGION>"
    dynamodb_table = "<YOUR_DYNAMODB_TABLE>"
    encrypt        = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure the AWS Provider
provider "aws" {
  region = "<YOUR_REGION>"
}

resource "aws_s3_bucket" "s3-terraform-remote-state" {
  bucket        = "<YOUR_BUCKET_NAME>"
  force_destroy = true
}

resource "aws_s3_bucket_versioning" "s3-terraform-remote-state-versioning" {
  bucket = aws_s3_bucket.s3-terraform-remote-state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "dynamodb-terraform-lock" {
  name         = "<YOUR_DYNAMODB_TABLE>"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

# CI/CD pipeline - GitHub Actions
To conform to security best practices I decided to adopt the OIDC + IAM Role to use short term credentials.

## Prerequisites
- OIDC and dedicated IAM role - you can check out these two guides to set this up: [GitHub docs](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) and [AWS docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html#idp_oidc_Create_GitHub). 

## Pipeline 
This is what my GitHub Actions pipeline file looks like, keep in mind that every expression has been replaced with $[  ] to make sure that the environment variables don't get processed by GitHub: 

```yaml
name: "Terraform CI/CD pipeline"

env:
  ROLE_TO_ASSUME: $[ secrets.ROLE_TO_ASSUME ] 
  AWS_REGION: $[ secrets.AWS_REGION ] 

  # S3 bucket for the Terraform state
  BUCKET_TF_STATE: $[ secrets.BUCKET_TF_STATE ] 

  # verbosity setting for Terraform logs
  TF_LOG: INFO

on:
  push:
    branches:
    - main
    paths:
    - src/terraform/**

  pull_request:
    branches:
    - main
    paths:
    - src/terraform/**

permissions:
  id-token: write
  pull-requests: write
  contents: read    # This is required for actions/checkout

jobs:
  terraform:
    name: "Terraform pipeline"
    runs-on: ubuntu-latest    
    defaults:
      run:
        shell: bash
        working-directory: ./src/terraform

    steps:

      - name: Checkout the repository to the runner 
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: $[ env.ROLE_TO_ASSUME ]
          aws-region: $[ env.AWS_REGION ]
          role-session-name: Terraform-GitHub
      
      - name: Setup Terraform with specified version on the runner
        uses: hashicorp/setup-terraform@v3.1.2
        with:
          terraform_version: 1.9.5
      
      - name: Terraform init
        id: init
        run: terraform init -backend-config="bucket=$BUCKET_TF_STATE"
      
      - name: Terraform format
        id: fmt
        run: terraform fmt -check
    
      - name: Terraform validate
        id: val
        run: terraform validate
      
      - name: Terraform plan
        id: plan
        if: github.event_name == 'pull_request'
        run: terraform plan -no-color -input=false
        continue-on-error: true

      - uses: actions/github-script@v6
        if: github.event_name == 'pull_request'
        env:
          PLAN: "terraform\n$[ steps.plan.outputs.stdout ]"
        with:
          script: |
            const output = `#### Terraform Format and Style 🖌$[ steps.fmt.outcome ]\
            #### Terraform Initialization ⚙️$[ steps.init.outcome ]\
            #### Terraform Validation 🤖$[ steps.val.outcome ]\
            #### Terraform Plan 📖$[ steps.plan.outcome ]\
  
            <details><summary>Show Plan</summary>
  
            \`\`\`\n
            ${process.env.PLAN}
            \`\`\`
  
            </details>
            *Pushed by: @$[ github.actor ], Action: \`$[ github.event_name ]\`*`;
  
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
      
      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

      - name: Terraform apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve -input=false

```
Basically what happens with this workflow is the following: when submitting a pull-request, ```terraform plan```, along with a script that shows its result, get executed. When the PR gets accepted, ```terraform apply``` will be executed, applying the changes highlighted by the previous step. 
