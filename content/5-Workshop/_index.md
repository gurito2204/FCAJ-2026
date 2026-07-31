---
title: "Workshop"
date: 2026-06-15
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Deploying Shopsflow on AWS with Enterprise HA Architecture

#### Overview

In this hands-on workshop, we will perform the deployment of the **Shopsflow** e-commerce system (React Frontend, Spring Boot Backend API, and PostgreSQL database) using a secure, multi-tier **Enterprise Highly Available (HA)** architecture on **Amazon Web Services (AWS)**.

The core objectives of this workshop are:
* **High Availability (HA):** Design a custom VPC topology spanning multiple Availability Zones (Multi-AZ). The application backend EC2 instances are managed by an Auto Scaling Group (ASG) behind an Application Load Balancer (ALB). The relational database is deployed as a Multi-AZ RDS PostgreSQL instance.
* **Defense in Depth (Security):** Fully isolate backend EC2 instances and the RDS PostgreSQL database in Private Subnets. Direct public access is restricted; the React frontend is hosted on S3 and distributed globally through Amazon CloudFront CDN integrated with AWS WAF (Web Application Firewall). Credentials and keys are securely stored in AWS Secrets Manager and KMS.
* **Optimized Connections & Observability:** Establish a VPC Gateway Endpoint for S3 to process backups over the private AWS backbone. System logs, container logs, and instance metrics are aggregated into Amazon CloudWatch.

#### Lab Outline

This workshop is structured into the following step-by-step guides:

1. [Introduction & Architecture Diagram](5.1-workshop-overview/)
2. [Network Configuration (VPC Multi-AZ), Access Control & Secrets](5.2-prerequiste/)
3. [Provisioning Amazon RDS PostgreSQL Database (Multi-AZ)](5.3-rds/)
4. [Deploying Frontend (S3+CloudFront) & Backend (ALB+ASG)](5.4-ec2/)
5. [Configuring CloudWatch Agent & Automated Private Backups](5.5-monitoring-backup/)
6. [Resource Clean-up](5.6-cleanup/)