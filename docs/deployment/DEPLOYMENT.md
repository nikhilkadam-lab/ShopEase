# ShopEase — AWS Deployment Documentation

## 1. Overview

ShopEase is a deployment-focused e-commerce application built as the application workload for an AWS cloud infrastructure case study.

The application is developed with Python and Flask and uses MySQL for structured application data. It supports customer and administrator workflows such as product browsing, authentication, wishlist, cart, checkout, order placement, order lifecycle management, product management, traffic tracking, revenue information, and image management.

The application is designed so that its image-storage backend can be switched between local storage and Amazon S3 through configuration. In the AWS version, the intended application data path is:

```text
Users
   |
   v
Application Load Balancer
   |
   v
EC2 Application Instances
   |
   +---------------------> Amazon S3
   |                         Product / uploaded images
   |
   +---------------------> Amazon RDS for MySQL
                             Application database
```

The AWS case study expands this into a highly available architecture using multiple Availability Zones, public and private subnets, an ALB, Auto Scaling, RDS, S3, EFS, EBS, CloudWatch, SNS, IAM, a bastion host, and VPC Flow Logs.

> **Documentation note:** The case study describes the target AWS infrastructure requirements. The application repository contains the implementation needed for the Flask workload, including RDS/MySQL configuration and an S3-backed storage implementation. Components that are infrastructure-level requirements of the case study are described separately from application code so that the documentation does not imply that a resource is created automatically by the application.

---

# 2. Architecture Goals

The deployment is designed around the following goals from the AWS case study:

- High availability across at least two Availability Zones.
- Horizontal scaling during traffic spikes.
- Separation of public, application, and database network tiers.
- No direct internet access to the database tier.
- Managed MySQL database using Amazon RDS.
- Object storage for product images and other uploaded assets.
- Controlled administrative access.
- Load balancing through an Application Load Balancer.
- Monitoring and alerting through CloudWatch and SNS.
- Shared storage where required by the infrastructure design.
- Performance testing under increasing traffic.

The case study identifies the original single-server problems as CPU saturation during flash sales, lack of backups, shared root SSH credentials, lack of monitoring, local serving of static files, and lack of disaster recovery. The proposed AWS architecture addresses these concerns by separating responsibilities across managed AWS services.

---

# 3. Reference AWS Architecture

The case study specifies a VPC using CIDR `10.0.0.0/16` with at least two Availability Zones.

A logical deployment is:

```text
                              Internet
                                  |
                                  v
                         Internet Gateway
                                  |
                 +----------------+----------------+
                 |                                 |
           Public Subnet AZ-1                Public Subnet AZ-2
                 |                                 |
                 |                            Application
                 |                            Load Balancer
                 |                                 |
          Bastion Host                            |
                 |                                 |
                 +----------------+----------------+
                                  |
                         Private App Subnets
                       AZ-1               AZ-2
                        |                   |
                     EC2 #1              EC2 #2
                        |                   |
                        +---------+---------+
                                  |
                         Private DB Subnets
                       AZ-1               AZ-2
                                  |
                            RDS MySQL
                             Multi-AZ

                    Amazon S3 <---- Application
                    Amazon EFS <--- Shared filesystem
                    CloudWatch ----> Monitoring
                    SNS -----------> Alerts
```

The exact subnet CIDRs beyond the required VPC CIDR are deployment choices and should be documented according to the actual AWS environment used.

---

# 4. Network Layer — VPC

## 4.1 VPC

Create a custom VPC with:

```text
CIDR: 10.0.0.0/16
```

The case study requires at least two Availability Zones.

The network is divided into:

- Public subnets for internet-facing components.
- Private application subnets for EC2 application instances.
- Private database subnets for RDS.

## 4.2 Internet Gateway

Attach an Internet Gateway to the VPC.

The public route table uses the Internet Gateway for internet-bound traffic.

```text
Public Subnet
      |
Route Table
      |
0.0.0.0/0
      |
Internet Gateway
      |
Internet
```

## 4.3 NAT Gateway

The case study requires a NAT Gateway so private application instances can initiate outbound connections for updates without being directly exposed to inbound internet traffic.

The NAT Gateway belongs in a public subnet.

```text
Private EC2
     |
Private Route Table
     |
NAT Gateway
     |
Internet Gateway
     |
Internet
```

## 4.4 Route Tables

Maintain separate routing for the public and private tiers.

