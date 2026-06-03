# High Availability Two-Tier Web Application Architecture on AWS

[![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)](https://www.linux.org/)
[![Nginx](https://img.shields.io/badge/Nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org/)

---

##  Project Overview

### The Business Problem
Modern web applications demand **100% uptime**, **robust security**, and the ability to handle **unpredictable traffic spikes**. Deploying a web application on a single standalone server creates a **Single Point of Failure (SPOF)** and exposes critical application components and backend code directly to the public internet, inviting security vulnerabilities.

### The Solution: Two-Tier Architecture
This project implements a **production-grade, highly available, and secure Two-Tier Architecture** on AWS. By decoupling the **presentation/routing tier (Application Load Balancer)** from the **application computing tier (EC2 instances)**, the infrastructure isolates execution environments while maximizing resource utilization.

### Engineering Design Goals
- **🔒 Security:** Zero direct public internet exposure for application servers. All compute resources reside in isolated private subnets.
- **✅ High Availability:** Multi-AZ deployment ensuring the application can survive the complete outage of an AWS Availability Zone.
- **📈 Elastic Scalability:** Automated scaling policies that dynamically provision or terminate instances based on real-time traffic demand.

---

##  Architecture Diagram

<img width="1062" height="762" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/ccc81fba-b2d5-4f37-949a-d44990820547" />


### Isolated Network Edge
- A **custom VPC** forms our isolated network boundary.

### Public Tier
- Spans **two Availability Zones** containing public subnets housing:
  - **Application Load Balancer (ALB)**
  - **Bastion Host** for secure infrastructure management.

### Private Compute Tier
- Spans **two Availability Zones** containing private subnets where **auto-scaling application EC2 instances** reside.

### Egress Security
- Instances in private subnets communicate out to the internet (e.g., for OS updates or package installations) via an explicit **one-way stateful path through a NAT Gateway**.

---

##  AWS Services Used & Engineering Purpose
| AWS Service | Category | Engineering Purpose |
| --- | --- | --- |
| VPC | Networking | Defines an isolated, private virtual network wrapper for all cloud infrastructure. |
| Public Subnets | Networking | Houses public-facing resources (ALB, Bastion) that require direct internet accessibility. |
| Private Subnets | Networking | Isolates application EC2 instances entirely from inbound internet traffic. |
| Internet Gateway (IGW) | Networking | Allows bidirectional communication between public subnet resources and the public internet. |
| NAT Gateway | Networking | Grants private instances highly secure, outbound-only internet access for dependencies/updates. |
| Route Tables | Networking | Controls network traffic routing paths explicitly separating public and private boundaries. |
| Security Groups | Security | Stateful, instance-level virtual firewalls implementing strict access control rules. |
| Bastion Host | Security | A hardened, minimal-footprint jump box utilized for secure SSH bridging to private instances. |
| EC2 Instances | Compute | Host the core web application logic behind an Nginx/application engine server. |
| Launch Template | Compute | Defines configuration baselines (AMI, Instance Type, Key Pairs, User Data scripts) for scaling. |
| Auto Scaling Group (ASG) | Compute | Enforces high availability by automatically adjusting fleet capacity based on health and load. |
| Application Load Balancer | Networking | Distributes incoming HTTP/HTTPS traffic evenly across target instances in multiple AZs. |
| Target Group | Networking | Groups healthy EC2 application instances logically to receive routed traffic from the ALB. |

---

## Architecture Flow
### Inbound Traffic Flow (Client Request)
```plaintext
[User Request] 
      │
      ▼
[Internet Gateway] ──(Port 80/443)──► [Application Load Balancer]
                                                     │
                                             (Health Checked)
                                                     │
                                                     ▼
                                            [Target Group]
                                                     │
                                             (Port 8000/80)
                                                     │
                                                     ▼
                                      [Private EC2 Web Instances]


```
### Outbound Traffic Flow (Dependency Updates)
[Private EC2 Instance] ──► [NAT Gateway (Public Subnet)] ──► [Internet Gateway] ──► [Ubuntu/Package Mirrors]

##  Network Design
```plain text
VPC CIDR block: 10.0.0.0/16 (65,536 available private IP addresses)
├── Availability Zone: ap-south-1a
│   ├── Public Subnet 1:  10.0.1.0/24  (251 usable IPs)
│   └── Private Subnet 1: 10.0.10.0/24 (251 usable IPs)
└── Availability Zone: ap-south-1b
    ├── Public Subnet 2:  10.0.2.0/24  (251 usable IPs)
    └── Private Subnet 2: 10.0.20.0/24 (251 usable IPs)
```
### Network Segmentation Strategy
- Subnet distribution across two distinct Availability Zones (AZs) ensures physical power/data center fault tolerance.
- Standard /24 subnets provide ample allocation space per zone while isolating public entry points completely from backend data lines.

## Security Implementation
### Principle of Least Privilege (PoLP)
Security Groups are treated as strict network perimeters, opening only the exact protocols and ports required for proper function:

- Public Application Load Balancer Security Group: Allows inbound HTTP (80) and HTTPS (443) from anywhere (0.0.0.0/0).
- Private Compute Security Group: Strictly limits incoming web traffic. It only accepts inbound traffic on application ports if the source explicitly originates from the ALB Security Group. It does not accept direct public internet requests.

Controlled SSH Management (Bastion-to-Private):
 - The Bastion Host is accessible via SSH only from the administrator's specific, whitelisted public IP address.
 - Private EC2 instances accept SSH traffic only if the source is the Bastion Host's internal private security group ID.

## 🚀 Step-by-Step Deployment Process
Step 1 – Create VPC & Core Networking: Provisioned custom VPC 10.0.0.0/16. Attached an Internet Gateway to establish outside connectivity.

Step 2 – Configure Subnets and Route Tables: Created 2 Public and 2 Private subnets across target AZs. Configured an explicit Public Route Table pointing 0.0.0.0/0 to the IGW. Bound public subnets to it.

Step 3 – Deploy NAT Gateway: Allocated an Elastic IP (EIP) and spun up a NAT Gateway inside Public Subnet 1. Updated Private Route Tables to direct all outbound internet traffic (0.0.0.0/0) through the NAT Gateway.

Step 4 – Establish Security Groups & Bastion Host: Configured cascading firewalls for ALB, Bastion, and Application nodes. Deployed a hardened Ubuntu micro-instance into Public Subnet 1 to serve as the secure Administrative Bastion proxy.

Step 5 – Build Target Group & Application Load Balancer: Provisioned an HTTP target group utilizing path-based health checks (/index.html or /demo/). Configured an internet-facing Application Load Balancer listening on Port 80 across both public subnets.

Step 6 – Configure Launch Template & Auto Scaling Group: Created an EC2 Launch Template with Nginx automation embedded inside the User Data initialization script. Established an Auto Scaling Group bound to the private subnets with a Min: 2, Desired: 2, Max: 4 instance capacity rule.

Step 7 – Application Verification & Health Checking: Validated targets registering as Healthy inside the Target Group status dashboard. Accessed the application successfully using only the public DNS string provided by the ALB.

## 🛠️ Challenges & Troubleshooting
### 🛑 Challenge 1: SSH Private Key Permission Error
The Issue: While attempting to SSH from the Bastion Host to a private EC2 instance, SSH rejected the key with the following terminal output:
```bash
Permissions 0664 for 'bastionhost.pem' are too open.
This private key will be ignored.
Load key "bastionhost.pem": bad permissions
```
**Root Cause Analysis**: The private key file held loose read/write access permissions (0664), making it readable by other user groups on the host system. OpenSSH enforces strict security standards and automatically rejects keys with insecure permissions to prevent unauthorized access.

**Solution**: Restricted file permissions exclusively to the file owner using octal notation:
```bash
chmod 400 bastionhost.pem
```
Key Takeaway: Mastery of Linux POSIX file permission models is a fundamental requirement for maintaining secure cryptographic authentication channels.

### 🛑 Challenge 2: SSH Authentication Failure (Permission denied)
The Issue: Even with correct file permissions, attempting a cross-node connection yielded a generic authentication block:
```plaintext
Permission denied (publickey)
```
**Systematic Investigation Process**:
Identity Verification: Confirmed the correct default SSH username for the target Linux AMI (ubuntu).
Firewall Auditing: Verified inbound Security Group rules explicitly whitelisted the Bastion Host's private security group ID.
Network Routing Validation: Confirmed routing tables and inner-VPC connectivity paths were fully active.
Metadata Inspection: Audited the specific EC2 instance configuration details via the AWS Console.

**Root Cause Identification**: Discovered a key pair mismatch; the active instance was provisioned with an older key pair configuration that differed from the active .pem key hosted on the Bastion session.

Solution: Used the correct Launch Template key pair when connecting to the private instance, ensuring perfect alignment across the fleet.

Key Takeaway: AWS binds the corresponding public key to an instance's metadata (authorized_keys) at boot. Cryptographic handshakes will instantly fail if the presenting private key does not match this immutable initialization point.

### 🛑 Challenge 3: Application Accessible Only on Port 8000
The Issue: The web application was successfully serving traffic only when explicitly defining the runtime port in the URL layout:
```plaintext
http://ALB-DNS:8000  --> Works (200 OK)
http://ALB-DNS
```
**Root Cause:** The Application Load Balancer listener was configured on port 8000 instead of using standard HTTP port 80.
**Solution:** Configured ALB Listener to Port 80, mapped it to a Target Group looking at Port 8000, and verified Security Groups allowed ALB-to-EC2 communication on port 8000.
**Visual Traffic Flow:** `User` ➔ `ALB Listener (80)` ➔ `Target Group (8000)` ➔ `Private EC2 Instance (8000)`.
**Learning:** Load Balancers should expose standard web ports while forwarding requests to application-specific ports internally.

### 🛑Challenge 4: Multi-AZ High Availability Implementation
  **Objective:** Eliminate the single point of failure by deploying application instances across multiple Availability Zones.
**Solution:** Created a second private subnet in another Availability Zone, deployed an additional EC2 instance, registered both instances with the Target Group, verified healthy status through ALB health checks, and enabled traffic distribution through the ALB.
**Validation:** Performed failover testing by stopping one instance and confirming uninterrupted application availability through the remaining healthy instance.
**Learning:** High Availability is achieved through redundancy, health monitoring, and automatic traffic routing.
