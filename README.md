# AWS VPN Gateway Deployment with CloudFormation and CI/CD Pipeline

This repository contains an AWS CloudFormation template for deploying a VPN Gateway (VGW) to establish a site-to-site VPN connection with Azure. It also includes a GitHub Actions CI/CD pipeline for automated deployment, validation, and security checks. The pipeline handles manual approvals and triggers parent stack deployments.

## Table of Contents

- [Introduction](#introduction)
- [CloudFormation Template](#cloudformation-template)
  - [Resources Created](#resources-created)
  - [Parameters](#parameters)
  - [Outputs](#outputs)
- [CI/CD Pipeline](#cicd-pipeline)
  - [Workflow Triggers](#workflow-triggers)
  - [Pipeline Overview](#pipeline-overview)
  - [Environment Variables and Secrets](#environment-variables-and-secrets)
- [Usage](#usage)
  - [Clone the Repository](#clone-the-repository)
  - [Set Up AWS Credentials](#set-up-aws-credentials)
  - [Configure the VPN Gateway](#configure-the-vpn-gateway)
  - [Branch Strategy](#branch-strategy)
  - [Manual Approval](#manual-approval)
  - [Triggering the Parent Stack](#triggering-the-parent-stack)
- [Notes](#notes)

## Introduction

This project automates the deployment of an AWS VPN Gateway (VGW) to establish a site-to-site VPN connection with Azure using CloudFormation. The VGW is attached to a specified VPC and creates a VPN connection to the Azure VPN Gateway. It includes necessary route tables and propagations for network traffic.

The GitHub Actions CI/CD pipeline automates validation, security scanning, deployment processes, and triggers the deployment of a parent stack via webhooks.

The CI/CD pipeline is designed to:

- Validate and test CloudFormation templates on pull requests.
- Deploy infrastructure on pushes to specific branches.
- Perform security checks using CloudFormation Guard.
- Require manual approval before deployment.
- Trigger a parent stack deployment after successful deployment.

## CloudFormation Template

### Resources Created

The CloudFormation template (`template.yaml`) provisions the following resources:

1. **VPN Gateway (VGW):**

   - **Type:** `ipsec.1`
   - **Amazon Side ASN:** `65001`
   - **Tags:** Includes `Name` with value `VGW_{Environment}`

2. **VGW Attachment (`VGWAttachment`):**

   - Attaches the VGW to the specified VPC (`VPCId` parameter).

3. **Customer Gateway (`CustomerGateway`):**

   - **IP Address:** Defined by `CustomerGatewayIP` parameter (default is `168.62.22.228`)
   - **Type:** `ipsec.1`
   - **BGP ASN:** `65002`
   - **Tags:** Includes `Name` with value `AzureVPN_{Environment}`

4. **VPN Connection (`VPNConnection`):**

   - Establishes a VPN connection between the VGW and the Customer Gateway.
   - **Type:** `ipsec.1`
   - **Static Routes Only:** `false` (enables dynamic routing with BGP)
   - **Pre-Shared Key:** Provided via `PreSharedKey` parameter (sensitive)
   - **Tunnel Options:**
     - Tunnel 1: Inside CIDR `169.254.21.0/30`
     - Tunnel 2: Inside CIDR `169.254.22.0/30`
   - **Tags:** Includes `Name` with value `S2SVPN_{Environment}`

5. **Route Table (`RouteTable`):**

   - Created for the specified VPC.
   - **Tags:** Includes `Name` with value `RouteTable_{Environment}`

6. **Subnet Route Table Association (`SubnetRouteTableAssociation`):**

   - Associates the route table with the specified subnet (`SubnetId` parameter).

7. **VGW Route Propagation (`VGWRoutePropagation`):**

   - Propagates routes from the VGW to the route table.
   - Ensures that traffic destined for the Azure network is routed through the VPN connection.

### Parameters

The template accepts the following parameters:

- **VPCId:**

  - **Type:** String
  - **Description:** The VPC ID to which the VGW will attach

- **SubnetId:**

  - **Type:** String
  - **Description:** The subnet ID to which the route table will attach

- **CustomerGatewayIP:**

  - **Type:** String
  - **Description:** The IP address of the Customer Gateway (Azure VPN Gateway)
  - **Default:** `168.62.22.228`

- **AzureCIDR:**

  - **Type:** String
  - **Description:** The CIDR block of the Azure VNet
  - **Default:** `10.0.0.0/16`

- **Environment:**

  - **Type:** String
  - **Description:** The environment name (provided via Parent Stack)
  - **Allowed Values:** `development`, `production`, `default`

- **PreSharedKey:**

  - **Type:** String
  - **Description:** The pre-shared key used to establish the VPN connection (provided via secrets)
  - **NoEcho:** `true` (sensitive parameter)

### Outputs

- **VGWId:**

  - **Description:** The ID of the created VPN Gateway
  - **Value:** Reference to `VGW`

## CI/CD Pipeline

The CI/CD pipeline is defined in the GitHub Actions workflow file `.github/workflows/aws-cloudformation.yml`. It automates the deployment process, ensures code quality and security, and triggers the deployment of a parent stack.

### Workflow Triggers

The pipeline is triggered on:

- **Pull Requests** to the following branches:

  - `development`
  - `production`
  - `testing`

- **Pushes** to the following branches:

  - `development`
  - `production`

### Pipeline Overview

The pipeline consists of two primary jobs:

1. **Validate and Test (`validate-and-test`):**

   - **Checkout Code:** Retrieves the repository code.
   - **Configure AWS Credentials:** Assumes an IAM role using OpenID Connect (OIDC) with the provided credentials.
   - **Validate CloudFormation Template:** Validates the syntax of the CloudFormation template.
   - **Run CloudFormation Guard:** Performs security checks using CloudFormation Guard against CIS benchmarks.
   - **Skip Apply in Pull Requests:** Ensures that deployment does not occur on pull requests.

2. **Upload and Trigger (`upload-and-trigger`):**

   - **Depends On:** The `validate-and-test` job must succeed.
   - **Runs On:** Not triggered on pull requests.
   - **Checkout Code:** Retrieves the repository code.
   - **Configure AWS Credentials:** Assumes an IAM role using OIDC with the provided credentials.
   - **Set Environment Variables:** Determines the environment and S3 bucket based on the branch.
   - **Manual Approval:** Requires manual approval via GitHub Issues before proceeding.
   - **Upload Template to S3:** Uploads the CloudFormation template to the specified S3 bucket.
   - **Trigger Parent Stack Deployment:** Makes an API call to trigger the parent stack's workflow via webhooks.

### Environment Variables and Secrets

The pipeline uses the following secrets and environment variables:

- **Secrets (Stored in GitHub Secrets):**

  - `AWS_ROLE_TO_ASSUME`
  - `TOKEN_PARENT_TRIGGER`

  - **`AWS_ROLE_TO_ASSUME`**: The ARN of an IAM role that the GitHub Actions workflow can assume, with the necessary permissions to deploy CloudFormation stacks and interact with S3.

  - **`TOKEN_PARENT_TRIGGER`**: A GitHub Personal Access Token (PAT) with permissions to trigger workflows in the parent repository.

- **Environment Variables:**

  - `ENVIRONMENT`: Set based on the branch (`development`, `production`, or `default`).
  - `S3_BUCKET`: The S3 bucket URL, determined by the environment.

## Usage

### Clone the Repository

```bash
git clone https://github.com/CommittingLearning/Site2Site-AWS-VGW.git
```

### Set Up AWS Credentials

Ensure that the following secrets are added to your GitHub repository under **Settings > Secrets and variables > Actions**:

- `AWS_ROLE_TO_ASSUME`
- `TOKEN_PARENT_TRIGGER`

- **`AWS_ROLE_TO_ASSUME`**: The ARN of an IAM role that the GitHub Actions workflow can assume, with the necessary permissions to deploy CloudFormation stacks and interact with S3.

- **`TOKEN_PARENT_TRIGGER`**: A GitHub Personal Access Token (PAT) with permissions to trigger workflows in the parent repository.

### Configure the VPN Gateway

- **VPC ID and Subnet ID**: These are provided by the outputs from the VPC stack. Ensure that you supply the correct IDs when deploying the parent stack.

- **Customer Gateway IP**: Adjust the `CustomerGatewayIP` parameter if the Azure VPN Gateway IP address is different.

- **Azure CIDR**: Change the `AzureCIDR` parameter to match the Azure VNet's CIDR block.

- **Pre-Shared Key**: Ensure that the `PreSharedKey` parameter is provided securely (e.g., via parameter overrides or secrets).

- **Environment Parameter**: The environment (`development`, `production`, or `default`) is determined based on the branch name.

### Branch Strategy

- **Development Environment**: Use the `development` branch to deploy to the development environment.

- **Production Environment**: Use the `production` branch to deploy to the production environment.

- **Default Environment**: Any other branches will use the `default` environment settings.

### Manual Approval

The pipeline requires manual approval before applying changes:

- A GitHub issue will be created prompting for approval.
- Approvers need to approve the issue to proceed with deployment.

### Triggering the Parent Stack

After successful deployment, the pipeline triggers the parent stack deployment via a webhook:

- The last step in the `upload-and-trigger` job makes an API call to the parent repository (`Site2Site-AWS-ParentStack`) to trigger its workflow.
- This ensures that the parent stack is deployed with the latest updates.

## Notes

- **Security Checks:**
  - The pipeline includes security checks using CloudFormation Guard to ensure compliance with CIS benchmarks.

- **Nested CloudFormation Templates:**
  - The VGW template is intended to be uploaded to an S3 bucket and used as a child template by a parent stack.

- **Testing:**
  - Pull requests to `development`, `production`, or `testing` branches will trigger the validation and testing steps without applying changes.

- **IAM Role Configuration:**
  - Ensure that the IAM role specified in `AWS_ROLE_TO_ASSUME` has permissions to deploy CloudFormation stacks, interact with S3, and invoke API calls to other repositories.

- **S3 Bucket Configuration:**
  - The S3 bucket URL is set based on the environment. Ensure that the bucket exists and is accessible.

- **Webhook Trigger:**
  - The `TOKEN_PARENT_TRIGGER` must have appropriate permissions (usually `repo` scope) to trigger workflows in the parent repository.
  - Adjust the API endpoint and payload in the `Trigger Parent Stack Deployment` step if your repository names or structures differ.

- **Deployment Order:**
  - **Important:** The VNet Gateway Terraform code must be deployed first.
  - After the Azure VPN Gateway is deployed, apply the AWS VPN configuration using CloudFormation from its respective repository.
  - Then, update this Terraform code to accurately reflect the public IP addresses of the AWS tunnels.
  - This specific order is necessary because AWS CloudFormation throws an error if you attempt to change the public IP address of the customer gateway after it has been created.

---

**Disclaimer:** This repository is accessible in a read only format, and therefore, only the admin has the privileges to perform a push on the branches.