Typical logical design:

### Public route table

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> Internet Gateway
```

### Private application route table

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> NAT Gateway
```

### Database route table

The database tier should not have a route that provides direct internet access.

---

# 5. Security Groups and Network Isolation

The architecture separates access by security group.

A logical security-group model is:

```text
Internet
   |
   v
ALB Security Group
   |
   v
EC2 Application Security Group
   |
   v
RDS Security Group
```

## ALB

Allow HTTP/HTTPS traffic according to the deployed application requirements.

## EC2

Allow application traffic from the ALB security group.

Administrative SSH access should be restricted to the intended bastion/jump-server path rather than exposing application instances directly.

## RDS

Allow MySQL port `3306` only from the EC2/application security group.

The case study explicitly requires that the database must never be directly accessible from the internet.

---

# 6. IAM

The case study defines three logical teams:

- Admins
- Developers
- Finance

Each team should have an IAM group with permissions appropriate to its responsibilities.

Individual IAM users should be used instead of sharing AWS root credentials.

The case study also requires:

- MFA for administrator users.
- An EC2 instance role allowing web servers to read from S3 and write CloudWatch metrics.
- An RDS-specific policy preventing developers from deleting production databases.

The application itself does not contain IAM configuration; IAM is an AWS infrastructure responsibility.

---

# 7. Amazon S3 — Application Image Storage

ShopEase contains a storage abstraction that supports both local storage and S3 storage.

The selection is controlled by:

```env
STORAGE_TYPE=s3
```

The application expects:

```env
S3_BUCKET=<bucket-name>
AWS_REGION=<aws-region>
```

The S3 implementation uses `boto3` and supports:

- Uploading objects.
- Checking whether an object exists.
- Reading objects.
- Deleting objects.

Uploaded images are processed by the application before being sent to the storage layer.

## Example configuration

```env
STORAGE_TYPE=s3
S3_BUCKET=your-shopease-bucket
AWS_REGION=ap-south-1
```

Do not commit real AWS credentials or secrets to GitHub.

For EC2 deployment, prefer an IAM instance role with the minimum S3 permissions required by the application.

---

# 8. Amazon RDS — MySQL Database

The application is configured to connect to MySQL using environment variables:

```env
DB_HOST=<RDS endpoint>
DB_PORT=3306
DB_NAME=shopease
DB_USER=admin
DB_PASSWORD=<password>
```

The Flask application uses:

- SQLAlchemy / Flask-SQLAlchemy
- PyMySQL
- MySQL

The application database contains structured information such as:

- Users
- Admins
- Products
- Categories
- Subcategories
- Brands
- Product sizes
- Product images
- Cart items
- Wishlist items
- Addresses
- Orders
- Order items
- Banners
- Traffic events

## RDS configuration from the case study

The case study specifies:

- MySQL 8.0.
- `db.t3.micro` for the case-study deployment.
- Multi-AZ mode.
- Automated backups.
- 7-day backup retention.
- Backup window: `2:00–3:00 AM IST`.
- Database subnet group using the two private DB subnets.
- Port `3306` accessible only from the EC2 application security group.

---

# 9. Database Initialization

The repository contains:

```text
database/shopease_structure.sql
```

This file contains the database structure required by the application.

A typical deployment sequence is:

1. Create the RDS MySQL instance.
2. Configure the RDS security group.
3. Verify that the EC2 application tier can reach RDS on port `3306`.
4. Create/select the `shopease` database.
5. Import the database structure.
6. Configure the application's `.env` file with the RDS endpoint and credentials.
7. Start the Flask application.
8. Verify application database operations.

Example environment configuration:

```env
DB_HOST=<RDS endpoint>
DB_PORT=3306
DB_NAME=shopease
DB_USER=<database-user>
DB_PASSWORD=<database-password>
```

---

# 10. EC2 Application Tier

The ShopEase application is a Python/Flask application.

The repository provides:

```text
requirements.txt
run.py
app/
templates/
static/
database/
```

The application factory is implemented in:

```text
app/__init__.py
```

The configuration is implemented in:

```text
app/config.py
```

## Application dependencies

Install the Python dependencies from:

```bash
pip install -r requirements.txt
```

The current dependency set includes Flask, SQLAlchemy, PyMySQL, boto3, Pillow, python-dotenv, Flask-WTF and related packages.

## Environment configuration

Create a local environment file:

```bash
cp .env.example .env
```

