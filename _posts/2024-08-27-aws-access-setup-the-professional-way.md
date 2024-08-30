---
title: "AWS access management setup - the professional way"
tags:
  - org-formation-cli
  - GitHub Actions
  - GHAs
  - CI/CD
  - AWS IAM
  - AWS IAM Identity Center
  - AWS SSO
  - AWS Organizations
---
**OUTDATED**: I made a mistake setting up OrgFormation for my CI/CD pipeline, Terraform does it all. <br> <br>
I'm gonna be using AWS Organizations, IAM Identity Center and org-formation-cli to manage users and access credentials, since the "one root user does it all" formula introduces security vulnerabilities and it isn't really used in production environments. 

# OrgFormation vs AWS Control Tower
OrgFormation is opensource, cheaper (this is really important since I don't want to drain my bank account for this project), more flexible than the AWS-native Control Tower and it's code defined and therefore can be managed as IaC, making my CI/CD pipeline more complete. Check out their documentation [here](https://github.com/org-formation/org-formation-cli/blob/master/docs/articles/aws-organizations.md). 

# OrgFormation setup and functioning
The basic setup and functioning is pretty easy and it's well documented [here](https://github.com/org-formation/org-formation-cli/blob/master/docs/articles/org-formation.md). The only tricky part is about the command that initializes the CI/CD pipeline:
```
> org-formation init pipeline organization.yml --region <REGION> --profile <PROFILE>
```
since it requires both AWS CodeCommit and CodePipeline, even though CodeCommit has been [discontinued](https://aws.amazon.com/blogs/devops/how-to-migrate-your-aws-codecommit-repository-to-another-git-provider/#:~:text=After%20careful%20consideration%2C%20we%20have,use%20the%20service%20as%20normal.). Because of this I'm going to setup a custom worfklow with GitHub Actions to effectively implement this into a CI/CD pipeline. 

## OrgFormation GitHub Actions CI/CD pipeline
I found really little information about working around the CodeCommit discontinuation, so in the following section I'm gonna write a short and effective guide on how to setup the pipeline using GitHub Actions. For more information you can check out this [issue](https://github.com/org-formation/org-formation-cli/issues/568) from the original repo and this [blog post](https://mantelgroup.com.au/managing-multiple-aws-accounts-with-org-formation/). You can also check out my [repo](https://github.com/aloilor/org-formation-github-actions) to get a better grasp of it. 

### Prerequisites:
- [org-formation-cli](https://github.com/org-formation/org-formation-cli) 
- OIDC and dedicated IAM role - you can check out these two guides to set this up: [GitHub docs](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) and [AWS docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html#idp_oidc_Create_GitHub). 

#### IAM GitHub Actions role
Setting up permissions for this role in a way that the OIDC can only access needed resources can be a bit tricky, I spent several hours searching for resources and this is the best I could find and should explain everything you need to know - [Issue 120](https://github.com/org-formation/org-formation-cli/issues/120#issuecomment-751415550). <br> <br>
Here is a basic template you can use to setup your workflow, just edit it with your information and you're good to go:
```yaml
name: "org-formation-cicd-pipeline-ghas"

env:
  ROLE_TO_ASSUME: <ARN of the GitHub Actions role to assume>
  AWS_REGION: <Your region>

on:
  push:
    branches:
      - main     
    paths:
      - <Path whose changes will trigger the GHAs>


permissions:
  id-token: write   # This is required for requesting the JWT
  contents: read    # This is required for actions/checkout

jobs:
  org-formation-push:
    name: "org-formation-cicd-pipeline-ghas"
    runs-on: ubuntu-latest
    if: github.event_name == 'push'  
    
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Organization Formation
        run: | 
          npm install -g aws-organization-formation
          org-formation -v

      - name: Update Organization
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: org-formation update <Path to organization.yml>
```

# SSO setup through IAM Identity Center 
I found these two blog posts to be insanely helpful in setting up SSO: [1](https://slaw.securosis.com/p/bring-federation-sso) and [2](https://slaw.securosis.com/p/another-sso-iam-identity-center-part-2). Anyway I'll try to sum them up here, for more detailed information you can just check those. 
1. Enable IAM Identity Center, along with AWS Organizations
2. Create a Group inside the IAM IC console (in my case I created the group Developers)
3. Setup MFA on every sign-in from the Settings -> Authentication tab 
4. Create a new permission set (PowerUserAccess for Developers in my case)
5. AWS Accounts -> Assign the role and the permission set to to the organizations account we want to SSO into

## IAM, Organizations and IAM Identity Center
This is something that really made my head hurt, so I'm going to spend a few words highlighting the differences between these AWS services, specifically on how users are managed between them (and why they are also different between them). Disclaimer: the IAM guide itself suggests to not use IAM Users anymore ([the big red Important rectangle here in the docs](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html)), and instead use IAM Identity Center to require users to use federations with an identity provider to access using temporary credentials (SSO). Ok now let's get going: 
- IAM focuses on managing access within the boundaries of a single account. You could call it a "local" service, IAM Users are confined to the account where they are created, thus IAM operates independently for each AWS account. IAM is not designed to provide centralized user management. 
<br> <br>
- Organizations is a service for centralized management of ACCOUNTS (keep in mind this term, since ACCOUNTS and USERS are not the same thing). It focuses on account-level management and does not handle individual identities or users (SPOILER: IAM Identity Center does). <br> <br>
- IAM Identity Center (previously known as AWS SSO, so the name should already give you a hint on what it's used for) is instead a service for centralized management of USERS (Users created inside IAM IC can access multiple AWS Accounts through a centralized login system - SSO). Multi-account access is possible only through IAM IC and it's not possible through IAM. Identity Center users are managed within IAM IC and are linked to multiple accounts through groups and permissions. What this means in a practical way is that, once an Organization account (or more than one) is associated to a Group + Permission set, using a single IAM IC User with that Group + Permission set, you'll be able to log in into every Organization account with that predefined Group + Permission set.





