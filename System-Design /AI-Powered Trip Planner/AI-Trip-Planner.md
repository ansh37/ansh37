# System Design Deep Dive: AI-Powered Trip Planner (Disney Scale)

![Level](https://img.shields.io/badge/Level-Staff%20%2F%20L5-blue)
![Topic](https://img.shields.io/badge/Topic-System%20Design%20%7C%20GenAI-success)

## Overview
This document outlines the architecture for an AI-powered Trip Planner Agent, designed to handle 10 million Daily Active Users (DAU). The system allows users to interact with a conversational AI to plan their trip, check real-time park data, and autonomously book tickets and hotels.

The primary engineering challenge is bridging the gap between **probabilistic LLM generation** (which requires high availability and low latency) and **deterministic booking systems** (which require strict ACID consistency).

---

## 1. Requirements

### Functional Requirements (FRs)
* **Conversational AI:** Users can chat via text to plan trips (rides, hotels, dining).
* **Personalization & RAG:** The agent recommends attractions based on real-time park data (e.g., wait times) and user history.
* **Transactional Agent:** The bot can execute actual bookings on behalf of the user.
* **Human Handoff:** Seamless routing to a human support agent if the bot fails or the user requests it.

### Non-Functional Requirements (NFRs)
* **NFR Split:**
  * **Chat/Search Path:** High Availability (HA) and Low Latency (`< 1s` Time-to-First-Token). Read-heavy.
  * **Booking Path:** Strict Consistency (ACID). We cannot overbook ride capacity or hotel rooms.
* **Scale:** 10 Million Daily Active Users (DAU).
* **Brand Safety:** Zero tolerance for toxic responses or hallucinated policies.

---

## 2. Back-of-the-Envelope (BoTE) Calculations
* **Traffic:** 10M DAU × 10 messages/day = **100M messages/day**.
* **Throughput:** 100M / 86,400 = **~1,200 Requests Per Second (TPS)** average. Peak: **~3,000 TPS**.
* **Network Constraint:** LLM generation takes several seconds. Standard HTTP connections will time out or cause terrible UX. **WebSockets (WSS)** are mandatory to stream tokens back to the client and maintain 10M concurrent connections.

---

## 3. Core Entities & APIs

### Entities
* `Session`: `session_id`, `user_id`, `chat_history` (Array), `active_intent`
* `KnowledgeChunk`: `chunk_id`, `text`, `embedding_vector`, `metadata` (tags)
* `Booking`: `booking_id`, `user_id`, `resource_id`, `status` (PENDING, CONFIRMED, FAILED)

### Core APIs (WebSocket + Internal REST)
* `WSS /v1/chat/stream?session_id={id}` -> Bidirectional streaming for the LLM chat.
* `POST /v1/agent/tools/book` -> Internal API called by the LLM Agent to trigger a transaction.
* `POST /v1/handoff` -> Transfers session state to the human customer service queue.

---

## 4. High-Level Design (HLD)
<img width="2639" height="1148" alt="image" src="https://github.com/user-attachments/assets/c1081036-2343-45f4-9a09-930fec52b9db" />

## 5. Deep Dives
### Deep Dive 1: Brand Safety & Output Guardrails
LLMs hallucinate. For a family-friendly brand, toxic or incorrect output is a critical incident.

- Architecture: The LLM never streams directly to the user. It streams to a Response Validator (using a smaller, ultra-fast model like NeMo Guardrails or a heuristic rule engine).

- Action: If a policy violation is detected mid-stream, the connection drops the bad tokens and injects a safe fallback: "I can only help plan magical vacations! Let's talk about rides."

### Deep Dive 2: The Agentic Workflow (Tool Calling)
LLMs cannot write to databases. We implement Function Calling.

- Flow: We provide the LLM with a JSON schema of our internal tools (e.g., book_ticket(date, ride_id)). When the user asks to book, the LLM outputs a JSON payload matching the schema instead of conversational text.

- Execution: The Orchestrator intercepts this JSON, pauses the LLM, drops the event into Kafka, and waits for the transactional worker. Once the SQL DB confirms the booking, the Orchestrator feeds the Success payload back into the LLM context so it can naturally say, "You're all set!"

###Deep Dive 3: Distributed Transactions (The Saga Pattern)
When a user says "Book my flight and my hotel," this spans multiple microservices.

- The Problem: What if the flight books, but the hotel fails? We cannot leave the user in a half-booked state.

- The Solution: The Kafka booking workers implement the Saga Pattern. If step 2 (hotel) fails, the worker executes a compensating transaction to explicitly cancel step 1 (flight), and notifies the LLM so it can ask the user for alternative dates. All events use Idempotency Keys to prevent double-booking during network retries.

### Deep Dive 4: Managing Context Windows at Scale (Redis)
Sending a 50-message chat history to the LLM for every new request wastes tokens and spikes latency.

- The Optimization: We use a Sliding Window + Async Summarization pattern. Redis stores the exact verbatim text of the last 5 messages. For anything older, a background asynchronous worker compresses the history into a dense summary (e.g., "User wants to visit Epcot, prefers fast rides, budget is $500"). This keeps the prompt lean and token costs low.

### Deep Dive 5: LLM-Ops & Production Observability
Building the system is only half the battle; operating it requires AI-specific observability.

- RAG Evaluation: We log all User Queries and retrieved Vector DB chunks to a data lake. Offline jobs measure Retrieval Precision to ensure the bot isn't pulling hotel data when the user asked about rollercoasters.

- Feedback Loops: The UI includes implicit (session length, successful bookings) and explicit (thumbs up/down) signals. These are piped back via Kafka to fine-tune future iterations of the routing model.