Then configure:

```env
SECRET_KEY=<strong-secret>
DB_HOST=<RDS endpoint>
DB_PORT=3306
DB_NAME=shopease
DB_USER=<database-user>
DB_PASSWORD=<database-password>

STORAGE_TYPE=s3
S3_BUCKET=<bucket-name>
AWS_REGION=ap-south-1
```

Never commit `.env`.

The repository's `.gitignore` already excludes:

```text
.env
venv/
uploads/*
*.log
```

---

# 11. Running the Application

The repository's development entry point is:

```text
run.py
```

It creates the Flask application and listens on:

```text
0.0.0.0:5000
```

For development/testing, the application can be started with:

```bash
python run.py
```

The current `run.py` uses Flask's development server with `debug=True`.

For a production deployment, the application should be served through a production WSGI server and placed behind the ALB. A production WSGI server is an infrastructure/deployment concern rather than part of the current application source.

---

# 12. Bastion Host

The case study specifies a bastion host:

```text
Instance type: t3.micro
Location: Public subnet
Public IP: Yes
Purpose: Administrative access
```

The bastion host is intended to be the controlled entry point for administration.

The application instances should remain private and should not require individual public IP addresses.

A logical access path is:

```text
Administrator
     |
     v
Bastion Host
     |
     v
Private EC2 Application Instance
```

SSH access should use key-based authentication and restricted security-group rules.

---

# 13. Application Load Balancer

The case study requires an Application Load Balancer in the public subnets.

The ALB:

- Provides a single entry point for users.
- Distributes requests to application EC2 instances.
- Performs target health checks.
- Works with the Auto Scaling Group.

Logical flow:

```text
Client
  |
  v
Application Load Balancer
  |
  +------> EC2 Instance 1
  |
  +------> EC2 Instance 2
  |
  +------> EC2 Instance N
```

The case study specifies the ALB listener on port `80`.

---

# 14. Target Group

Create a target group for the EC2 application instances.

The target group is attached to the ALB and is used for:

- Routing traffic.
- Health checking.
- Removing unhealthy instances from service.

The health-check path should be selected according to the application endpoints available in the deployed version.

---

# 15. Auto Scaling Group (ASG)

Auto Scaling is a core part of the ShopEase AWS architecture. The purpose of the **Auto Scaling Group (ASG)** is to maintain the required number of application EC2 instances and automatically adjust capacity when traffic changes.

The ASG sits behind the Application Load Balancer:

```text
                         Application Load Balancer
                                  |
                            Target Group
                                  |
                    +-------------+-------------+
                    |             |             |
                  EC2 #1        EC2 #2        EC2 #N
                    |             |             |
                    +-------------+-------------+
                                  |
                         Auto Scaling Group
```

The case study specifies the following ASG configuration:

| Setting | Required value |
|---|---|
| Minimum capacity | 1 |
| Desired capacity | 2 |
| Maximum capacity | 4 |
| Scale-out condition | CPU > 60% for 2 minutes |
| Scale-in condition | CPU < 30% for 5 minutes |
| Traffic distribution | Application Load Balancer |
| Instance source | Launch Template |

## 15.1 Launch Template

The ASG uses a Launch Template rather than manually creating each EC2 instance.

The Launch Template should define the standard application-server configuration, including:

- AMI created from the prepared application server.
- Instance type.
- IAM instance profile.
- Security group.
- Network/subnet configuration.
- Storage configuration.
- User-data/bootstrap commands if required.

This makes newly launched instances consistent with the existing application tier.

## 15.2 ASG Deployment Flow

The deployment sequence is:

```text
Prepare application EC2 instance
          |
          v
Install and configure ShopEase
          |
          v
Test application
          |
          v
Create Golden AMI
          |
          v
Create Launch Template
          |
          v
Create Target Group
          |
          v
Create Application Load Balancer
          |
          v
Create Auto Scaling Group
          |
          v
Attach ASG to Target Group
          |
          v
Configure scaling policies
```

## 15.3 Capacity Configuration

Set:

```text
Minimum capacity = 1
Desired capacity = 2
Maximum capacity = 4
```

A desired capacity of two instances provides an application tier with more than one running instance under normal operation and allows the ALB to distribute requests between instances.

## 15.4 Scale-Out Policy

Configure a scale-out policy to add one EC2 instance when:

```text
Average CPU utilization > 60%
for 2 consecutive minutes
```

