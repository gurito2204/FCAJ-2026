---
title: "Proposal"
date: 2026-07-30
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

## Shopsflow

### Proposal for an AWS container-based e-commerce application

#### 1. Executive summary

Shopsflow is a full-stack e-commerce application with a React/Vite frontend, a Spring Boot backend, PostgreSQL data storage, and an online payment flow. This proposal separates the frontend, application runtime, and database so that each layer has a clear responsibility, security boundary, and deployment path.

The frontend is delivered from Amazon S3 through Amazon CloudFront. The backend runs as a container on Amazon ECS Fargate behind an Application Load Balancer (ALB), while Amazon RDS for PostgreSQL stores application data privately. Amazon ECR stores versioned backend images; Amazon CloudWatch supports logs and operational checks.

#### 2. Problem statement and proposed solution

Running the complete application on one public server would couple static delivery, API runtime, and database access. It would also expose more infrastructure than necessary and make deployment, scaling, troubleshooting, and cost review harder.

The proposed solution uses a three-tier AWS architecture:

- **Frontend delivery:** CloudFront serves the React/Vite build from a private S3 bucket.
- **Application tier:** An internet-facing ALB routes API requests to Spring Boot containers on ECS Fargate in private subnets.
- **Data tier:** RDS PostgreSQL remains private and accepts port `5432` only from the ECS security group.

This approach keeps the public entry point limited to CloudFront and the ALB, while backend tasks and the database do not have public IP addresses.

#### 3. Solution architecture

![Architecture Diagram](/images/5-Workshop/5.1-Workshop-overview/diagram1.png)
_Figure 1. Logical architecture of Shopsflow. The deployed configuration uses Region `us-east-1`._

**Request flow**

1. A user opens the website through CloudFront.
2. CloudFront retrieves the React/Vite static artifacts from S3.
3. Requests to `/api/*` are forwarded by CloudFront to the ALB.
4. The ALB routes healthy requests to ECS Fargate tasks in private subnets.
5. The Spring Boot backend reads and writes data in RDS PostgreSQL.

**Payment flow**

The backend creates payment parameters, calls the payment provider through the NAT Gateway, and returns the payment URL to the user. After the provider responds, the backend validates the signature and payment status before updating the order.

#### 4. AWS services and project settings

| Area               | Service and configuration                                                                    |
| ------------------ | -------------------------------------------------------------------------------------------- |
| Region             | `us-east-1`                                                                                  |
| Network            | `shopsflow-vpc` with two public and two private subnets across `us-east-1a` and `us-east-1b` |
| Frontend           | Amazon S3 private origin and CloudFront distribution `fe cloudfront`                         |
| Public domain      | `d2m34udjfc5fxq.cloudfront.net`                                                              |
| API entry point    | Internet-facing ALB `shopsflow-alb`                                                          |
| Container registry | Amazon ECR repository `shopsflow-repo`                                                       |
| Compute            | ECS cluster `shopsflow-cluster`, Fargate, `awsvpc`, private subnets, public IP disabled      |
| Container port     | `8080`, with an IP target group                                                              |
| Database           | RDS PostgreSQL `database-shopsflow`, private DB subnet group, port `5432`                    |
| Observability      | CloudWatch log group `/shopsflow/ecs/backend` through the `awslogs` driver                   |

#### 5. Security and operations

Security groups enforce the traffic path `CloudFront → ALB → ECS → RDS`:

- `alb-sg` accepts HTTP `80` and HTTPS `443`, then sends backend traffic only to `ecs-sg` on port `8080`.
- `ecs-sg` accepts port `8080` only from `alb-sg`.
- `rds-sg` accepts PostgreSQL port `5432` only from `ecs-sg`.

ECS uses separate task execution and task roles. The execution role pulls images from ECR and delivers logs to CloudWatch; the task role contains only application permissions. Database credentials are supplied during deployment rather than stored in source code.

CloudWatch is used to review ECS task starts and stops, Spring Boot errors, database connectivity, payment failures, ALB target health, and deployment revisions.

#### 6. Implementation plan

1. Create the VPC, subnets, Internet Gateway, NAT Gateways, route tables, and security groups.
2. Create the ECS task execution role and task role with least-privilege access.
3. Create the RDS PostgreSQL instance and private DB subnet group.
4. Build the backend image, push it to ECR, and create the ECS Fargate service with ALB health checks.
5. Build the React/Vite frontend, upload the `dist` artifacts to S3, and configure CloudFront with the ALB `/api/*` behavior.
6. Configure CloudWatch logging, validate payment and API flows, then review cost and resource usage.

Each backend release produces a new ECR image and ECS task-definition revision. The ECS service performs a health-checked rollout rather than changing files inside a running container.

#### 7. Risks and mitigation

| Risk                     | Mitigation                                                                                                       |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| Unhealthy ECS tasks      | Use ALB health checks, ECS stopped reasons, and CloudWatch Logs to isolate startup, routing, or database errors. |
| Database exposure        | Keep RDS private and restrict port `5432` to `ecs-sg`.                                                           |
| Incorrect payment status | Validate the provider signature and status on the backend; do not trust client redirects alone.                  |
| Cost left after testing  | Review NAT Gateway, ALB, Fargate, RDS, CloudWatch, and data-transfer usage; clean up in dependency order.        |
| Failed redeployment      | Keep versioned ECR images and use a new task-definition revision for each release.                               |

#### 8. Expected outcomes

The proposal delivers a deployable e-commerce architecture with a CDN-backed frontend, private containerized backend, private PostgreSQL database, health-checked API routing, structured logging, and a controlled payment flow. It also provides a clear operational model for validating releases, troubleshooting failures, and cleaning up resources to avoid unnecessary cost.
