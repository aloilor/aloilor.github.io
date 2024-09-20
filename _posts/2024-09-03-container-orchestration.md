---
title: "Containers Orchestration and Network setup"
tags:
  - AWS
  - Docker
  - Rolling deployments
---

Hey there! In this blog post I'm going to talk about my design choices abouth container orchestration and Network setup in the cloud.


# Orchestration: ECS vs EKS
Using Kubernetes for this project would be overkill and would introduce a high degree of complexity that is not really required for a project like this, so I'm going to take out AWS EKS from the list. Since I'm surely going to use more than 1 container and I will require some sort of orchestration (in an easier way than Kubernetes), I'm going to be using ECS. 

# ECS: Fargate vs EC2
Now this is going to be a choice based purely on how much money I'm going to spend. AWS ECS is always free, you just pay for whatever underlying resources you are using. EC2 has a free tier of 750 hours per month for the first year of AWS subscription. Fargate doesn't even have a free tier. Since my application is going to run endlessly, so 720 hours per month approximately, I'm going to build and run all the needed containers in a single EC2 instance to make use of the free tier. Fargate would bill me 4 cents per hour, 36 euros per month approximately. I know I will have to manage the underlying infrastructure under ECS, but it's still better than paying those money. 
    
There's also the Fargate + ECS Scheduled Task triggered by AWS EventBridge: the effective runtime would be significantly lower because the app wouldn't be always running, but still I would be billed for it and it would be > 0, so I will probably still go with the EC2 deployment option and I would the event-based option only in case of necessity. 

# AWS VPC: 


# Rolling deployment


# Passing secrets: AWS Secrets Manager, AWS RDS and ECS Tasks IAM Roles
Since I'm going to need to access my database from each container, I'm going to use AWS Secrets Manager to handle the retrieval of the keys from the Fargate instances through IAM Roles. 
