---
title: "Blog 2"
date: 2026-06-25
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

# HIDDEN TRAPS IN AWS THAT OFFICIAL DOCS RARELY WARN YOU ABOUT

When starting out with AWS, we often hear about common concepts like EC2, S3, RDS, or clean architecture designs shown on slides. However, only when we directly manage production systems and pay with real money or deal with early morning incidents do we realize there are hidden corners rarely mentioned in introductory courses.

Here are 4 practical lessons regarding the background mechanisms of AWS that I learned after paying several "tuition fees" for my lack of understanding.

---

### 1. NAT Gateway and the S3 Data Processing Cost Trap

Many documents recommend placing EC2 instances in a Private Subnet for security, then using a NAT Gateway for outbound internet access. This design is correct until your application begins reading and writing data continuously to S3 (such as logs, backups, or media uploads).

*   **The Hidden Trap:** By default, traffic from EC2 to S3 routes through the NAT Gateway. NAT Gateways do not just bill you for uptime; they also charge a data processing fee per GB of data passing through.
*   **The Consequence:** Your end-of-month bill surges simply because of NAT Gateway processing fees, even though both the EC2 instance and S3 bucket reside within the same AWS Region.
*   **The Fix:** Simply enable an **S3 Gateway VPC Endpoint**. This feature is completely free. Traffic will route over the AWS internal network backbone instead of escaping through the NAT Gateway, which accelerates transfers and cuts out 100% of these processing fees.

---

### 2. Transitioning Small Files to Glacier: When Cost Savings Turn into a Billing Disaster

S3 Glacier and Glacier Deep Archive are famous for offering long-term cold storage at extremely low prices. I once confidently configured an S3 Lifecycle Rule stating: *"Objects older than 30 days are automatically transitioned to Glacier Deep Archive"*. However, the system at that time stored millions of small log files (each only a few KB in size).

*   **Hidden API Requests Cost:** AWS charges transition requests fees (Lifecycle Transition Request) per file count ($0.03 - $0.05 per 1,000 requests). Transitioning 10 million small files means the transition request fees alone will cost many times more than the actual storage costs saved over a whole year.
*   **Metadata Overhead Fee:** For every object transitioned to Glacier, AWS automatically appends a 32KB metadata index block. A 2KB file on S3 Glacier will be billed for 34KB of storage.
*   **The Fix:** Before creating a transition Lifecycle Rule, zip/tar small files into larger archives, or configure lifecycle filters to restrict transitions to objects with a minimum size of 128KB.

---

### 3. Privilege Escalation Vulnerability with iam:PassRole

During IAM configuration, developers are often granted `iam:PassRole` permissions coupled with a wildcard (`*`) to simplify deployment. This is done so they can attach IAM Roles to AWS resources like Lambda functions or EC2 instances.

*   **The Threat Scenario:** An IAM User that only has permissions to create Lambda functions alongside `iam:PassRole` for `*` can easily escalate their privileges to become an Administrator.
*   **How it Works:** They simply create a new Lambda function, assign it an existing `AdministratorAccess` IAM Role, and write a small script inside the function to execute any admin actions they desire.
*   **The Fix:** `iam:PassRole` permissions are as powerful as Admin privileges. Always specify exactly which Roles can be passed (using explicit Resource ARNs) instead of using the wildcard `*`.

---

### 4. The Implicit 6-Hour Cooldown Limit of EBS Volumes

AWS provides Elastic Volumes, which allow you to expand disk size or modify volume types (such as transitioning gp2 to gp3) on a running EC2 instance without downtime. While convenient, there is a time constraint trap that is easy to miss.

*   **The Cooldown Period:** Once you modify an EBS Volume, AWS locks the modification capability for that specific volume for exactly 6 hours.
*   **Real-World Incident:** During an out-of-disk emergency, you hurriedly expand a volume from 100GB to 120GB to temporarily resolve the issue. Shortly after, you realize 120GB is still insufficient and need 500GB. AWS will reject the modification request and force you to wait 6 hours before you can modify the volume again.
*   **The Fix:** When modifying EBS Volumes, calculate a generous buffer size from the very first modification attempt, as you will not have a second chance for the next 6 hours.

---

### Conclusion

AWS is vast, and official documentation focuses on describing ideal use-cases. In production environments, however, it is the minor technical details and hidden limits that dictate system stability and budget safety.

These lessons were learned the hard way. If you have run into similar traps on AWS, share them below so the community can avoid them.

---

### Post Information
*   **Facebook Group:** AWS Study Group
*   **Original Post Link:** [Facebook Post Link](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2229330564498570/)