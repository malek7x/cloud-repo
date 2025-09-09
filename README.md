AWS CI/CD Project Setup Guide
Overview
This project sets up a complete CI/CD pipeline on AWS using EC2, Docker, ECR, and GitHub Actions. The infrastructure includes a VPC with public and private subnets, DocumentDB, ElastiCache Redis, and an Auto Scaling Group behind an Application Load Balancer.

Architecture
text
Internet -> ALB (public subnet) -> EC2 Instances (private subnet) -> DocumentDB & Redis (private subnet)
Phase 1: Network Infrastructure (VPC + Subnets + Security Groups)
VPC Configuration
CIDR: 10.0.0.0/16

Public Subnet: 10.0.1.0/24 (us-east-1a)
Private Subnet: 10.0.10.0/24 (us-east-1a)
Dummy Private Subnet: 10.0.11.0/24 (us-east-1b) - Required for managed services

Security Groups
alb-sg (ALB Security Group)
Inbound: HTTP/80, HTTPS/443 from 0.0.0.0/0
Outbound: All traffic

app-sg (Application Security Group)
Inbound:
Port 3000 from alb-sg
Port 8080 from alb-sg
SSH/22 from My IP
Outbound: All traffic

docdb-sg (DocumentDB Security Group)
Inbound: Port 27017 from app-sg
Outbound: All traffic

redis-sg (ElastiCache Security Group)
Inbound: Port 6379 from app-sg
Outbound: All traffic

Routing
Public Route Table: Routes to Internet Gateway
Private Route Table: Routes to NAT Gateway

Phase 2: Data Layer (DocumentDB + ElastiCache)
DocumentDB Cluster
Cluster ID: my-app-cluster
Instance: db.t3.medium
Subnet Group: my-app-db-subnet-group (spans 2 AZs)
Security Group: docdb-sg
Authentication: Master username/password

ElastiCache Redis
Cluster: my-app-redis (Redis OSS)
Node Type: cache.t3.micro
Subnet Group: my-app-cache-subnet-group (spans 2 AZs)
Security Group: redis-sg

Phase 3: ECR + CI/CD Pipeline
ECR Repositories
ecom-backend

ecom-frontend

GitHub Actions Workflow (.github/workflows/ci-cd.yml) you can find the file there.

GitHub Secrets Required
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_REGION
ECR_REGISTRY

Phase 4: Compute Layer (EC2 + ASG + ALB)
IAM Role for EC2 (my-app-ec2-role)
Attached policies:
AmazonEC2ContainerRegistryReadOnly
CloudWatchAgentServerPolicy
AmazonSSMManagedInstanceCore

Launch Template (my-app-template)
AMI: Ubuntu Server 22.04 LTS
Instance Type: t3.micro
Key Pair: my-app-key (optional)
Security Group: app-sg
IAM Role: my-app-ec2-role
User Data: Installs Python3/pip for Ansible
Auto Scaling Group (my-app-asg)
Min/Max: 1/2 instances
Subnets: Private subnet only
Health Check: ELB health checks on /healthz
Application Load Balancer (my-app-alb)
Scheme: Internet-facing
Listeners: HTTP/80
Target Group: my-app-tg (port 3000/8080)
Health Check: /healthz

Phase 5: Configuration Management (Ansible) 
check the playbook in the Ansible/playbook

Phase 6: IAM User for CI/CD (ecom-github-actions)
Attached Policies
AmazonEC2ContainerRegistryPowerUser - Push/pull to ECR
AmazonEC2ReadOnlyAccess - Discover instances for Ansible
AmazonSSMStartSession - Connect via SSM for Ansible
Custom inline policy for ssm:StartSession

**********************************

Validation Checklist
VPC and subnets created

Security groups configured with proper rules

DocumentDB cluster available

ElastiCache Redis cluster available

ECR repositories created

GitHub Actions workflow successfully builds and pushes images

EC2 Launch Template and ASG configured

ALB created and healthy

Ansible can connect via SSM and deploy application

Application accessible via ALB DNS name

Health checks passing (/healthz endpoint)

************************************

Key AWS Services Used
EC2: Application hosting
VPC: Network isolation
DocumentDB: Managed MongoDB
ElastiCache: Redis caching
ECR: Docker image registry
IAM: Access control
ASG: Auto scaling
ALB: Load balancing
Systems Manager: Secure instance access
