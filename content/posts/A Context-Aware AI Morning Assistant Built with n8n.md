---
title: A Context-Aware AI Morning Assistant Built with n8n
date: 2026-01-07
tags:
topics:
---
This n8n workflow is the backbone of a **fully automated AI morning assistant**. Every morning, it gathers personal context from an Obsidian vault, combines it with long-term goals and daily focus, and uses a local large language model (LLM) to generate a personalized morning message—delivered directly via Telegram.

The core idea is simple:

> **Use your own notes as the source of truth and let AI reflect on them every morning.**

---

## 1. Daily Trigger (08:00 AM)

The workflow is initiated automatically every day at **08:00 AM** using an n8n `Schedule Trigger`.

This makes the assistant fully proactive—no manual input is required to start the day.

---

## 2. Preparing Daily Context in Obsidian

### 2.1 Load Current 12-Week Cycle Metadata
The flow first retrieves metadata from:
- `02. 12 Week Year / Current Cycle.md`

This file represents the current strategic cycle and provides high-level direction for the AI assistant.

---

### 2.2 Ensure Today’s Daily Note Exists

A separate sub-workflow is executed to check whether today’s daily note already exists in Obsidian.

- If it does not exist, it is created automatically.
- This guarantees that daily context is always available before continuing.

---

## 3. Central Configuration: Obsidian Vault Path

The workflow explicitly sets the base path of the Obsidian vault:

```
/home/obsidian/Documents/PersonalAssistant/
```

By defining this once and reusing it throughout the flow, the setup remains maintainable and easy to adapt to other environments.

---

## 4. Retrieving Contextual Source Files

Several markdown files are then fetched in parallel via an internal HTTP service (`/find_file`). Each file provides a specific layer of context:

- **System Prompt**  
    Defines fixed behavioral rules and instructions for the AI.
- **Focus Points (`Focus.md`)**  
    Captures what deserves attention right now.
- **Keystone Habits (`Keystone Habits.md`)**  
    Represents the core habits that drive consistent behavior.
- **Daily Note (today)**  
    Provides day-specific context; missing or empty notes are handled gracefully.

Together, these files turn Obsidian into a **personal knowledge API** for the AI.

---

## 5. Merging and Structuring the Prompt

All retrieved content is merged into a single data stream.
A custom `Code` node then assembles a **clean, structured prompt** by:

- Combining focus points, keystone habits, and user context
- Removing duplicated or outdated “Context:” sections
- Separating system instructions from dynamic input

This step is critical to keep the LLM input predictable and noise-free.

---

## 6. AI Processing with a Local LLM (Ollama)

The final prompt is sent to a locally hosted LLM via **Ollama**:

- **Model:** `llama3.1:8b`
- **Type:** Chat-based
- **Deployment:** Fully on-premise

A LangChain LLM Chain is used to:

- Apply the system prompt consistently
- Inject the dynamically generated user prompt

This ensures stable, repeatable output while preserving personal nuance.

---

## 7. Delivering the Morning Message via Telegram

The generated AI response is sent directly to **Telegram** as a message.

Key characteristics:
- `Force Reply` enabled to encourage interaction    
- No AI attribution, making it feel like a personal assistant rather than a bot

Telegram becomes the primary daily interface for reflection and intent-setting.

---

## 8. Message Tracking and Cleanup
After the message is sent:
1. Previous message records are cleaned up from an n8n Data Table 
2. The new Telegram `message_id` is stored with a label (`ap_morning`)

This enables advanced follow-up flows such as:
- Reply handling
- Context continuation
- Conversation state management

---
## Summary: What This Workflow Does
Functionally, this n8n flow:
- Triggers automatically every morning
- Ensures daily notes exist in Obsidian
- Pulls strategic, habitual, and daily context
- Builds a clean AI prompt from personal knowledge
- Runs a local LLM for privacy-friendly reasoning
- Delivers a personalized morning message via Telegram
- Tracks conversation state for future automation

👉 **The result:**  
A scalable, privacy-first, and deeply personal AI morning assistant—powered entirely by your own notes and infrastructure.

---
