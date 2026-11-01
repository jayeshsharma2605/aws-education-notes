# AWS Education Notes

*Author: Jayesh Sharma*

A beginner-friendly AWS learning repository with original explanations, practical examples, architecture diagrams, safety notes, and interview questions.

## Study Guides

Companion notes for the [25-video AWS from Basics to Advanced playlist](https://www.youtube.com/playlist?list=PL6W8uoQQ2c60g3PAgzVPpi0YWwxITD43p) by Concept && Coding – by Shrayansh. **Video 1 (AWS Roadmap)** is a brief course introduction; the detailed repository notes begin at Video 2.

**Scope:** Lesson titles and order follow the playlist. Full video transcripts were not accessible for the newly created videos 5–25, so these are **independently written topic-based study companions**, not verbatim summaries. Always use [official AWS documentation](https://docs.aws.amazon.com/) for current service configuration, pricing and security guidance.

### Foundations and global infrastructure

- [AWS Account Setup & Cloud Computing Models](AWS_Account_Setup_and_Cloud_Service_Models.md) — Account security, IaaS, PaaS, CaaS, FaaS and SaaS.
- [AWS Global Infrastructure (Part 1)](AWS_Global_Infrastructure_Part_1_Regions_AZs_Local_Zones_Edge.md) — Regions, AZs, Local Zones and CloudFront edge.
- [AWS Global Infrastructure (Part 2)](AWS_Global_Infrastructure_Part_2_Disaster_Recovery_Strategies.md) — Backup/restore, pilot light, warm standby and active/active.

### IAM and networking

- [AWS IAM Users Explained with Demo](05_IAM_Users_Explained.md)
- [AWS IAM Policies & User Groups](06_IAM_Policies_and_User_Groups.md)
- [AWS IAM Roles Explained](07_IAM_Roles_Explained.md)
- [AWS Networking (Part 1): Fundamentals](08_Networking_Fundamentals.md)
- [AWS Networking (Part 2): VPC, Subnets, NAT, IGW & Route Tables](09_VPC_Subnet_IGW_NAT_Route_Tables.md)
- [AWS Networking (Part 3): Security Groups and NACLs](10_Security_Groups_and_NACLs.md)

### Terraform and compute

- [Terraform Architecture & Core Blocks](11_Terraform_Architecture_and_Core_Blocks.md)
- [Terraform Hands-on: AWS Networking](12_Terraform_AWS_Networking_Hands_On.md)
- [EC2: AMI, Instance Types, Keys & Launch](13_EC2_AMI_Instance_Types_and_Launch.md)
- [Deploy Spring Boot Application on EC2](14_Deploy_Spring_Boot_on_EC2.md)
- [EC2 Auto Scaling Groups](15_EC2_Auto_Scaling_Groups.md)
- [Application Load Balancer with EC2 and ASG](16_Application_Load_Balancer_EC2_ASG.md)

### Storage, CDN and messaging

- [Elastic Block Store (EBS)](17_Elastic_Block_Store_EBS.md)
- [S3 Part 1: Architecture](18_S3_Part_1_Architecture.md)
- [S3 Part 2: Spring Boot Integration & Authentication](19_S3_Part_2_Spring_Boot_Integration.md)
- [S3 Part 3: Upload, Multipart Upload & Download](20_S3_Part_3_Upload_Multipart_Download.md)
- [S3 Part 4: Presigned URLs](21_S3_Part_4_Presigned_URLs.md)
- [CloudFront Part 1: CDN Setup](22_CloudFront_Part_1_Setup.md)
- [CloudFront Part 2: Advanced Setup](23_CloudFront_Part_2_Advanced_Setup.md)
- [Simple Queue Service (SQS)](24_SQS_Simple_Queue_Service.md)
- [Simple Notification Service (SNS)](25_SNS_Simple_Notification_Service.md)

## How to use these notes

1. Open the guide and read the simple examples first.
2. Use the Mermaid diagrams to connect the concepts visually (GitHub renders Mermaid blocks).
3. Answer the interview questions without looking at the notes.
4. For hands-on exercises, follow current AWS pricing and Free Tier guidance; delete unused resources after every lab.

## Planned subjects

- IAM and access control
- AWS networking and VPC
- EC2, storage, and load balancing
- Containers (ECS/EKS/Fargate)
- Serverless and event-driven architecture
- Databases, security, monitoring, and costs

> These learning notes are not official AWS documentation. Refer to the linked AWS documentation for current pricing, account plans, console screens, and service limits.
