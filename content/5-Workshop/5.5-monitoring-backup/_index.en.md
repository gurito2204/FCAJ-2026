---
title: "Monitoring, Backup & Testing"
date: 2026-06-15
weight: 5
chapter: false
pre: " <b> 5.5. </b> "
---

This article guides you through configuring centralized monitoring via Amazon CloudWatch for ECS Fargate, the RDS backup mechanism, and practical testing scenarios.

### 1. System Monitoring (CloudWatch Logs & Alarm)

The backend running on ECS Fargate has the `awslogs` driver configured in section 5.4, so the container automatically sends logs to CloudWatch without needing to install an agent manually like on EC2.

**Log Group**
* Navigate to **CloudWatch** → **Log groups** → `/shopsflow/ecs/backend`. 
* Each Fargate task creates a separate log stream — when the ECS Service scales and adds more tasks, new log streams appear automatically.
* Monitor: task start/stop, Spring Boot startup errors, database connection errors, payment errors, 4xx/5xx responses, and new revision deployments.

**Configure CloudWatch Alarm for ECS Service**
* CloudWatch → Alarms → **Create alarm**.
* Select metric → ECS → By Service Name → select `CPUUtilization` of the `shopsflow-cluster` cluster, `shopsflow-backend-service` service.
* Condition: Static, Greater than 80%, 2 consecutive evaluation periods (5 minutes/period).
* Configure actions: create/select an SNS Topic linked to an email.
* Alarm name: `shopsflow-ecs-high-cpu-alarm` → **Create alarm**.

**Other metrics to monitor**
* **ALB:** healthy/unhealthy host count, request count, target response time, 4xx/5xx.
* **RDS:** CPU utilization, database connections, storage/memory.

---

### 2. Data Backup (RDS Automated Backup)

Since RDS has Multi-AZ enabled in section 5.3, AWS automatically performs daily automated backups during the configured backup window; no need to write manual backup scripts via EC2.

* Navigate to **RDS** → **Databases** → `shopsflow-db` → **Maintenance & backups**.
* Check the **Backup retention period** (default is 7 days, can be increased up to 35 days).
* To restore to a specific point in time: Actions → **Restore to point in time**.
* Before making major changes (e.g., changing a task definition that affects the schema), create an additional manual snapshot: Actions → **Take snapshot** to have a proactive recovery point outside the automated schedule.

---

### 3. System Testing Scenarios (Validation Scenarios)

**🧪 Scenario 1: Edge-to-Backend Routing Test**
* **Execution:** Access the CloudFront Distribution Domain Name (`https://dxxxxx.cloudfront.net`).
* **Expectation:** The React Frontend loads successfully from S3 via CloudFront. The `/api/*` requests are forwarded by CloudFront to the ALB, the ALB routes them to a healthy ECS Fargate task in the private subnet, the backend reads/writes data to RDS and returns the result.

**🧪 Scenario 2: Centralized Logging on CloudWatch**
* **Execution:** CloudWatch → Log groups → `/shopsflow/ecs/backend`.
* **Expectation:** Log streams corresponding to each Fargate task appear. As the service scales and adds tasks, new log streams are automatically created.

**🧪 Scenario 3: Concurrency Checkout Test**
* **Execution:** Run the load test command:
  ```bash
  ab -n 20 -c 10 -p payload.json -T application/json [https://dxxxxx.cloudfront.net/api/checkout](https://dxxxxx.cloudfront.net/api/checkout)
  ```
* **Expectation:** Only 1 request succeeds, the remaining requests receive a `409 Conflict`. The Spring Boot log records an `OptimisticLockingFailureException`. The inventory in RDS does not drop below zero (no negative inventory).

**🧪 Scenario 4: ECS Service Auto Scaling Test**
* **Execution:** Use an HTTP concurrent load testing tool (e.g., `ab` or `hey`) to hit a CPU-intensive API, or use `ecs exec` into a running container to generate artificial CPU load.
* **Expectation:** The service's `CPUUtilization` increases, the `shopsflow-ecs-high-cpu-alarm` transitions to `ALARM`, and Service Auto Scaling increases the running task count (e.g., from 2 to 3–4). When the load subsides, the service automatically scales in back to the initial number of tasks.

**🧪 Scenario 5: VNPay Payment Flow Test**
* **Execution:**
  1. Call `POST /api/payment/vnpay/checkout/{orderId}` (with JWT) → receive `payUrl`.
  2. Open the `payUrl` and pay using a VNPay Sandbox test card.
  3. VNPay redirects to the `VNPAY_RETURN_URL` (`/payment-result`) with query parameters containing the result.
  4. VNPay simultaneously calls the `VNPAY_IPN_URL` (server-to-server) for the backend to verify the signature and update the status.
  5. The frontend calls `GET /api/orders/{orderId}` to fetch the official status from the DB.
* **Expectation:** The order status changes from `PENDING` to `PAID` only after successful IPN verification — do not trust the client-side redirect. `GET /api/orders/{orderId}` returns the correct final status.