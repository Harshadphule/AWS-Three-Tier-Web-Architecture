AWS-Three-Tier-Web-Architecture

AWS Project — Deploying a 3-Tier Architecture Using AWS Services
Description

This project demonstrates how to design and deploy a highly available, fault-tolerant, and secure 3-tier architecture on AWS from the ground up. It separates an application into three independent layers — Web Tier, Application Tier, and Database Tier — each isolated in its own subnet and protected by tightly scoped security groups, so a failure or compromise in one tier cannot directly reach another.

The infrastructure is built manually across two Availability Zones for redundancy, using Auto Scaling Groups and Load Balancers at each tier boundary, IAM roles instead of SSH keys for instance access, and native AWS monitoring, logging, and edge/security services layered on top.

Purpose
Create a highly available infrastructure across multiple Availability Zones.
Create a fault-tolerant infrastructure that keeps running through instance or AZ failure.
Ensure the infrastructure is highly secured — no SSH, no key pairs, least-privilege IAM, and edge protection via WAF/Shield.
AWS Services Used
Category	Services
Compute & Scaling	EC2, Auto Scaling Group (ASG)
Load Balancing	Application Load Balancer (External + Internal ALB)
Networking	VPC, Subnets, Internet Gateway, NAT Gateway, Route Tables
Database	RDS (MySQL, Multi-AZ)
Access & Security	IAM Roles, Security Groups, WAF, Shield
DNS & CDN	Route 53, CloudFront
Storage	S3
Monitoring & Auditing	CloudWatch, SNS, CloudTrail
Architecture Overview

The application traffic flows through the following path:

User
  │
  ▼
Route 53  ──▶  CloudFront  ──▶  External Load Balancer (Port 80)
                                        │
                         ┌──────────────┴──────────────┐
                         ▼                              ▼
                 WEB-TIER-SUBNET-1               WEB-TIER-SUBNET-2  (AZ-1a / AZ-1b)
                 Web Server (Nginx + Node.js/React)
                         │  Port 80
                         ▼
                Internal Load Balancer (Port 4000)
                         │
                         ▼
                 APP-TIER-SUBNET-1 / 2
                 App Server (Node.js backend + MySQL client)
                         │  Port 3306
                         ▼
                DATABASE-TIER-SUBNET-1 / 2
                RDS MySQL — Multi-AZ (Primary + Standby)

Traffic and port map:

Hop	Port	Notes
Client → External ALB	80 (HTTP)	Public entry point, fronted by CloudFront + Route 53
External ALB → Web Tier	80	Nginx serves the React frontend
Web Tier → Internal ALB	80	Internal-only, not internet-facing
Internal ALB → App Tier	4000	Node.js backend API
App Tier → DB Tier	3306	MySQL

Network layout (VPC 10.0.0.0/16):

Subnet	AZ	CIDR
Web-Tier-Subnet-1	AZ-1a	10.0.1.0/24
App-Tier-Subnet-1	AZ-1a	10.0.2.0/24
DB-Tier-Subnet-1	AZ-1a	10.0.3.0/24
Web-Tier-Subnet-2	AZ-1b	10.0.4.0/24
App-Tier-Subnet-2	AZ-1b	10.0.5.0/24
DB-Tier-Subnet-2	AZ-1b	10.0.6.0/24

Only the Web Tier subnets have a route to the Internet Gateway (public). The App Tier reaches the internet outbound only through NAT Gateways sitting in the Web Tier subnets (for patches/updates), and the DB Tier has no route to the internet at all.

Supporting services wrap around this core:

IAM roles (no SSH, no key pairs) grant EC2 instances S3 read access and Systems Manager (SSM) access, so servers are managed via Session Manager instead of exposing port 22.
S3 hosts the application code/build artifacts and stores VPC Flow Logs.
CloudWatch + SNS monitor CPU usage on instances and email-alert on thresholds.
CloudTrail audits all API activity across the account.
Route 53 + CloudFront + WAF/Shield sit at the edge for DNS resolution, caching, and protection against common web exploits and DDoS.

Architecture diagrams:

md
![3-Tier Flow](docs/images/3-tier-flow.png)
![Full Infrastructure Diagram](docs/images/full-architecture.png)



Algorithm

The build follows a strict sequence — each step depends on the resources created in the previous one.

Step 1 — Get the Application Code

Download/clone the application source code (frontend + backend) to your local system so it's ready to upload.

Step 2 — Create an S3 Bucket and Upload the Code
Create an S3 bucket with a globally unique name.
Upload the application code (frontend build and backend source) to this bucket.
This bucket becomes the source EC2 instances pull code from at launch (via the IAM role, not credentials).

Security note: Only port 80 is ever exposed to the internet. EC2 instances are not accessed via SSH and no key pairs are created — all instance access goes through AWS Systems Manager Session Manager, which means there's no open SSH port and no key to leak. This is a major part of what makes the server "highly secure."

Step 3 — Create an IAM Role with Policies

Create one IAM role and attach it to every EC2 instance (web + app tiers), with:

