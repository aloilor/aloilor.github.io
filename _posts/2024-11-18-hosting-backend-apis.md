---
title: "Publicly hosting the backend APIs"
tags:
  - AWS
  - Manga Alert Italia
---

Hey there! In today's blog post I'm going to talk about a topic I had to wrap my head around for quite a while. My goal was to host the backend APIs, enforcing communication over HTTPS, without resorting to pricy solutions (Load Balancers I'm talking to you).

# Automatic Load Balancer
I know, I know, it's included in the AWS Free Tier, so for the first 12 months it's entirely free. Problem is I want to create a "long-term" solution with my application, so I want to keep it hosted even when the Free Tier will expire, so there's no chance in hell I'm ever gonna pay $16-20 a month to keep it, that's madness.

# API Gateway + NLB
Again, this is too pricy: the API Gateway is not, but the Network Load Balancer costs a little less than an ALB, so it's *not feasible*. 

# Cloudfront CDN: automatic SSL certificates management
This could have been a really good solution to be honest. I can enforce HTTPS through the CDN (also the SSL certificates would be free and entirely AWS-managed, renewal included, so that would be really great), problem is that the communication between the CDN and the EC2 on which the backend is hosted would be over HTTP, and the EC2 should still be accessible by the whole Internet. You could argue that I could restrict, through the security group rules, the number of IPs that can access the machine, but it's simply not doable: there's a list of IPs associated to AWS' CDNs, but they change regularly and they are also A LOT, not to mention that this is just a "workaround" to the initial problem, so no. 

# Final Solution: SSL certificates by Let's Encrypt, automatic renewal and Nginx
I'm actually going to talk about automatic SSL Certificates renewal through AWS Lambda in the following blog post (), since it's going to be quite long

### Nginx reverse proxy
There's nothing much to say about this choice: I used Nginx as a reverse proxy, to redirect HTTPS traffic from ```https://api.mangaalertitalia.it``` to the local Flask server. 

