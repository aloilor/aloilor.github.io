---
title: "A CI/CD fresh start"
tags:
    - AWS
---

Hey there! In the following weeks (most likely months) I'll be using this archive to document a side project of mine. I will implement a CI/CD production environment, along with all the DevOps magic, Infrastructure as Code (IaC), container orchestration and monitoring/logging/alerting systems (I bet I forgot something). The goal is to have a fully functional, automated production environment ready to use anytime. 

# Stack choices
- Cloud: AWS
- CI/CD: GitHub Actions
- IaC: Terraform

## CI/CD: GitHub Actions vs GitLab CI/CD vs Circle CI vs AWS Codebuild vs Jenkins
I will start by crossing out Jenkins from this list since it's difficult to mantain and configure if you're a solo dev (as I am) or a smaller team and it also seems to be really dated. <br>
AWS Codebuild has to go too, since I want to setup a CI pipeline that's cloud-agnostic: to be tied to a specific cloud provider would be limiting. <br>
Even though CircleCI seems to be a much more mature product and a better choice because its free tier allows for 6,000 build minutes and 2GB of storage, I'm going to use GHA for simplicity since my repo will be uploaded to GitHub and I probably won't be needing complex enough workflows that justify choosing CircleCI. 

## Cloud provider: AWS vs GCP vs Azure
AWS doesn't provide a free tier for its K8S service (EKS) while GCP and Azure do. The money I will be spending on EKS will be negligible though because I will not be hosting large enough applications for it to drain my bank account. <br>
I will stick to AWS since it's the most used, and also because it's the one I'm more accustomed to since I use it at work. 
UPDATE: I ended up not even using AWS EKS but instead I used ECS for simplicity. Anyway AWS is the most used cloud provider, so no harm has been done. 

## IaC: Terraform
There's really no need to justify this choice. Terraform is the most known, most documented and most used IaC service. Even if it's not open-source anymore, it's still free and that does it for me. 