AmazonS3ReadOnlyAccess — so instances can pull the application code/build artifacts from S3.
AmazonSSMManagedInstanceCore — so instances can be managed through Session Manager instead of SSH.
Step 4 — Create VPC, Subnets, IGW, NAT Gateway, and Route Tables
Create a VPC (10.0.0.0/16).
Create six subnets across two AZs — Web, App, and DB tier subnets in each AZ (see table above). Enable auto-assign public IP only on the web-tier public subnets.
Create an Internet Gateway and attach it to the VPC.
Create a Web-Tier Route Table:
Add route 0.0.0.0/0 → Internet Gateway.
Associate it with both Web-Tier subnets.
Create NAT Gateways (nat-gw-1, nat-gw-2), one in each Web-Tier subnet (one per AZ, for redundancy).
Create an App-Tier Route Table per AZ:
Add a route to the corresponding NAT Gateway.
Associate it with the App-Tier subnet in that AZ.
Create an S3 bucket (or reuse the one from Step 2) and enable VPC Flow Logs, pointing them at that bucket (VPC → Flow Logs → create → select the S3 bucket ARN as destination).
Step 5 — Create Security Groups

Chain each security group so it only accepts traffic from the layer directly in front of it:

Security Group	Inbound Rule
External-Load-Balancer-SG	HTTP (80) from 0.0.0.0/0
Web-Tier-SG	HTTP (80) from External-LB-SG
Internal-Load-Balancer-SG	HTTP (80) from Web-Tier-SG
App-Tier-SG	Custom TCP (4000) from Internal-LB-SG
DB-Tier-SG	MySQL/Aurora (3306) from App-Tier-SG

No tier is ever reachable directly from the internet except the External Load Balancer.

Step 6 — Create the DB Subnet Group and RDS Instance
Create a DB Subnet Group using the two DB-tier subnets (one per AZ).
Create an RDS MySQL instance in Multi-AZ mode, placing it in the DB subnet group above, and attach the DB-Tier-SG.
Step 7 — Build and Test the App Tier
Launch a test EC2 instance in an App-Tier subnet, attach the IAM role and App-Tier-SG.
Install required packages (Node.js, MySQL client) and test connectivity to the RDS endpoint.
Once verified, create an AMI from this instance.
Create a Launch Template from the AMI.
Create a Target Group for the app tier (port 4000).
Create the Internal Load Balancer, pointing to that target group.
Create an Auto Scaling Group using the launch template, spanning both App-Tier subnets.
Edit the local nginx.conf to point to the Internal-LB DNS name, then upload the updated config to S3 (so web-tier instances pick it up at launch).
Step 8 — Build and Test the Web Tier
Launch a test EC2 instance in a Web-Tier subnet, attach the IAM role and Web-Tier-SG.
Install Nginx and Node.js (serving the React build), pulling the app code and the updated nginx.conf from S3.
Test connectivity end-to-end (Web → Internal ALB → App → RDS).
Create an AMI from this instance.
Create a Launch Template from the AMI.
Create a Target Group for the web tier (port 80).
Create the External Load Balancer, pointing to that target group.
Create an Auto Scaling Group using the launch template, spanning both Web-Tier subnets.
Step 9 — Configure Route 53

Add a DNS record in Route 53 pointing your domain to the External ALB DNS name (optionally routed through CloudFront, with WAF/Shield attached for edge protection).

Step 10 — Configure CloudWatch Alarms with SNS

Create a CloudWatch alarm (e.g., on CPU utilization) for the ASGs, and connect it to an SNS topic so you receive an email/notification when thresholds are breached.

Step 11 — Enable CloudTrail

Create a CloudTrail trail to log and audit all API activity across the account for governance and troubleshooting.

Step 12 — Tear Down

When done, delete resources in reverse dependency order to avoid orphaned/billed resources:

Delete CloudFront distribution → Route 53 records.
Delete Auto Scaling Groups (this terminates the EC2 instances).
Delete Load Balancers (External + Internal) and their Target Groups.
Delete Launch Templates and AMIs (deregister + delete backing snapshots).
Delete the RDS instance (and its subnet group).
Delete NAT Gateways (and release any associated Elastic IPs).
Delete the Internet Gateway (detach, then delete).
Delete Route Tables, Subnets, and finally the VPC.
Delete the IAM role/policies.
Empty and delete the S3 bucket(s).
Delete the CloudWatch alarm, SNS topic, and CloudTrail trail.
Key Design Highlights
No SSH, no key pairs — all instance access is via Session Manager, closing off the most commonly attacked port entirely.
Least-privilege IAM — instances only get S3 read + SSM core, nothing more.
Defense in depth via security group chaining — each tier only accepts traffic from the specific SG in front of it, never from 0.0.0.0/0 beyond the external ALB.
Multi-AZ everywhere — subnets, NAT gateways, ASGs, and RDS are all duplicated across two AZs.
Full observability — VPC Flow Logs, CloudWatch + SNS alarms, and CloudTrail auditing are built in from the start, not bolted on afterward.