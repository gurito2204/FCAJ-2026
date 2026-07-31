---
title: "Blog 1"
date: 2026-06-15
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

# 3 AWS NICHE GOTCHAS NOBODY TELLS YOU ABOUT, BUT WILL BREAK YOUR APP

Instead of talking about macro concepts, this post compiles 3 very minor technical details on AWS. They rarely appear on lecture slides but can cause unwarranted costs or cost you days of debugging.

---

### 1. "Invisible" Files on S3: Money Spent, But Files Not Found (Incomplete Multipart Uploads)

When an application uploads a large file to S3 and the network drops mid-transit, the process is aborted.

*   **The Hidden Truth:** The chunk of data transmitted before the connection dropped remains on S3 storage, and AWS silently bills you monthly for this partial data.
*   **The Gotcha:** You cannot see these data fragments in the S3 Console or through standard `aws s3 ls` CLI commands. If you have thousands of failed large uploads, this hidden cost penalty can be significant.
*   **The Solution:** Always configure an **S3 Lifecycle Rule** and check *Delete incomplete multipart uploads* after 1 to 2 days to automatically clean up these hidden fragments.

---

### 2. The IMDSv2 Hop Limit Trap when Running Applications inside Containers

Upgrading from IMDSv1 to IMDSv2 is a mandatory security best practice to prevent IAM Role credentials leaks on EC2. However, if your application runs inside a Docker container on that EC2 instance, it will immediately lose access to the AWS SDK.

*   **The Hidden Cause:** IMDSv2 utilizes the IP packet TTL (Time to Live) index to block Session Hijacking. By default, AWS sets `PutResponseHopLimit = 1`.
*   **The Issue:** Requests traveling from inside the container pass through the Docker virtual bridge network interface (`veth bridge`) to reach the metadata IP (`169.254.169.254`), which counts as 2 hops. The packet is dropped immediately because it exceeds the Hop Limit of 1.
*   **The Solution:** Increase the **Metadata Response Hop Limit** of the EC2 instance from 1 to 2.

---

### 3. AWS Lambda's /tmp Directory is Not as Clean as You Think

Many developers assume that every time a Lambda function is invoked, it runs in a completely isolated and clean environment.

*   **The Hidden Truth:** Due to the **Warm Start** execution model, AWS reuses execution containers for subsequent calls to optimize invocation speed. This means files written to the `/tmp` directory during a previous request will remain there for the next request.
*   **The Consequences:**
    *   *Security Risk:* If you temporarily write files containing personal information for User A and forget to delete them, User B on the next invocation (using the same warm container) might read that file.
    *   *Disk Exhaustion Risk:* The `/tmp` directory accumulates files over multiple invocations, leading to random, hard-to-trace *No space left on device* errors.
*   **The Solution:** Always wrap temporary file writing in a `try...finally` block to ensure files are actively deleted immediately after processing, independent of the Lambda execution lifecycle.

---

### Post Information
*   **Facebook Group:** AWS Study Group
*   **Original Post Link:** [Facebook Post Link](https://www.facebook.com/groups/awsstudygroupfcj/permalink/2229330564498570/)