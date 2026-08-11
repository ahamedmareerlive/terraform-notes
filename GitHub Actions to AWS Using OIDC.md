# GitHub Actions to AWS Using OIDC

## A Beginner-Friendly Guide for Terraform

## Table of Contents

1. [What are we trying to do?](#1-what-are-we-trying-to-do)
2. [What is OIDC?](#2-what-is-oidc)
3. [Why do we use OIDC?](#3-why-do-we-use-oidc)
4. [The complete authentication flow](#4-the-complete-authentication-flow)
5. [Important components](#part-1-important-components)
6. [Do we need GitHub Secrets?](#part-2-do-we-need-github-secrets)
7. [Manual AWS setup](#part-3-manual-aws-setup)
8. [GitHub Actions setup](#part-4-github-actions-setup)
9. [Adding Terraform](#part-5-adding-terraform)
10. [What happens during a workflow run?](#part-6-what-happens-during-a-workflow-run)
11. [Creating the OIDC setup with Terraform](#part-7-creating-the-oidc-setup-with-terraform)
12. [Production security recommendations](#part-8-production-security-recommendations)
13. [Common errors](#part-9-common-errors)
14. [Beginner summary](#part-10-beginner-summary)

---

## 1. What Are We Trying to Do?

Imagine you have Terraform code stored in a GitHub repository.

When you push the code to GitHub, you want GitHub Actions to run Terraform and create resources in your AWS account.

For example, Terraform may need to create:

- An S3 bucket
- An EC2 instance
- A VPC
- An RDS database
- IAM roles and policies

The problem is that AWS does not automatically trust GitHub.

GitHub needs to prove its identity before AWS allows it to create or change resources.

OIDC provides a secure way for GitHub to prove its identity to AWS **without storing permanent AWS access keys in GitHub**.

---

## 2. What Is OIDC?

OIDC stands for:

```text
OpenID Connect
```

OIDC is a method that allows one system to prove its identity to another system.

In our example:

- GitHub is the system proving its identity
- AWS is the system checking the identity
- GitHub sends a temporary identity token to AWS
- AWS verifies the token
- AWS gives temporary credentials to GitHub Actions

A simple way to understand OIDC is to think of it as a temporary digital identity card.

```text
GitHub says:

"I am a workflow running from this repository
and this branch."
```

AWS checks this information and decides whether GitHub should be allowed to access the AWS account.

---

## 3. Why Do We Use OIDC?

Before OIDC, people normally created an IAM user and saved these values in GitHub Secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

These are long-lived credentials.

If someone steals these credentials, they may be able to access the AWS account until the credentials are disabled or deleted.

With OIDC:

- You do not store permanent AWS access keys in GitHub
- GitHub receives short-lived credentials
- The temporary credentials expire automatically
- AWS can restrict access to a particular repository
- AWS can restrict access to a particular branch
- AWS can restrict access to a GitHub Environment
- Different repositories can receive different permissions

OIDC therefore reduces the need to create, store, and rotate permanent AWS access keys.

---

## 4. The Complete Authentication Flow

The complete process looks like this:

```text
Developer pushes Terraform code
              |
              v
GitHub Actions workflow starts
              |
              v
GitHub creates a temporary OIDC JWT
              |
              v
GitHub sends the JWT to AWS STS
              |
              v
AWS verifies the JWT
              |
              v
AWS checks the IAM role trust policy
              |
              v
AWS provides temporary credentials
              |
              v
Terraform uses the temporary credentials
              |
              v
Terraform creates or changes AWS resources
```

The important point is:

> Terraform does not directly create the OIDC token.

GitHub Actions obtains the OIDC token. AWS verifies it and returns temporary credentials. Terraform then uses those temporary credentials.

---

# Part 1: Important Components

## 5. What Is the GitHub OIDC Provider?

GitHub operates an OIDC service at:

```text
https://token.actions.githubusercontent.com
```

This service can issue identity tokens for GitHub Actions workflows.

When we add this URL as an OIDC identity provider in AWS, we are telling AWS:

> Trust properly signed identity tokens that come from GitHub's OIDC service.

Adding the provider does not give GitHub permission to create AWS resources.

It only tells AWS that GitHub is a recognized identity provider.

AWS permissions are provided separately through an IAM role.

---

## 6. What Is a JWT?

JWT stands for:

```text
JSON Web Token
```

A JWT is a signed token containing information about the workflow requesting access.

A simplified JWT payload could contain information like this:

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "aud": "sts.amazonaws.com",
  "sub": "repo:my-company/my-terraform-repository:ref:refs/heads/main"
}
```

### Meaning of the fields

#### `iss`

`iss` means issuer.

```json
"iss": "https://token.actions.githubusercontent.com"
```

It identifies the system that created the token. In this case, GitHub created the token.

#### `aud`

`aud` means audience.

```json
"aud": "sts.amazonaws.com"
```

It means the token is intended to be used with AWS STS.

#### `sub`

`sub` means subject.

```json
"sub": "repo:my-company/my-terraform-repository:ref:refs/heads/main"
```

It identifies the GitHub organization, repository, branch, or environment requesting access.

AWS can check this value in the role's trust policy.

> **Important:** GitHub OIDC claims can vary depending on how the repository, environment, and current GitHub claim format are configured. The actual subject claim must match the condition used in the AWS trust policy.

---

## 7. How Are Private and Public Keys Used?

GitHub digitally signs the JWT.

### GitHub's responsibility

GitHub holds a private signing key.

```text
GitHub private key
        |
        v
Signs the JWT
```

The private key remains inside GitHub's infrastructure.

You do not create it. You do not save it in GitHub Secrets. You do not provide it to AWS.

### AWS's responsibility

AWS uses GitHub's published public key to verify the token's signature.

```text
Signed JWT from GitHub
          |
          v
AWS obtains GitHub's public key
          |
          v
AWS verifies the signature
```

If the signature is valid, AWS knows the token was signed by GitHub.

If someone creates a fake JWT without GitHub's private key, the signature verification will fail.

Therefore:

```text
Private key: Managed by GitHub
Public key: Published by GitHub and used by AWS
```

You do not manually exchange these cryptographic keys.

---

## 8. What Is AWS STS?

STS stands for:

```text
AWS Security Token Service
```

AWS STS creates temporary security credentials.

GitHub sends its OIDC token to AWS STS and requests access to an IAM role.

The OIDC operation used for this process is:

```text
AssumeRoleWithWebIdentity
```

If AWS accepts the request, STS returns temporary credentials containing values similar to:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
```

The credentials expire after a limited time.

GitHub Actions puts these temporary values into the workflow environment. Terraform's AWS provider then uses them automatically.

---

## 9. What Is an AWS IAM Role?

An IAM role is a temporary identity inside AWS.

For this setup, you might create a role named:

```text
GitHubTerraformRole
```

The role has two important parts:

1. Trust policy
2. Permissions policy

---

## 10. What Is a Trust Policy?

The trust policy answers this question:

> Who is allowed to use this IAM role?

For example:

```text
Only GitHub Actions
from my-company/my-terraform-repository
using the main branch
can assume this role.
```

A simplified trust policy looks like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:MY-ORGANIZATION/MY-REPOSITORY:*"
        }
      }
    }
  ]
}
```

### What does `Principal` mean?

```json
"Principal": {
  "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
}
```

This identifies GitHub's OIDC provider as the trusted identity provider.

### What does `Action` mean?

```json
"Action": "sts:AssumeRoleWithWebIdentity"
```

This allows GitHub to request the role using an OIDC identity token.

### What does `aud` check?

```json
"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
```

This checks that the token was intended for AWS STS.

### What does `sub` check?

```json
"token.actions.githubusercontent.com:sub":
"repo:MY-ORGANIZATION/MY-REPOSITORY:*"
```

This limits access to the specified GitHub organization and repository.

---

## 11. What Is a Permissions Policy?

The permissions policy answers this question:

> What is this role allowed to do after it has been assumed?

For example, the role might be allowed to:

- Read and write Terraform state in S3
- Create EC2 instances
- Create VPC resources
- Read parameters from AWS Systems Manager
- Read secrets from Secrets Manager

The trust policy and permissions policy have different jobs:

```text
Trust policy:
Who can use this role?

Permissions policy:
What can the role do?
```

For production, the role should receive only the permissions Terraform requires. This is called least privilege.

Do not use `AdministratorAccess` for a real production pipeline.

---

## 12. What Does the Role ARN Mean?

An IAM role ARN may look like this:

```text
arn:aws:iam::123456789012:role/GitHubTerraformRole
```

Breaking it into parts:

```text
arn
```

This means Amazon Resource Name.

```text
aws
```

This identifies AWS.

```text
iam
```

This identifies the IAM service.

```text
123456789012
```

This is the AWS account ID.

```text
role/GitHubTerraformRole
```

This identifies the IAM role and its name.

Therefore, the full ARN means:

> The IAM role named `GitHubTerraformRole` inside AWS account `123456789012`.

This is the role that GitHub Actions requests permission to assume.

The role belongs to AWS, not GitHub. The ARN is an identifier, not a password or secret.

---

# Part 2: Do We Need GitHub Secrets?

## 13. Do We Need to Store AWS Keys in GitHub Secrets?

No permanent AWS keys are required when using OIDC correctly.

You should not need these GitHub Secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

The workflow only needs information such as:

- AWS region
- IAM role ARN
- Terraform working directory
- Terraform backend configuration

For example:

```yaml
role-to-assume: arn:aws:iam::123456789012:role/GitHubTerraformRole
aws-region: us-east-1
```

The AWS account ID and role ARN are identifiers. They are not authentication secrets.

You may store them as GitHub repository variables for easier configuration, but they do not need to be treated like passwords.

---

## 14. Are GitHub Secrets Completely Unnecessary?

OIDC removes the need to store permanent **AWS credentials**.

However, your application or Terraform process may use other secrets.

Examples include:

- Database passwords
- API keys
- Terraform Cloud tokens
- Private module registry tokens
- Third-party service credentials

These must still be managed securely.

Where possible, production secrets should be stored in a secure secret-management service such as:

- AWS Secrets Manager
- AWS Systems Manager Parameter Store
- GitHub Environment Secrets
- Terraform Cloud sensitive variables

Do not place passwords directly inside Terraform code.

---

# Part 3: Manual AWS Setup

## 15. Prerequisites

Before starting, you need:

1. An AWS account
2. Permission to create IAM identity providers
3. Permission to create IAM roles and policies
4. A GitHub repository
5. Permission to create a GitHub Actions workflow
6. Your GitHub organization or username
7. Your repository name

Example values:

```text
GitHub organization or username: my-company
Repository: terraform-aws-infrastructure
AWS account ID: 123456789012
AWS region: us-east-1
```

---

## 16. Step 1: Create the GitHub OIDC Provider in AWS

Open the AWS Management Console.

Go to:

```text
IAM
  > Identity providers
  > Add provider
```

Choose:

```text
Provider type: OpenID Connect
```

Enter the provider URL:

```text
https://token.actions.githubusercontent.com
```

Enter the audience:

```text
sts.amazonaws.com
```

Create the provider.

### Why are we doing this?

AWS does not trust every token received from the internet.

This step tells AWS:

> GitHub is a known identity provider. AWS may verify tokens issued by GitHub.

This step does not yet provide permission to create AWS resources.

---

## 17. Step 2: Create an IAM Role

Go to:

```text
IAM
  > Roles
  > Create role
```

Select the web identity or federated identity option.

Choose the GitHub OIDC provider:

```text
token.actions.githubusercontent.com
```

Choose the audience:

```text
sts.amazonaws.com
```

Give the role a clear name:

```text
GitHubTerraformRole
```

### Why are we creating this role?

The role is the AWS identity that GitHub Actions will temporarily use.

GitHub does not become an AWS IAM user. Instead, it temporarily assumes this IAM role.

---

## 18. Step 3: Configure the Role's Trust Policy

Configure the role to trust only your intended GitHub repository.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:my-company/terraform-aws-infrastructure:*"
        }
      }
    }
  ]
}
```

Replace the account ID, GitHub organization or username, and repository name with your real values.

### Why is this important?

Without a repository restriction, more GitHub repositories may be able to request access to the role.

The trust policy should be as specific as possible.

For production, you may restrict access to:

- A specific repository
- A specific branch
- A GitHub Environment such as `production`

---

## 19. Step 4: Add Permissions to the Role

Now decide what Terraform should be allowed to do.

For a small learning example, Terraform might need permission to create an S3 bucket.

For production, create a custom IAM permissions policy with only the required actions.

### Why are we adding permissions?

The trust policy only allows GitHub to assume the role.

It does not automatically allow the role to create resources.

The permissions policy specifies exactly what Terraform can do after GitHub assumes the role.

---

## 20. Step 5: Copy the Role ARN

After creating the role, copy its ARN.

```text
arn:aws:iam::123456789012:role/GitHubTerraformRole
```

You will use this ARN inside the GitHub Actions workflow.

### Why is the ARN required?

GitHub needs to tell AWS which IAM role it wants to assume.

The role ARN uniquely identifies that role.

---

# Part 4: GitHub Actions Setup

## 21. Create the Workflow File

Inside your GitHub repository, create:

```text
.github/workflows/terraform.yml
```

Add a workflow similar to this:

```yaml
name: Terraform AWS OIDC Test

on:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  test-aws-connection:
    runs-on: ubuntu-latest

    steps:
      - name: Download repository code
        uses: actions/checkout@v4

      - name: Authenticate to AWS using OIDC
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubTerraformRole
          aws-region: us-east-1

      - name: Check AWS identity
        run: aws sts get-caller-identity
```

Replace the example role ARN with your real role ARN.

---

## 22. Why Is `id-token: write` Required?

```yaml
permissions:
  id-token: write
```

It allows the GitHub Actions job to request an OIDC token from GitHub.

It does not mean that the workflow can write to your AWS account.

It only gives the workflow permission to request its GitHub identity token.

The AWS role determines what the workflow can do in AWS.

---

## 23. Why Is `contents: read` Required?

```yaml
contents: read
```

This allows the workflow to read the repository.

The checkout action needs this permission to download your Terraform code onto the GitHub Actions runner.

---

## 24. What Does `configure-aws-credentials` Do?

```yaml
uses: aws-actions/configure-aws-credentials@v6
```

This action performs the OIDC authentication process.

It approximately does the following:

1. Requests an OIDC JWT from GitHub
2. Sends the token to AWS STS
3. Requests the specified IAM role
4. Receives temporary AWS credentials
5. Adds those credentials to the job environment
6. Makes the credentials available to later steps

Terraform can then use the AWS credentials without you manually passing them.

---

## 25. How Do We Test the Connection?

```bash
aws sts get-caller-identity
```

A successful response will look similar to:

```json
{
  "UserId": "AROAXXXXXXXXX:GitHubActions",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/GitHubTerraformRole/GitHubActions"
}
```

The returned ARN contains:

```text
assumed-role/GitHubTerraformRole
```

This confirms that GitHub Actions successfully assumed the AWS IAM role.

---

# Part 5: Adding Terraform

## 26. Terraform AWS Provider

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

You do not need to put an access key or secret key in the provider configuration.

Do not do this:

```hcl
provider "aws" {
  access_key = "permanent-key"
  secret_key = "permanent-secret"
}
```

The AWS provider automatically uses the temporary credentials added by the GitHub Actions authentication step.

---

## 27. Example Terraform Resource

For a simple test, Terraform could create an S3 bucket:

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "replace-this-with-a-globally-unique-name"
}
```

Remember that S3 bucket names must be globally unique.

---

## 28. Complete Terraform Workflow

```yaml
name: Terraform Deployment

on:
  workflow_dispatch:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
      - name: Download repository code
        uses: actions/checkout@v4

      - name: Authenticate to AWS using OIDC
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubTerraformRole
          aws-region: us-east-1

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Install Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform init
        run: terraform init

      - name: Terraform format check
        run: terraform fmt -check

      - name: Terraform validate
        run: terraform validate

      - name: Terraform plan
        run: terraform plan
```

For production, `terraform apply` should normally have additional protection, such as:

- Pull-request review
- GitHub Environment approval
- Protected branches
- Separate plan and apply jobs
- Restricted production role
- Saved Terraform plan file

---

# Part 6: What Happens During a Workflow Run?

## 29. Complete Step-by-Step Process

### Step 1: The workflow starts

A developer pushes code or manually starts the workflow.

### Step 2: GitHub prepares the runner

GitHub creates a temporary runner to execute the workflow.

### Step 3: The workflow requests an OIDC token

Because the workflow has `id-token: write`, it can request a JWT from GitHub.

### Step 4: GitHub creates and signs the JWT

The JWT contains information about the repository and workflow. GitHub signs it using GitHub's private key.

### Step 5: The AWS credentials action receives the JWT

The `configure-aws-credentials` action sends the token to AWS STS.

### Step 6: AWS verifies the token

AWS checks:

- Was the token signed by GitHub?
- Is the token expired?
- Is the audience correct?
- Does the repository match the trust policy?
- Does the branch or environment match the trust policy?
- Is the requested role the correct role?

### Step 7: AWS STS creates temporary credentials

If all checks pass, AWS creates short-lived credentials for the IAM role.

### Step 8: Terraform uses the credentials

Terraform uses the temporary credentials to call AWS APIs.

### Step 9: AWS checks permissions

For every operation, AWS checks the role's permissions policy.

If Terraform tries to do something that the role does not allow, AWS returns an `AccessDenied` error.

### Step 10: The credentials expire

After their validity period ends, the credentials can no longer be used.

The next workflow run must request a new OIDC token and temporary AWS credentials.

---

# Part 7: Creating the OIDC Setup with Terraform

## 30. The Bootstrap Problem

Terraform can create the GitHub OIDC provider, IAM role, trust policy, and permissions policy.

However, Terraform needs AWS access before it can create these resources.

```text
Terraform needs authentication
to create the authentication setup.
```

The first setup can be created using one of these methods:

1. Create it manually through the AWS Console
2. Run Terraform locally using AWS SSO
3. Run Terraform with an existing approved administrative role
4. Use an organization-managed bootstrap pipeline

After the initial OIDC setup exists, GitHub Actions can use OIDC for later Terraform deployments.

---

## 31. Terraform OIDC Provider

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = [
    "sts.amazonaws.com"
  ]
}
```

### Why is this required?

This registers GitHub as an identity provider in your AWS account.

Before using exact code in production, confirm the currently required provider arguments against the AWS provider documentation for the version your project uses.

---

## 32. Terraform Trust Policy

```hcl
data "aws_iam_policy_document" "github_trust" {
  statement {
    effect = "Allow"

    actions = [
      "sts:AssumeRoleWithWebIdentity"
    ]

    principals {
      type = "Federated"

      identifiers = [
        aws_iam_openid_connect_provider.github.arn
      ]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"

      values = [
        "sts.amazonaws.com"
      ]
    }

    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"

      values = [
        "repo:my-company/terraform-aws-infrastructure:*"
      ]
    }
  }
}
```

`aws_iam_policy_document` helps Terraform build a correctly formatted IAM JSON policy.

---

## 33. Terraform IAM Role

```hcl
resource "aws_iam_role" "github_terraform" {
  name = "GitHubTerraformRole"

  assume_role_policy = data.aws_iam_policy_document.github_trust.json
}
```

The trust policy controls who can assume the role.

---

## 34. Terraform Permissions Policy

```hcl
data "aws_iam_policy_document" "terraform_permissions" {
  statement {
    effect = "Allow"

    actions = [
      "s3:ListBucket",
      "s3:GetObject",
      "s3:PutObject"
    ]

    resources = [
      "arn:aws:s3:::my-terraform-state-bucket",
      "arn:aws:s3:::my-terraform-state-bucket/*"
    ]
  }
}
```

Create the policy:

```hcl
resource "aws_iam_policy" "terraform_permissions" {
  name   = "GitHubTerraformPermissions"
  policy = data.aws_iam_policy_document.terraform_permissions.json
}
```

Attach it to the role:

```hcl
resource "aws_iam_role_policy_attachment" "terraform_permissions" {
  role       = aws_iam_role.github_terraform.name
  policy_arn = aws_iam_policy.terraform_permissions.arn
}
```

This is only a structural example. Your real deployment will require permissions for the AWS resources it manages.

---

## 35. Terraform Output for the Role ARN

```hcl
output "github_terraform_role_arn" {
  value = aws_iam_role.github_terraform.arn
}
```

After applying the Terraform code, the output will look similar to:

```text
github_terraform_role_arn =
"arn:aws:iam::123456789012:role/GitHubTerraformRole"
```

Copy this value into the GitHub Actions workflow.

---

# Part 8: Production Security Recommendations

## 36. Restrict the Repository

Avoid broad conditions such as:

```text
repo:*/*
```

Use a specific organization and repository:

```text
repo:my-company/terraform-aws-infrastructure:*
```

---

## 37. Restrict Production Access

For production, use a GitHub Environment such as:

```text
production
```

Protect that environment with required reviewers.

The AWS trust policy should match the correct GitHub OIDC subject claim for that environment.

---

## 38. Use Separate Roles

```text
GitHubTerraformDevRole
GitHubTerraformTestRole
GitHubTerraformProdRole
```

Example:

```text
Development workflow -> Development AWS role
Testing workflow     -> Testing AWS role
Production workflow  -> Production AWS role
```

The production role should have stricter permissions and approval requirements.

---

## 39. Use Least Privilege

Do not permanently attach:

```text
AdministratorAccess
```

Instead, allow only the services and actions Terraform requires.

If Terraform only manages S3 buckets, it should not receive permission to delete IAM users, change networking, create databases, or stop EC2 instances.

---

## 40. Protect the Terraform State

Terraform state may contain sensitive infrastructure information.

For production:

- Store state remotely
- Use an S3 backend
- Enable S3 versioning
- Enable server-side encryption
- Restrict access to the state bucket
- Use separate state files for environments
- Enable state locking using the currently supported Terraform S3 backend locking method
- Never commit `terraform.tfstate` to GitHub

OIDC authenticates the workflow, but state security must still be configured separately.

---

# Part 9: Common Errors

## 41. `Not authorized to perform sts:AssumeRoleWithWebIdentity`

Possible causes:

- The IAM role trust policy is incorrect
- The repository name does not match
- The organization name does not match
- The branch or environment does not match
- The audience is incorrect
- The OIDC provider is missing
- The workflow requested the wrong role ARN

---

## 42. `No OpenIDConnect provider found`

AWS cannot find the GitHub OIDC provider.

Check that this identity provider exists in the correct AWS account:

```text
token.actions.githubusercontent.com
```

---

## 43. `Credentials could not be loaded`

Check that the workflow contains:

```yaml
permissions:
  id-token: write
  contents: read
```

Also confirm that the AWS authentication step runs before Terraform.

---

## 44. `AccessDenied` During Terraform

If OIDC authentication succeeds but Terraform receives `AccessDenied`, authentication is probably working.

The IAM role likely does not have permission to perform the requested AWS action.

```text
Authentication succeeded:
GitHub successfully assumed the role.

Authorization failed:
The role cannot create the requested resource.
```

Authentication and authorization are different concepts.

---

# Part 10: Beginner Summary

## 45. What You Need to Remember

### GitHub OIDC provider

```text
Tells AWS that GitHub is a recognized identity provider.
```

### JWT

```text
A short-lived signed identity token created by GitHub.
```

### GitHub private key

```text
Used by GitHub to sign the JWT.
You do not manage this key.
```

### GitHub public key

```text
Used by AWS to verify the JWT signature.
You do not manually manage this key.
```

### AWS STS

```text
Verifies the request and provides temporary AWS credentials.
```

### IAM role

```text
The AWS identity that GitHub Actions temporarily assumes.
```

### Trust policy

```text
Controls who can assume the IAM role.
```

### Permissions policy

```text
Controls what the IAM role can do in AWS.
```

### Role ARN

```text
The address or unique identifier of the AWS IAM role.
It is not a password.
```

### Terraform

```text
Uses the temporary AWS credentials to create or change resources.
```

---

## 46. Final Simple Explanation

```text
1. GitHub starts the workflow.

2. GitHub creates a temporary identity token.

3. GitHub signs the token with its private key.

4. AWS verifies the token using GitHub's public key.

5. AWS checks whether the repository is trusted.

6. AWS checks which IAM role GitHub wants to use.

7. AWS gives temporary credentials for that role.

8. Terraform uses the temporary credentials.

9. AWS allows only the actions permitted by the role.

10. The temporary credentials expire automatically.
```

No permanent AWS access key needs to be stored in GitHub.

---

## Official References

- [GitHub Docs: Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)
- [AWS Security Blog: Use IAM roles to connect GitHub Actions to AWS](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/)
- [AWS Configure Credentials GitHub Action](https://github.com/aws-actions/configure-aws-credentials)
