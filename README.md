# High Availability Two-Tier Web Application Architecture on AWS

[![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)](https://www.linux.org/)
[![Nginx](https://img.shields.io/badge/Nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)

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

## Architecture Flow

