---
title: "Containers Orchestration and Network setup"
tags:
  - AWS
  - Docker
  - Rolling deployments
  - Manga-Alert-Italia

---

Hey there! In this blog post I'm going to talk about my design choices abouth container orchestration and Network setup in the cloud.

# Orchestration: ECS vs EKS
Using Kubernetes for this project would be overkill and would introduce a high degree of complexity that is not really required for a project like this, so I'm going to take out AWS EKS from the list. Since I'm surely going to use more than 1 container and I will require some sort of orchestration (in an easier way than Kubernetes), I'm going to be using ECS. 

# ECS: Fargate vs EC2
Now this is going to be a choice based purely on how much money I'm going to spend. AWS ECS is always free, you just pay for whatever underlying resources you are using. EC2 has a free tier of 750 hours per month for the first year of AWS subscription. Fargate doesn't even have a free tier. Since my application is going to run endlessly, so 720 hours per month approximately, I'm going to build and run all the needed containers in a single EC2 instance to make use of the free tier. Fargate would bill me 4 cents per hour, 36 euros per month approximately. I know I will have to manage the underlying infrastructure under ECS, but it's still better than paying those money. 
    
There's also the Fargate + ECS Scheduled Task triggered by AWS EventBridge: the effective runtime would be significantly lower because the app wouldn't be always running, but still I would be billed for it and it would be > 0, so I will probably still go with the EC2 deployment option and I would use the event-based option only in case of necessity. 

## ECS: EC2 and EventBridge Scheduler
Finally I decided to use the following approach: an EC2 instance running hosting the backend, and two ECS Scheduled Task that get triggered every 6 hours on that same EC2 for the email notifier and the scraper services.

## ECS Task Execution vs Task vs Instance (roles) 
Ok this is kinda tricky. To define the cluster and its instances I had to understand the difference between these three roles. 
- The ECS Task Execution role is automatically used by ECS whenever an instance is launched and needs logging, Docker image pulling and other operational tasks. 
- The ECS Task role is the role that assumes the application code running inside the container, whenever the code needs to acces other AWS services, i.e. to read from a DynamoDB table.
- The ECS Instance Role is instead the role that's taken by the EC2 Instance associated to the ECS Cluster. This allows the EC2 Instance to interact mainly with the ECS control plane and other AWS services. 

# AWS VPC: 
The VPC setup is pretty standard, so I'll just write in the following lines about a problem I encountered: the ECS Task was not able to connect to the internet, while the EC2 instance that "hosted" the task was able to connect without any problem at all.   
In awsvpc network mode, each ECS task gets its own Elastic Network Interface (ENI) with a private IP address, separate from the EC2 instance's interface. When using the EC2 launch type, the assign_public_ip parameter is not supported, meaning tasks do not receive public IPs and cannot directly access the internet.  
To provide internet access I had to switch to bridge network mode, where ECS tasks use the EC2 instance's network interface (which has internet access).


## TODO: (Private subnets + NAT gateway) VS (Public subnets + internet gateway)


# ASG and ELB:
One could argue that an Auto Scaling Group and a Load Balancer are useless if provided with just one EC2 instance, but that's not entirely true. The ASG will terminate the instance if it's unhealthy (aka fails) and replace it with a new one. Instead I'm not going to use an ELB because it's because it's pricy (I know about the 750 hours by the free tier for the first 12 months but still I want to make this "long-term", in a sense that in a year from now I would like to still be able to use my application without high maintenance costs) and the workload I expect my app to be able to handle doesn't really justify it that much. 


# Storing secrets: AWS Secrets Manager
Since I'm going to need to access my database from each container, I'm going to use AWS Secrets Manager to handle the retrieval of the keys from the ECS instances. 

# TO BE CONTINUED: Rolling Deployments