Example:

```text
EC2 #1 + EC2 #2
       |
CPU increases
       |
CPU > 60% for 2 minutes
       |
       v
ASG launches EC2 #3
       |
       v
ALB registers healthy instance
       |
       v
Traffic can be distributed to EC2 #3
```

The maximum capacity remains four instances.

## 15.5 Scale-In Policy

Configure a scale-in policy to remove one EC2 instance when:

```text
Average CPU utilization < 30%
for 5 consecutive minutes
```

Example:

```text
EC2 #1 + EC2 #2 + EC2 #3
          |
     Traffic drops
          |
CPU < 30% for 5 minutes
          |
          v
ASG terminates one instance
          |
          v
Remaining healthy instances
continue serving traffic
```

The ASG must never scale below the configured minimum capacity.

## 15.6 ALB and ASG Relationship

The ALB does not create EC2 instances itself. The responsibilities are separated:

- **ALB** receives user traffic and distributes requests.
- **Target Group** keeps track of application instances and health.
- **ASG** creates or terminates EC2 instances according to capacity rules.
- **Launch Template** defines how a new EC2 instance should be created.
- **CloudWatch metrics/alarms** provide the measurements used by scaling policies.

The complete relationship is:

```text
Users
  |
  v
ALB
  |
  v
Target Group
  |
  +-------- EC2 Instance 1
  |
  +-------- EC2 Instance 2
  |
  +-------- EC2 Instance 3
  |
  +-------- EC2 Instance 4
              ^
              |
        Auto Scaling Group
              ^
              |
       Launch Template
```

## 15.7 Important Implementation Note

The **ShopEase application source code does not itself create or configure the AWS Auto Scaling Group**. ASG, ALB, Target Groups, Launch Templates and related AWS resources are infrastructure resources configured separately in AWS.

The supplied AWS case study explicitly requires the ASG configuration above, while the application repository provides the Flask workload that runs on the EC2 instances.

Therefore, this document describes the ASG as part of the **AWS deployment architecture/case-study infrastructure**, not as functionality implemented inside the Python application.

---

# 16. Golden AMI and Launch Template

The case study specifies creating a golden AMI from a configured EC2 application instance.

The high-level process is:

```text
Base EC2
   |
Install required software
   |
Configure application
   |
Configure application environment
   |
Test
   |
Create AMI
   |
Create Launch Template
   |
Auto Scaling Group
```

The Launch Template becomes the standard definition used to create application instances.

The AMI should contain the software and configuration required to start a healthy application instance. Secrets should not be baked into the AMI.

---

# 17. EFS — Shared Storage

The case study includes Amazon EFS as the shared filesystem requirement.

It specifies:

```text
Throughput mode: Bursting
Mount point: /var/www/html/uploads
```

The intended test is:

1. Mount EFS on application instance A.
2. Create a test file.
3. Mount the same EFS filesystem on application instance B.
4. Confirm that the file is visible from instance B.

This demonstrates shared filesystem access across EC2 instances.

### Application-specific note

The ShopEase application has an S3 storage implementation and is configured for S3-backed image storage in the RDS + S3 configuration. Therefore, EFS is an infrastructure/case-study component rather than a requirement of the S3 storage path in the application code.

---

# 18. EBS Volume Management

The case study requires EBS practice on the bastion host.

The specified exercise is:

1. Attach a `10 GB gp3` EBS volume.
2. Create a partition.
3. Format it as `ext4`.
4. Mount it permanently at:

```text
/data
```

5. Add test files.
6. Create a manual snapshot.
7. Configure an automated snapshot lifecycle.
8. Take daily snapshots at midnight.
9. Retain snapshots for 7 days.

This provides practice with persistent block storage and snapshot-based recovery.

---

# 19. CloudWatch Monitoring

The case study requires monitoring for:

- EC2 CPU utilization.
- ALB unhealthy host count.
- RDS free storage.
- Network activity.
- Request count.
- Application/user activity through a custom metric.
- VPC Flow Logs.

The required alarms are:

```text
EC2 CPU > 70%
ALB unhealthy host count > 0
RDS free storage < 2 GB
```

A CloudWatch dashboard named:

```text
ShopEase-Production
```

is specified to show CPU, network I/O, and request-count information.

---

# 20. SNS Alerts

Create an SNS topic:

```text
shopease-alerts
```

Subscribe the required email address to the topic.

