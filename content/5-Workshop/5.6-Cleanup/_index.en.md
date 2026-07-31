---
title: "Resource Cleanup"
date: 2026-06-15
weight: 6
chapter: false
pre: " <b> 5.6. </b> "
---

The High Availability Enterprise Architecture (Enterprise HA) uses services that incur hourly maintenance fees (such as NAT Gateway, Application Load Balancer, Fargate tasks, and RDS). Be sure to clean up the resources in the exact order below to avoid unexpected charges on your AWS account.

---

### Step-by-Step Cleanup Sequence

#### 1. Delete CloudFront & WAF
1. **CloudFront** → select the distribution → **Disable**, and wait for the status to change.
2. Once disabled, check the box → **Delete**.
3. **AWS WAF → Web ACLs** → select the created Web ACL → **Delete**.

#### 2. Delete ECS Service, Tasks, and Cluster
1. **ECS → Clusters** → select `shopsflow-cluster` → **Services**.
2. Select `shopsflow-backend-service` → set **Desired tasks = 0**, or **Delete service**.
3. Wait for all Fargate tasks to transition to **STOPPED**.
4. Once the cluster has no remaining services/tasks, delete `shopsflow-cluster`.

#### 3. Delete Application Load Balancer
1. **EC2 → Load Balancers** → select `shopsflow-backend-alb` → **Actions → Delete**.
2. **Target Groups** → select `shopsflow-backend-tg` → **Actions → Delete**.

#### 4. Delete ECR
1. **ECR → Repositories** → select `shopsflow-repo`.
2. For a complete cleanup: delete all images in the repo first, then delete the repository. If you want to keep the images as a record, you can skip this step.

#### 5. Delete Multi-AZ RDS PostgreSQL Database
1. **RDS → Databases** → select `shopsflow-db` → **Actions → Delete**.
2. Consider checking **Create final snapshot?** before permanent deletion if you want to retain the data.
3. Confirm deletion and wait for it to complete.
4. **Subnet groups** → select `shopsflow-db-subnet-group` → **Delete**.

#### 6. Delete S3 Buckets
* **Empty** and then **Delete** the frontend bucket `shopsflow-frontend-static-999` if you do not need to keep it.

#### 7. Delete Secrets Manager & KMS Key
1. **Secrets Manager** → select `shopsflow/production/secrets` → **Actions → Delete secret** (minimum wait time of 7 days).
2. **KMS** → select `shopsflow-kms-key` → **Key actions → Schedule key deletion** (minimum 7 days).

#### 8. Delete IAM Roles
* Delete `shopsflow-ecs-task-execution-role` and `shopsflow-ecs-task-role` — only delete them after ensuring ECS no longer references these two roles.

#### 9. Release Network Resources (NAT Gateway & VPC)
1. **NAT Gateways:** delete `shopsflow-nat-gw-1` and `shopsflow-nat-gw-2`, and wait for them to transition to **Deleted**.
2. **Elastic IPs:** release the 2 IP addresses allocated for the NAT Gateways.
3. **VPC:** after the ALB, RDS, ECS network interfaces, and NAT Gateways are entirely removed, delete custom route tables, detach & delete `shopsflow-igw`, delete custom subnets + security groups, and finally delete `shopsflow-vpc`.

---

### Future Development Directions

* **Infrastructure as Code (IaC):** use Terraform or AWS CDK to define the entire infrastructure as source code, preventing manual operational errors on the Console.
* **Automated CI/CD:** use GitHub Actions to automatically build & push images to ECR, register a new task-definition revision, and update the ECS service whenever changes are merged into `main`, replacing manual deployment.