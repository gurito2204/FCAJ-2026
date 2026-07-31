---
title: "Amazon RDS PostgreSQL Multi-AZ Provisioning"
date: 2026-06-15
weight: 3
chapter: false
pre: " <b> 5.3. </b> "
---

We will provision a relational database using **Amazon RDS PostgreSQL** running in **Multi-AZ** (Multiple Availability Zones) mode. The database will reside inside isolated network segments (**Private DB Subnets**) and only permit private SQL traffic originating from backend EC2 application instances.

---

### 1. Creating DB Subnet Group for RDS

Before creating an RDS database instance, AWS requires you to define a **DB Subnet Group** mapping at least two subnets in different Availability Zones to support the Multi-AZ failover mechanism.

1. Navigate to **AWS Console** -> Search for and choose the **RDS** service.
2. In the left navigation pane, choose **Subnet groups** -> Click **Create DB subnet group**.
3. Configure the settings:
   * **Name:** `shopsflow-db-subnet-group`
   * **Description:** `DB Subnets Group for Shopsflow Multi-AZ PostgreSQL`
   * **VPC:** Select the custom `shopsflow-vpc`.
4. **Add subnets:**
   * **Availability Zones:** Select `ap-southeast-1a` and `ap-southeast-1b`.
   * **Subnets:** Check the boxes for the two Private DB Subnets created earlier (`10.0.5.0/24` and `10.0.6.0/24` corresponding to `shopsflow-private-db-1` and `shopsflow-private-db-2`).
5. Click **Create**.

---

### 2. Provisioning RDS PostgreSQL in Multi-AZ Mode

1. In the RDS dashboard, navigate to **Databases** -> Click **Create database**.
2. Configure DB options:
   * **Database creation method:** Select **Standard create** (Allows enabling Multi-AZ deployment).
   * **Engine options:** Select **PostgreSQL**.
   * **Engine version:** Select the recommended version (e.g., *PostgreSQL 16.x*).
3. **Templates:** Select **Production** or **Dev/Test** (Do not select Free Tier, as Free Tier does not support Multi-AZ deployment).
4. **Availability and durability:**
   * Select **Multi-AZ DB instance** (This configuration automatically deploys a primary database instance in AZ1 and maintains a synchronized standby instance in AZ2 to execute automated failovers during failures).
5. **Settings:**
   * **DB instance identifier:** `shopsflow-db`
   * **Master username:** `postgres`
   * **Master password:** Input your password (e.g., `ShopsflowPass123!`). *Note: This password must match the value stored as `SPRING_DATASOURCE_PASSWORD` in Secrets Manager.*
6. **Instance configuration:**
   * Under Burstable classes, select a resource-efficient type (e.g., `db.t3.micro` or `db.t4g.micro` to minimize lab costs).
7. **Storage:**
   * Set the Allocated storage minimum to `20 GiB`. Disable *Enable storage autoscaling* to prevent unexpected scaling costs.
8. **Connectivity:**
   * **Virtual private cloud (VPC):** Select `shopsflow-vpc`.
   * **DB subnet group:** Select the `shopsflow-db-subnet-group` created in Step 1.
   * **Public access:** Select **No** (Secures the database by preventing AWS from assigning it a public IP address).
   * **Existing VPC security groups:** Choose the `shopsflow-rds-sg` created in Step 3. Remove the default security group.
9. **Database authentication:** Select **Password authentication**.
10. **Additional configuration:**
    * Expand the panel -> Enter **Initial database name:** `shopsflow` (Directs RDS to automatically create an empty `shopsflow` database on initialization).
11. Click **Create database**.

---

### 3. Retrieving Connection Host Endpoint

1. Provisioning a Multi-AZ database takes **10 to 15 minutes** as AWS sets up instances in two physical Availability Zones and establishes synchronous data replication.
2. Once the database status changes to a green **Available**, click on the instance identifier `shopsflow-db`.
3. In the **Connectivity & security** tab, copy the **Endpoint** address (e.g., `shopsflow-db.xxxx.ap-southeast-1.rds.amazonaws.com`).
4. This endpoint represents your database connection hostname. We will store this value in Secrets Manager or use it to configure the application backend connection string.

> [!IMPORTANT]
> The database is now network-isolated. Because public access is disabled and the ingress rule of `shopsflow-rds-sg` only permits port 5432 traffic from instances belonging to `shopsflow-ec2-sg`, SQL connections are only authorized from EC2 instances residing in the Private App Subnets. Outside connections will be rejected.