CloudWatch alarms can publish notifications to this SNS topic.

Logical flow:

```text
AWS Resource
     |
     v
CloudWatch Alarm
     |
     v
SNS Topic
     |
     v
Email / Notification
```

---

# 21. VPC Flow Logs

Enable VPC Flow Logs and send them to a CloudWatch Log Group.

This provides network-level visibility for the VPC and supports troubleshooting and security analysis.

---

# 22. Logging and Retention

The case study requires server access logs to be retained for at least 90 days.

The S3 log bucket is specified as:

```text
shopease-logs-[yourname]
```

with a lifecycle policy:

```text
Day 0      -> S3 Standard
Day 30     -> S3 Standard-IA
Day 90     -> Delete
```

This separates log retention from the EC2 application's local filesystem.

---

# 23. S3 Buckets from the Case Study

The case study defines three logical buckets.

### Static assets

```text
shopease-static-assets-[yourname]
```

Purpose:

- Product images.
- Frontend assets.
- Other static objects.

Required features:

- Versioning.
- Static website hosting as specified by the case study.

### Logs

```text
shopease-logs-[yourname]
```

Purpose:

- Access logs.

Required lifecycle:

```text
30 days -> Standard-IA
90 days -> Delete
```

### Database backups

```text
shopease-db-backups-[yourname]
```

Purpose:

- Database exports.

Required feature:

- Versioning.

The actual bucket names should be changed to the unique names used in the AWS account.

---

# 24. Application Storage Configuration

The application supports two storage modes.

## Local storage

```env
STORAGE_TYPE=local
```

Uploaded files are stored under the application's local `uploads` directory.

## S3 storage

```env
STORAGE_TYPE=s3
S3_BUCKET=<bucket-name>
AWS_REGION=ap-south-1
```

The S3 storage implementation uses `boto3`.

The storage abstraction is implemented through:

```text
app/services/storage_service.py
app/services/storage.py
```

This allows application routes to use the same storage interface without directly depending on a specific storage backend.

---

# 25. Deployment Validation Checklist

After deployment, validate the application in layers.

## Network

- [ ] VPC CIDR is `10.0.0.0/16`.
- [ ] At least two Availability Zones are configured.
- [ ] Public subnets exist in the required AZs.
- [ ] Private application subnets exist.
- [ ] Private DB subnets exist.
- [ ] Internet Gateway is attached.
- [ ] NAT Gateway is configured for private application outbound access.
- [ ] Public/private route tables are correctly associated.
- [ ] NACLs are configured according to the security design.

## IAM

- [ ] Individual IAM users are configured.
- [ ] Team-based IAM groups are configured.
- [ ] MFA is enabled for administrators.
- [ ] EC2 instance role is configured.
- [ ] S3 and CloudWatch permissions follow least privilege.

## Database

- [ ] RDS MySQL 8.0 is running.
- [ ] RDS is in the DB subnet group.
- [ ] RDS is not publicly accessible.
- [ ] Port 3306 is allowed only from the application tier.
- [ ] Automated backups are enabled.
- [ ] The application can connect to RDS.

## Application

- [ ] Python dependencies are installed.
- [ ] `.env` is configured.
- [ ] `SECRET_KEY` is set.
- [ ] RDS connection variables are correct.
- [ ] S3 variables are correct.
- [ ] Application starts successfully.
- [ ] Customer login works.
- [ ] Product browsing works.
- [ ] Cart and checkout work.
- [ ] Admin login works.
- [ ] Images load correctly.

## Load Balancing and Scaling

- [ ] ALB is reachable.
- [ ] Target group contains healthy targets.
- [ ] Health checks pass.
- [ ] Auto Scaling Group has the expected capacity.
- [ ] Scale-out policy is configured.
- [ ] Scale-in policy is configured.

## Monitoring

- [ ] SNS topic exists.
- [ ] Email subscription is confirmed.
- [ ] EC2 CPU alarm exists.
- [ ] ALB unhealthy-host alarm exists.
- [ ] RDS storage alarm exists.
- [ ] CloudWatch dashboard exists.
- [ ] VPC Flow Logs are enabled.

---

# 26. Performance Testing

ShopEase is intended to be tested under increasing traffic.

A load-testing tool such as **k6** can be used to generate realistic traffic against the deployed application.

The testing process should measure:

