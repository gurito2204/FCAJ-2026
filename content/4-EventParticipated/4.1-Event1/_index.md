---
title: "Event 1"
date: 2026-7-26
weight: 1
chapter: false
pre: " <b> 4.1. </b> "
---


# Summary Report: "FCAJ x Agentic AI Build Week"

### Purpose of the Event

- Experience and practice AI: Create a platform for students to familiarize themselves with, learn, and apply Agentic AI technology in real-world scenarios to solve practical problems.
- Develop system design skills: Guide students on how to architect systems on AWS, from the ideation stage to product deployment.
- Enhance soft skills: Cultivate teamwork, time management, pressure handling, and product pitching skills within a tight timeframe.
- Problem-solving mindset key takeaways: Learn how to transform a raw idea into a project that addresses actual pain points for businesses.

### Highlighted Project
**Signal Scout**

### Speaker List

- **Le Tan Luc**
- **Do Hoang Hieu**
- **Trieu Quoc Hao**
- **Nguyen Van Duy Khiem**
- **Nguyen Cong Minh**
- **Nguyen Tran Minh Quan**


#### Overview

* **Problem & Pain Point:**
    * Large enterprises and corporations often struggle to monitor and detect early strategic shifts, restructuring, or new directions from competitors.
    * These signals are scattered across financial reports, shareholder meeting documents, press releases, or websites.

* **Solution:**
    * Build a Multi-Agent System to collect, chain, and analyze these scattered signals into a clear, comprehensive narrative.
    * Provide evaluations, index forecasts - ROI, risks, and strategic recommendations - Maintain, Adapt, Accelerate to assist strategy and risk management teams in decision-making.
    * Deliver a user-friendly, intuitive, and transparent Self-service Executive Dashboard.

#### Architecture & Implementation

* **Main Processing Flow of the Multi-Agent System:**
    * **Frontend & Security:** A React UI dashboard hosted on AWS Amplify Hosting, secured by AWS WAF, and authenticated via Amazon Cognito.
    * **Supervisor Agent:**
        * Uses AgentCore Runtime / AgentCore Management to orchestrate sub-agents following an Agent-to-Agent - A2A model.
        * Integrates AgentCore Short-Term Memory to retain session context throughout the conversation.
    * **Sub-agents & Tools:**
        * **Crawler Subagent:** Triggers web scraping tools.
            * Uses Apify for static websites requiring large volumes of data collection.
            * Uses TinyFish for deep dynamic websites or those behind access barriers like login walls.
        * **Data Sanitization:** Filters and cleans data using pure Lambda code right after crawling to reduce token costs and prevent Prompt Injection risks.
        * **Analysis Subagent & Governance:** Employs Amazon Bedrock Guardrails to control input and output data. Data is sent to Langfuse for analysis and evaluation scoring.
    * **Evaluation & Data Persistence:**
        * If the evaluation score is high, data is stored in Amazon S3 and metadata in Amazon DynamoDB, then rendered on the Dashboard.
        * If the score is low, the system triggers a retry mechanism - up to 2 times to save costs. If scores remain low after retries, a human-in-the-loop review tag is assigned.

* **Cost-efficient Architecture:**
    * The team performed a cost analysis and realized that relying heavily on third-party tools like Apify, TinyFish, and Langfuse caused costs to spike from around $35/month up to roughly $359/month under maximum usage.
    * The team proposed a Native AWS improvement by replacing third-party scrapers with AWS Built-in Web/Browser tools, reducing costs while ensuring Data Residency & Compliance for enterprises.

#### Team's Lessons Learned

* **Clear Direction:** Initially, the team had too many ideas leading to heated debates. The lesson learned was to set ego aside, listen to one another, lock in the most feasible solution, and commit to it fully.
* **Execution:** No matter how great an idea is, it holds no value in a Hackathon if it remains on slides. Prioritize building a functional product or demo to prove it solves real user pain points.
* **Teamwork:** Trust, leveraging each member's strengths, and setting aside personal conflicts were key to driving the team through the pressure of working overnight.

#### Key Takeaways

* **Business-driven Tech:** Complex or impressive engineering must serve a concrete business problem and deliver practical value to enterprises.
* **FinOps in AI:** When designing Agentic AI architectures, calculating token costs, choosing between Native AWS and Third-party services, and capping retries are critical to preventing cost explosion.
* **Hackathon & Pitching Skills:**
    * The system does not need to be 100% complete, but the demo must accurately reflect the pain point and MVP workflow.
    * Prepare thoroughly for Q&A, as deep-dive questions from judges indicate that the idea successfully made an impression.

![Photo with Mr. Nguyen Cong Minh](/images/event.jpg)