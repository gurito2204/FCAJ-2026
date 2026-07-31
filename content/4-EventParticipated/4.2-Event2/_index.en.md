---
title: "FCAJ Community Day - June 2026"
date: 2026-06-30
weight: 2
chapter: false
pre: " "
---

Summary Report “FCAJ Community Day - June 2026”

#### Event Purpose

- **Community Connection:** Create a space for networking, learning, and sharing practical experience among members, experts, and tech/AI enthusiasts.
- **Updating Trends & New Technologies:** Review key concepts, frameworks, and Voice AI models to help attendees research, build projects, compete in hackathons, or prepare for real-world career opportunities.
- **Solving Real-World Problems:** Listen to specific corporate use cases and learn how technology addresses actual operational pain points at major banks (e.g., VPBank, VIB).
- **Career Development Orientation:** Share opportunities and challenges in deploying Voice AI models within the Vietnamese market.

#### Projects / Topics That Caught My Attention

Voice AI - Building a Voice Mechanism for AI

#### Speakers List

- Mr. Hieu Nghi (Renova Cloud)
- Mr. Kiet (AWS Study Group)
- Mr. Trung (CEO of R AI / Revve AI)

#### Overview - Project / Topic Overview

#### Problems & Pain Points:

- High-quality Vietnamese voice datasets for AI training remain extremely scarce.
- End-to-End Speech-to-Speech models perform poorly with Vietnamese, make real-time guardrailing of AI outputs difficult, and severely limit tool-calling capabilities.
- Practical implementation challenges in the BFSI (Banking, Financial Services, and Insurance) sector due to strict requirements for low latency, high accuracy, and automated workflow execution (e.g., card locking, ID verification).

#### Solution:

- Adopt a 3-part Pipeline Architecture: `STT (Speech-to-Text) -> LLM -> TTS (Text-to-Speech)`.
- Use STT to convert speech to text, feed it into an LLM with specific prompt/agent contexts and execute Tool Calling (locking cards, handling requests), then use TTS to synthesize the voice response.
- Leverage the strong Vietnamese processing capability of current LLMs to ensure full control over output content, accuracy, and data security.

#### Architecture & Implementation

#### System Main Processing Flow:

- **Frontend Layer:** Directly integrated into telephony systems (PBX/Call Center) or banking/customer applications.
- **Logic / Backend & AI Layer:** Utilizes the 3-model pipeline (STT - LLM - TTS) combined with auxiliary models trained specifically for Vietnamese:
  - **Gender Detection:** Identifies male/female voices upfront to apply appropriate Vietnamese honorifics (Anh/Chi), preventing awkward customer interactions.
  - **Interruption & Context Handling (Barge-in & Turn-taking):** Prevents AI from prematurely interrupting customers when they pause mid-sentence (e.g., pausing while reciting a phone number), while stopping AI audio immediately when a user speaks over it.
- **Governance & Security:** Validates text outputs before passing them to TTS, with seamless human-in-the-loop escalation when AI encounters edge cases.

#### Cost-efficient Architecture:

- Prioritizes real-time streaming across all 3 layers: STT streams text as audio arrives -> LLM streams text output -> TTS streams audio immediately without waiting for the full sentence to finish, minimizing latency and maximizing conversational responsiveness.

#### Team's / Speaker's Lessons Learned

- **Context-awareness:** Must deeply understand user context and Vietnamese linguistic nuances (gender detection for correct honorifics, appropriate pause detection to avoid interrupting customers).
- **Accent Handling & Persona:** Incorporates 10%–20% regional accent data into STT/TTS training datasets so the AI understands diverse regional dialects, while maintaining a consistent professional voice persona instead of mimicking customer accents.
- **Human-in-the-loop:** AI serves as an assistant. Systems must allow human operators to jump in smoothly whenever AI struggles or customer dissatisfaction arises.

#### Key Takeaways

- **Business-driven Tech & Tool Calling:** Voice AI delivers real value only when tied to concrete business scenarios and capable of executing actual tasks (locking cards, account support).
- **Resource Optimization:** Latency control via streaming and natural interruption handling are key to successfully deploying AI in large enterprise/banking environments.
- **Value of Native Language Understanding:** Deep research into Vietnamese linguistic nuances (honorifics, natural pause handling, regional accents) serves as the core competitive advantage for Voice AI solutions in Vietnam.