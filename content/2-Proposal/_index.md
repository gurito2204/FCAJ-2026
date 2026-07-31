---
title: "Project Proposal"
date: 2026-06-15
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

# Nabathico - Unveiling Destiny via Hybrid Astrology
Minimalist AWS Serverless Solution for a Cross-Cultural Astrological Application

### 1. Executive Summary
Nabathico is a divination application that combines Eastern philosophical systems (Zi Wei Dou Shu / Tu Vi) and Western systems (Astrological Natal Charts) to provide comprehensive, multi-dimensional interpretations. The platform enables users to quickly look up information through a smooth mobile/web interface. The project maximizes the use of basic AWS Serverless services, combined with AI, to deliver personalized analytical readings with extremely low maintenance costs, easily scalable from a personal project to thousands of active users.

### 2. Problem Statement
**Current Problem**
The current market is fragmented: users have to install separate applications for Eastern Tu Vi and Western Astrology. Manually comparing and synthesizing information between the two theoretical systems (e.g., conflicts between the Ascendant in Tu Vi and the Sun Sign) is extremely complicated for average users. Current chart calculation systems are often disconnected and hard to maintain.

**The Solution**
The platform utilizes a minimalist architecture: the user application communicates with Amazon API Gateway, triggering AWS Lambda to run core algorithms in Python (using the `lasotuvi` and `pyswisseph` libraries). User data is simply stored on Amazon DynamoDB. After generating the charts, the system calls an LLM API (such as OpenAI) to synthesize and output the interpretation. Amazon Cognito is used for secure login management. Key features include: instant dual-chart generation, East-West correlation analysis via AI, and birth profile storage.

**Benefits and Return on Investment (ROI)**
The solution creates a unique niche tech product with the potential to attract a large Gen Z user base. The Serverless architecture eliminates fixed server costs (pay-as-you-go). Development time is drastically reduced as there is no need to design complex infrastructure. The app can generate revenue through a Freemium model (free basic charts, paid in-depth AI interpretations).

### 3. Solution Architecture (Minimalist)
The platform applies a single-flow AWS Serverless architecture to process calculation requests from user devices. Requests are sent via API, computed directly in Lambda's RAM, and results are returned instantly without complex intermediary steps.

**AWS Services Used**
*   **Amazon API Gateway:** Receives requests (birth date and time data) from the mobile/web app.
*   **AWS Lambda:** Handles all core logic (calculating planetary positions, arranging Tu Vi stars, calling AI API).
*   **Amazon DynamoDB:** Ultra-fast NoSQL storage for user profiles and generated chart history.
*   **AWS Amplify:** Hosting and continuous deployment for the Frontend interface.
*   **Amazon Cognito:** Manages user registration/login.

**Component Design**
*   **Frontend (Interface):** The application (React Native or Next.js) is designed for UI/UX on Figma before coding, sending birth data to the system.
*   **Core Engine (Backend):** Python scripts processing astronomical ephemeris libraries.
*   **AI Synthesizer:** The module communicating with the LLM API to "translate" technical parameters (e.g., aspects, ruling stars) into easy-to-understand text.

### 4. Technical Implementation
**Implementation Phases**
The project is divided into 4 streamlined phases:
1.  **Research & Design:** Sketch UI/UX on Figma, test open-source calculation libraries.
2.  **Core Engine Development (Backend):** Package `pyswisseph` and `lasotuvi` libraries into AWS Lambda.
3.  **AI & Database Integration:** Build Prompt Engineering for AI to comprehend chart data; connect DynamoDB.
4.  **Testing & Launch:** Code the Frontend, connect APIs, test Timezone accuracy, and publish to app stores.

**Technical Requirements**
*   **Backend:** Processing logic written in Python. When running tests and configuring the local environment on Windows, use the `python` command (instead of `python3`) to execute calculation scripts to avoid terminal errors.
*   **Libraries:** `pyswisseph` (Western Natal Chart) and lunisolar calendar conversion algorithms (Zi Wei Dou Shu).
*   **Database:** Flexible JSON structure on DynamoDB to store complex data arrays of the 12 zodiac signs and 12 Tu Vi palaces.

### 5. Roadmap & Milestones
*   **Month 0:** Design user flows, draw wireframes, test the accuracy of calculation libraries.
*   **Month 1 (Core System):** Set up AWS (Cognito, DynamoDB, API Gateway). Write Lambda functions to process datetime and coordinate conversions.
*   **Month 2 (AI & Experience):** Design Prompt structure, integrate OpenAI API. Start coding the frontend UI.
*   **Month 3 (Finalization):** Connect Frontend with Backend, test data security, patch display bugs, and launch version 1.0.

### 6. Budget Estimation (MVP)
Costs are highly optimized by leveraging the AWS Free Tier for the initial phase.
**Infrastructure Costs (Estimated monthly for ~1,000 active users):**
*   **AWS Lambda:** $0.00 (within the 1 million free requests limit).
*   **Amazon DynamoDB:** $0.00 (within the 25 GB free storage limit).
*   **Amazon API Gateway:** ~$0.01.
*   **AWS Amplify:** ~$0.35 (frontend hosting and bandwidth).
*   **LLM API (OpenAI/Claude):** ~$5.00 - $10.00 (pay-as-you-go based on actual tokens generated from chart interpretations).
*   **Total AWS/Cloud Cost:** < $1.00/month. Total operational cost (including AI): ~$6-11/month.

### 7. Risk Assessment
**Risk Matrix**
*   **Timezone Inaccuracy:** High impact, medium probability (birth time is the most vital element of a chart).
*   **AI API Latency (Timeout):** Medium impact, high probability (LLMs sometimes take 10-15s to respond).
*   **Exceeding AI Token Budget:** Medium impact, low probability (in case of spam request abuse).

**Mitigation Strategies**
*   **Timezone:** Use international standard timezone database libraries (TZ database) combined with Geo-coding from Google Maps to extract absolute accurate coordinates and timezones.
*   **Latency:** Design the UI to display a "Divining..." or "Connecting the stars..." animation to retain users while Lambda waits for the AI response.
*   **Cost:** Set request quotas on API Gateway and establish AWS Billing Alarms.

### 8. Expected Outcomes
*   **Technical Improvements:** Fully automate star assignments, chart plotting, and information synthesis, replacing manual cross-referencing between separate books or platforms.
*   **Long-term Value:** Build a core Engine for East-West occultism, which can be packaged and sold as a SaaS API for third parties, or expanded to include Tarot and Bazi (Four Pillars of Destiny) in the future.