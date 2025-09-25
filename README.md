# Guidance for Automated Setup for Elastic VMware Service (EVS) on AWS

## Table of Contents

1. [Overview](#overview)
    - [Architecture](#architecture)
    - [Cost](#cost)
3. [Prerequisites](#prerequisites)
    - [Operating System](#operating-system)
    - [AWS Account Requirements](#aws-account-requirements)
    - [Service Limits](#service-limits)
4. [Deployment Steps](#deployment-steps)
5. [Deployment Validation](#deployment-validation)
6. [Running the Guidance](#running-the-guidance)
7. [Next Steps](#next-steps)
8. [Cleanup](#cleanup)
9. [Important Notes and Limitations](#important-notes-and-limitations)
10. [Notices](#notices)
11. [Authors](#authors)

## Overview

This Guidance provides an automated solution for deploying Amazon Elastic VMware Service (EVS) environments using AWS CloudFormation. It simplifies the complex process of setting up EVS by automating the creation of required networking infrastructure, Route 53 DNS configuration, and EVS environment deployment.

### Architecture

[Insert architecture diagram here showing the components created by the template]

The CloudFormation template creates and configures:
1. VPC networking infrastructure
2. Route 53 DNS zones and records
3. VPC Route Server configuration
4. EVS environment with 4-node cluster

Key Components and Their Relationships:

1. VPC Infrastructure

 - Underlay VPC with specified CIDR block
 - Two subnets:
     Service Access Subnet (MyCIDR.0.0/24)
     Public Access Subnet (MyCIDR.5.0/24)
2. Networking Components
 - Internet Gateway for public internet access
 - NAT Gateway for private subnet internet access
 - Route Server (ASN 65022) with two endpoints
 - Optional Transit Gateway connection
3. DNS Infrastructure
 - Route 53 Resolver Endpoints
 - Forward and Reverse lookup zones
 - DHCP Options Set with custom DNS settings
4. EVS Environment

 - 4 ESXi hosts (i4i.metal instances)
 - vCenter Server
 - NSX Manager Cluster (3 nodes)
 - NSX Edge Cluster (2 nodes)
 - SDDC Manager
 - Cloud Builder

5. Network Segments (VLANs)

 - VMkernel Management (MyCIDR.10.0/24)
 - vMotion (MyCIDR.20.0/24)
 - vSAN (MyCIDR.30.0/24)
 - VTEP (MyCIDR.40.0/24)
 - Edge VTEP (MyCIDR.50.0/24)
 - VM Management (MyCIDR.60.0/24)
 - HCX (MyCIDR.70.0/24)
 - NSX Uplink (MyCIDR.80.0/24)
 - Expansion VLANs (MyCIDR.90.0/24 & MyCIDR.100.0/24)
   
### Cost

You are responsible for the cost of the AWS services used while running this Guidance. As of September 2025, the cost for running this Guidance with the default settings in the US East (N. Virginia) Region `us-east-1` is approximately _$8,000 per month_ for a standard 4 ESXi node deployment.

We recommend creating a [Budget](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) through [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/) to help manage costs. Prices are subject to change. For full details, refer to the pricing webpage for each AWS service used in this Guidance.

### Sample Cost Table

| AWS Service | Dimensions | Cost [USD] |
|-------------|------------|------------|
| Amazon EC2 (i4i.metal instances) | 4 instances per month | $7,603.20 |
| NAT Gateway | 1 NAT Gateway with data transfer | $32.85 |
| Route 53 | Hosted zones and queries | $1.00 |
| VPC Route Server | 1 Route Server with endpoints | $91.25 |
| Data Transfer | 1 TB outbound | $90.00 |

## Prerequisites

### Operating System

These deployment instructions are optimized for Amazon Linux 2. You'll need:
- AWS CLI v2
- Python 3.8 or later
- git CLI client

### AWS Account Requirements

1. AWS Business Support or higher subscription
2. Valid VMware Cloud Foundation (VCF) and vSAN license keys
3. VCF Site ID from Broadcom
4. ACM certificate (if using TLS)

## Important Notes and Limitations
1. DNS Configuration: This template only supports Route 53 as the DNS provider. For environments requiring custom DNS servers, please refer to the manual setup process in the Amazon EVS User Guide.
2. Single Availability Zone: This template deploys the EVS environment in a single Availability Zone.
3. Initial Configuration: The template provides a basic EVS environment. Additional configuration may be required based on your specific use case.

### Important Note About DNS Configuration
This CloudFormation template is designed to work exclusively with Amazon Route 53 for DNS management. If you need to use your own DNS server:
- Do not use this automation template
- Ensure your DNS server is resolvable within your VPC
- Follow the manual setup instructions in the [Amazon EVS User Guide](https://docs.aws.amazon.com/evs/latest/userguide/getting-started.html)

### Service Limits

The following service quotas must be available:
- EVS host count per environment: Minimum 4
- EC2 Running On-Demand i4i.metal instances: 512 vCPUs (4 instances × 128 vCPUs)

### Deployment Steps

1. Clone this project repository:
```bash
git clone https://github.com/aws-samples/aws-evs-automated-deployment.git
```

2. Navigate to the cloned directory:
```bash
cd [repository-name]
```

3. Deploy the CloudFormation template 

### Deploy From the CLI
```bash
aws cloudformation create-stack \
  --stack-name evs-environment \
  --template-body file://template.yaml \
  --parameters \
    ParameterKey=MyFQDN,ParameterValue=my.fqdn.evs \
    ParameterKey=MyCIDR,ParameterValue=10.0. \
    ParameterKey=MySiteId,ParameterValue=your-site-id \
    ParameterKey=MySolutionKey,ParameterValue=your-solution-key \
    ParameterKey=MyVsanKey,ParameterValue=your-vsan-key
```

## CLI Deployment Validation

1. Monitor the CloudFormation stack status in the AWS Console or using:
```bash
aws cloudformation describe-stacks --stack-name evs-environment
```



### Deploy Using AWS CloudFormation Console

1. Sign in to the AWS Management Console and open the CloudFormation console at https://console.aws.amazon.com/cloudformation/
2. Choose "Create stack" and then select "With new resources (standard)".
3. In the "Specify template" section, select "Upload a template file".
4. Click "Choose file" and navigate to the cloned repository directory.
5. Select the `evs-deployment-template.yaml` file and click "Open".
6. Click "Next".
7. On the "Specify stack details" page:
- Enter a Stack name (e.g., "EVS-Automated-Deployment")
- Fill in the required parameters:
  - MyFQDN: The fully qualified domain name for your EVS environment
  - MyCIDR: The base CIDR for the VPC (e.g., 10.0.)
  - MySiteId: Your Broadcom VCF Site ID
  - MySolutionKey: Your VCF solution license key
  - MyVsanKey: Your vSAN license key
  - (Fill in any other required parameters as specified in the template)
8. Click "Next".
9. On the "Configure stack options" page, you can add tags, set permissions, and configure advanced options if needed. For most deployments, you can leave these settings at their defaults.
10. Click "Next".
11. On the "Review" page, review your settings. Be sure to check the acknowledgment at the bottom of the page if your template creates IAM resources.
12. Click "Create stack".

CloudFormation will now begin creating the resources for your EVS environment. This process can take several hours to complete.

## Monitor Console Deployment Progress

1. On the CloudFormation console, select your stack.
2. Go to the "Events" tab to monitor the creation progress.
3. Once the status changes to "CREATE_COMPLETE", your EVS environment is ready.

## View Console Outputs

After the stack creation is complete:

1. Go to the "Outputs" tab of your stack.
2. Here you will find important information about your deployed resources, including:
- EVSEnvironmentId: The ID of your new EVS environment
- VPCId: The ID of the VPC created for your environment
- ServiceAccessSubnetId: The ID of the subnet used for service access
- Route53ForwardZoneId and Route53ReverseZoneId: IDs for the DNS zones
- Other relevant IDs and IP addresses

Make note of these outputs as they will be useful for further configuration and management of your EVS environment.

### Verify Environment
1. Once complete, verify EVS environment creation in the EVS AWS console or using command:
```bash
aws evs get-environment --environment-id [environment-id]
```

2. Validate Route 53 configuration:
   a. Check that the hosted zones are created:
   ```bash
   aws route53 list-hosted-zones
   ```
   b. Verify DNS records for EVS components:
   ```bash
   aws route53 list-resource-record-sets --hosted-zone-id [your-hosted-zone-id]
   ```
   c. Test DNS resolution for a few key components:
   ```bash
   nslookup [component-name].[your-domain] [route53-resolver-ip]
   ```

## Running the Guidance

After successful deployment, follow these steps to access your EVS environment:

1. Retrieve VCF credentials from AWS Secrets Manager following the [EVS User Guide](https://docs.aws.amazon.com/evs/latest/userguide/getting-started-retrieve-credentials.html)

2. Access your environment components:
   - vCenter Server
   - NSX Manager
   - SDDC Manager

For detailed instructions on using EVS, refer to the [Amazon EVS User Guide](https://docs.aws.amazon.com/evs/latest/userguide/).

## Next Steps

1. Configure additional security settings
2. Set up VMware HCX for migrations
3. Add additional hosts as needed
4. Configure backup and disaster recovery

## Cleanup

1. Delete the EVS environment:
```bash
aws evs delete-environment --environment-id [environment-id]
```

2. Delete the CloudFormation stack:
```bash
aws cloudformation delete-stack --stack-name evs-environment
```

3. Verify all resources are properly cleaned up in the AWS Console


## Notices

Customers are responsible for making their own independent assessment of the information in this Guidance. This Guidance: (a) is for informational purposes only, (b) represents AWS current product offerings and practices, which are subject to change without notice, and (c) does not create any commitments or assurances from AWS and its affiliates, suppliers or licensors. AWS products or services are provided "as is" without warranties, representations, or conditions of any kind, whether express or implied. AWS responsibilities and liabilities to its customers are controlled by AWS agreements, and this Guidance is not part of, nor does it modify, any agreement between AWS and its customers.

## Authors

- David Piet, Principal SA, Migration & Modernization
- Daniel Zilberman, Sr. Specialist SA, Technical Solutions
- Deepika Suresh, Specialist SA, Technical Solutions
- Johanna Wood, Technical Lead, Technical Solutions
