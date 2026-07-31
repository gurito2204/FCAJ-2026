---
title: "Blog 3"
date: 2026-06-30
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

# AWS UNDER-THE-HOOD MECHANICS: FROM HIDDEN CHARGES TO INVISIBLE NETWORK FAILURES

When working with AWS long enough, you start to realize there is a massive gap between what is taught in certification courses and what actually happens in production environments. Some incidents do not trigger clear alarms and are not listed under common StackOverflow posts, yet they cause flaky system behavior or consume thousands of dollars in service charges.

Here are 4 advanced and overlooked technical gotchas on AWS that I spent sleepless nights debugging to extract valuable lessons.

---

### 1. Cross-AZ Data Transfer: The Network Billing Shock Within the Same Region

Most people understand that data transfer out to the internet costs money. However, many developers mistakenly believe: *"As long as data transfers internally within the same Region, it is completely free"*.

*   **The Hidden Truth:** AWS charges for data transferred between different Availability Zones (AZs) within the same Region. The typical cost is $0.01 per GB sent and $0.01 per GB received (totaling $0.02/GB for a full network trip).
*   **The Scenario:** You deploy an EKS cluster or EC2 instances running microservices spread across both `AZ-a` and `AZ-b` to ensure High Availability (HA). These services continuously call each other's APIs or query a shared Redis Cache or Database located in `AZ-a`.
*   **The Consequence:** Thousands of cross-AZ inter-service calls per second silently generate terabytes of traffic between AZs. Consequently, your end-of-month "Data Transfer" bill might end up costing more than the compute resources themselves.
*   **The Fix:** Adopt an **AZ-Awareness** mindset. Use routing mechanisms such as *Topology Aware Hints* in Kubernetes to prioritize routing traffic between services residing within the same AZ. Allow cross-AZ traffic only as a failover fallback.

---

### 2. The MTU 9001 Trap and Flaky Connections Over VPC Peering / VPN

This is a classic debugging scenario that leaves junior DevOps engineers stuck for days because the system fails to spit out any explicit error logs.

*   **The Core Issue:** By default, EC2 instances within the same VPC communicate using a Maximum Transmission Unit (MTU) of 9001 bytes (Jumbo Frames). However, when traffic passes through inter-region VPC Peering, VPN tunnels, or Direct Connect links, the MTU limit is shrunk to the standard 1500 bytes.
*   **The Symptom:** Ping commands between 2 EC2 instances in different VPCs respond smoothly. SSH connections work fine. Yet, whenever the application attempts to transfer a large file or send a long JSON API payload over that connection, the request hangs indefinitely until it timeouts.
*   **The Underlying Cause:** Packets exceeding 1500 bytes must be fragmented, but intermediate routers cannot fragment them and must send back an ICMP notification. If sysadmins block all ICMP traffic in Security Groups, this notification packet is swallowed, creating a **Path MTU Discovery Black Hole** issue.
*   **The Fix:** Always allow ICMP (specifically *Custom ICMP - IPv4: Type 3, Code 4 - Destination Unreachable*) in Security Groups, or reduce the MTU on the EC2 network interface to 1500 if your system relies heavily on inter-VPC/VPN connections.

---

### 3. The Implicit Limit Ceiling of DynamoDB On-Demand During Flash Sales

DynamoDB On-Demand is marketed as a hands-off auto-scaling solution that eliminates the need to plan Read/Write Capacity Units (RCUs/WCUs). However, this "automatic" capability comes with a crucial constraint.

*   **The Under-the-Hood Behavior:** DynamoDB On-Demand does not scale up instantly to infinity. It can only handle up to double the historical peak traffic rate achieved by the table in the past.
*   **The Disaster Scenario:** Your system runs smoothly at 1,000 requests/second. At exactly 12:00 AM, a Flash Sale starts, and traffic surges to 20,000 requests/second within a few seconds.
*   **The Consequence:** DynamoDB immediately returns `ProvisionedThroughputExceededException` errors, dropping client requests. DynamoDB's internal partition-splitting mechanism takes several minutes to adjust to the new workload, but by then, your customers have already left.
*   **The Fix:** If you anticipate sudden traffic spikes within a short timeframe, manually switch the table capacity mode to **Provisioned Capacity** a few hours in advance, configure appropriate RCU/WCU targets to let DynamoDB provision physical partition infrastructure, and switch back to On-Demand afterward.

---

### 4. The Expensive Click on CloudWatch Logs Insights

CloudWatch Logs Insights is a useful tool that allows you to query log groups using SQL-like queries. However, AWS's billing model for this tool can catch you off guard.

*   **The Pricing Model:** Logs Insights does not charge based on the number of log lines returned. Instead, it bills you per GB of raw log data scanned (approximately $0.005 per GB scanned).
*   **The Common Mistake:** You open the tool, select a Log Group that has accumulated logs for the past 6 months (about 500GB), write a generic query without defining a specific time range filter, and hit Run.
*   **The Consequence:** With a single click and a few seconds of execution time, you have just spent $2.5. If a developer sets up a cronjob to trigger that query every 5 minutes for a custom dashboard, you will receive a bill of thousands of dollars just for scanning log files.
*   **The Fix:** Always limit the query time range to the smallest window necessary (e.g., 15 minutes or 1 hour). For analyzing large historical log archives, export the logs to S3 and query them using **AWS Athena** at a much lower cost.

---

### Conclusion

Working on AWS is more than just assembling services on an architecture diagram; it is about understanding the operating parameters of the underlying infrastructure. Details like MTU packet sizes, partition-splitting behaviors, or cross-AZ data fees are often ignored because they are absent from basic labs.

I hope sharing these lessons helps you avoid invisible network failures and optimize your AWS workloads. If you have run into other gotchas on AWS, leave a comment below.

---

### Post Information
*   **Facebook Group:** AWS Study Group
*   **Original Post Link:** [Facebook Post Link](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2229330564498570/)