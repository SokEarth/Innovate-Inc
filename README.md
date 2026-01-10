# Architecture Design Document
## Innovate Inc. – AWS Cloud Infrastructure

**Author:** Somto Ezeh
**Date:** 10-01-2026
**Cloud Provider:** AWS

Target Audience: Engineering, Platform, Security

## 1. Introduction

Innovate Inc. is a small startup with limited cloud infrastructure experience and is seeking a managed solution that minimizes operational overhead while following industry best practices.

This document proposes a cloud-native architecture for Innovate Inc.’s web application, designed to be secure, reliable, scalable and cost-effective while supporting rapid growth and frequent deployments.

The proposed architecture leverages AWS-managed Kubernetes(Amazon EKS) to support a Python/Flask REST API backend, a React single-page application frontend and a PostgreSQL database.

The design prioritizes security due to the handling of sensitive user data, supports CI/CD for continuous delivery and allows the platform to scale from an initial low-traffic workload to potentially millions of users.

## 2. Scope and Objectives
### 2.1 Objectives

The primary objectives of this architecture are to:

- Provide a secure and production-ready AWS environment.
- Leverage managed Kubernetes to reduce operational complexity.
- Support horizontal scalability to accommodate rapid user growth.
- Enable CI/CD for frequent and reliable deployments.
- Protect sensitive user data through strong security controls.
- Remain cost-effective at low initial traffic, with the ability to scale as needed.

### 2.2 Out of Scope

The following areas are explicitly out of scope for this document:

- Application-level business logic.
- Frontend or backend code design.
- Advanced analytics, machine learning, or data warehousing.
- Multi-cloud or on-premises deployments.

## 3. Assumptions

- The application components are containerized using Docker.
- Innovate Inc. prefers managed AWS services over self-managed solutions.
- Standard security requirements apply (encryption at rest and in transit).
- Infrastructure will be defined using Infrastructure as Code (IaC).
- CI/CD pipelines will be integrated with the Kubernetes deployment process.

## 4. High-Level Architecture Overview

At a high level, the proposed architecture consists of:

* Multiple AWS accounts to provide isolation and governance.
* A dedicated Virtual Private Cloud (VPC) with public and private subnets.
* An Amazon EKS cluster hosting the Flask API and React frontend.
* Amazon RDS for PostgreSQL as the managed database service.
* AWS-native load balancing, IAM, and secrets management.
* CI/CD pipelines for automated build, test, and deployment.
* Centralized logging, monitoring, and alerting.

A detailed breakdown of each component is provided in the following sections.

---

## 5. Cloud Environment Structure

### 5.1 Context
When a company starts small, as is the case with Innovate Inc, everything might live in a single cloud account i.e the dev, staging and production environments all sharing the same AWS account. 
That’s fine early on, but as things grow, it leads to:
- Security risks: Devs accidentally accessing prod resources.
- Unclear cost visibility IAM sprawl: Too many permissions, this becomes difficult to manage.
- Environment interference: Testing code impacting production.
- Compliance issues: No proper data separation or isolation.

With this in mind we need to define a multi-account strategy for our cloud platform. A multi-account defines how an organization structures and governs its cloud accounts. This strategy fixes the above listed problems by organizing aws infrastructure into multiple, isolated accounts or projects based on function, environment or team ownership.

### 5.2 AWS Account Strategy

**Recommended Setup:**

* **Management Account**
* **Production Account**
* **Non-Production Account (Dev / Staging)**

### 5.3 Account Breakdown & Purpose 

1. **Management (Shared Services) Account**
   Purpose:
   Acts as the root account of the AWS Organization Centralized governance and shared tooling.
   
   Responsibilities:
   AWS Organizations and SCPs Centralized billing and cost management IAM identity federation (e.g. SSO) Centralized logging (CloudTrail, Config) Shared CI/CD tooling (optional at early stage).
   
   Justification: Prevents day-to-day workloads from running in the root account Enables centralized security controls Simplifies access management across environments

2. **Non-Production Account (Dev / Staging)**
   Account purpose:
   Hosts development and staging environments

   Target Workloads:
   EKS cluster for development/testing RDS PostgreSQL (smaller instance sizes) CI/CD test deployments Experimental or temporary infrastructure.

   Justification: 
   Isolates unstable or experimental workloads from production.
   Enables developers to iterate safely.
   Keeps costs low by using smaller resources.
   Reduces blast radius of misconfigurations.
   
3. **Production Account**
   Purpose:
   Hosts all customer-facing production workloads

   Workloads:
   Production EKS cluster.
   Production RDS PostgreSQL (Multi-AZ).
   Load balancers, networking, monitoring.
   Production secrets and encryption keys.

   Justification:
   Strong isolation for sensitive user data.
   Allows stricter IAM and security controls.
   Limits impact of security incidents or human error.
   Enables accurate cost tracking for production.


## 6. Network Design

### 6.1 VPC Architecture
The recommended architecture for Innovate Inc's VPCs consists of:
* One VPC per AWS account
* VPC spans **at least two Availability Zones**
* Subnets:

  * **Public subnets:** Load balancers, NAT gateways
  * **Private compute subnets:** EKS worker nodes, application pods
  * **Private compute subnets:** RDS

### 6.2 Network Security

These fundamentals would secure Innovate Inc's VPC design:
* Security Groups should be used as primary firewall mechanism.
  This is used to control who can talk to whom.
  Prevent lateral movement between services and enforce service-to-service boundaries.
* NACLs should be used at the subnet level to help with blast-radius reduction if security
  groups are misconfigured.
* Private compute and database subnets with no direct internet access.
  Traffic can only go in through the AWS Load Balancer in the public subnets. 
* Egress control should be done using NAT Gateway.

Network isolation should be implemented like this:

* App SG (Security Group) allows inbound only from ALB SG.
* Database SG allows inbound only from App SG.
* No public inbound access is allowed to private tiers.

## 7. Compute Platform – Kubernetes (Amazon EKS)

### 7.1 Kubernetes Cluster Design

The Kubernetes Cluster would consist of:
* One EKS cluster per environment.
* Managed control plane provided by AWS.
* Worker nodes deployed in private subnets.

### 7.2 Node Groups & Scaling

* Managed node groups.
* Separate node groups for:

  * Application workloads.
  * System workloads (optional at larger scale).
* Auto Scaling Groups with cluster autoscaler or karpenter
* Horizontal Pod Autoscaler (HPA) for application scaling with KEDA.

### 7.3 Resource Allocation

* CPU and memory requests/limits should be defined per pod.
* Namespace-level isolation should be applied between environments or teams.

## 8. Containerization & Deployment Strategy

* Docker should be used for containerizing backend and frontend.
* Images are built via CI pipeline.
* Images are stored in **Amazon ECR**.
* Deployments managed using Kubernetes manifests or Helm using the GitOps workflow.
* CI/CD pipeline triggers rolling deployments to EKS.

## 9. Database Design

### 9.1 PostgreSQL Service Selection

**Recommended Service:** Amazon RDS for PostgreSQL

**Justification:**

* RDS is an aws-managed service. This removes the hassle of handling the backups and patching of the database.
* High availability (Multi-AZ) is built in.
* Automated failover is an added advantage.
* Encryption at rest and in transit is readily available. 

### 9.2 Backup & Disaster Recovery Strategy

* Use automated daily backups.
* Point-in-time recovery should be enabled.
* Multi-AZ deployment should be used for high availability.
* The option to add read replicas as traffic grows is provided.