- Response time.
- Request throughput.
- Error rate.
- EC2 CPU utilization.
- Database behavior.
- Connection-pool utilization.
- ALB behavior.
- Auto Scaling response.
- Application bottlenecks.

The important lesson from this type of testing is that scaling EC2 instances alone does not guarantee better performance. Application-level and database-level bottlenecks must also be considered.

---

# 27. Security Considerations

The deployment should follow these principles:

### Never commit secrets

Do not commit:

```text
.env
AWS access keys
database passwords
API keys
private keys
```

### Use IAM roles

For EC2 access to S3 and CloudWatch, prefer an IAM instance role rather than storing long-lived AWS access keys inside the application.

### Keep RDS private

RDS should not have direct internet access.

### Restrict security groups

Only required ports and trusted sources should be allowed.

### Use individual IAM identities

Do not share AWS root credentials between team members.

### Use MFA

Enable MFA for administrator IAM users.

### Protect application secrets

Use secure environment configuration or a dedicated secret-management service for production deployments.

---

# 28. Deployment Flow Summary

The overall deployment process can be summarized as:

```text
1. Create VPC
        |
2. Create public/private subnets across AZs
        |
3. Configure IGW, NAT and route tables
        |
4. Configure IAM and security groups
        |
5. Create RDS MySQL
        |
6. Create S3 storage
        |
7. Prepare EC2 application instance
        |
8. Configure Flask + database + S3
        |
9. Validate application
        |
10. Create AMI
        |
11. Create Launch Template
        |
12. Create Target Group
        |
13. Create ALB
        |
14. Create Auto Scaling Group
        |
15. Configure EFS/EBS where required
        |
16. Configure CloudWatch
        |
17. Configure SNS alerts
        |
18. Enable VPC Flow Logs
        |
19. Perform load testing
        |
20. Analyze performance and bottlenecks
```

---

# 29. Lessons Learned

The ShopEase deployment exercise demonstrates that a cloud deployment is more than simply moving an application to an EC2 instance.

The application, database, storage, networking, security, load balancing, scaling, monitoring, and backup layers all have to work together.

The architecture also demonstrates why separating application and database tiers is important. Keeping RDS in private subnets reduces direct exposure, while an ALB provides a controlled entry point for users.

The S3 storage abstraction in the application is also important for scalable deployment because application instances do not need to depend on a single local filesystem for product images when the S3 backend is selected.

Finally, performance testing is necessary even after a scalable architecture has been created. Load testing helps identify whether the limiting factor is compute capacity, application processing, database connections, storage access, or another part of the system.

---

# 30. Current Implementation vs. Case-Study Target

| Area | Application / Repository Support | Case-Study Requirement |
|---|---|---|
| Flask application | Implemented | Workload |
| MySQL / SQLAlchemy | Implemented | RDS MySQL |
| RDS configuration | Environment-based | MySQL 8.0 Multi-AZ |
| S3 storage | Implemented | Static/object storage |
| Local storage | Implemented | Development/fallback |
| VPC | Infrastructure | `10.0.0.0/16`, multi-AZ |
| Public/private subnets | Infrastructure | Required |
| NAT Gateway | Infrastructure | Required |
| ALB | Infrastructure | Required |
| Auto Scaling | Infrastructure | 1–4 instances |
| Bastion host | Infrastructure | Required |
| EFS | Infrastructure | Required case-study exercise |
| EBS | Infrastructure | Required case-study exercise |
| CloudWatch | Infrastructure | Required |
| SNS | Infrastructure | Required |
| VPC Flow Logs | Infrastructure | Required |
| k6 load testing | External testing workflow | Recommended/used for performance testing |

This table is intentionally included so that the repository documentation distinguishes **application capabilities** from **AWS resources configured as part of the deployment exercise**.

---

# 31. Important Deployment Note

The repository is designed to support AWS deployment, but the Flask application should not be treated as the component that creates the AWS infrastructure.

AWS resources such as VPCs, subnets, ALB, Auto Scaling Groups, RDS, EFS, EBS, CloudWatch, SNS and IAM are configured separately in AWS.

The application consumes the infrastructure through configuration such as:

```env
DB_HOST=
DB_PORT=
DB_NAME=
DB_USER=
DB_PASSWORD=
STORAGE_TYPE=s3
S3_BUCKET=
AWS_REGION=
```

This separation keeps the application portable and allows the same ShopEase workload to be deployed using different infrastructure approaches.
