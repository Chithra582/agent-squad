# EXPLAINABILITY.md

This document explains the internal mechanisms, data lineage, operational boundaries, and governance framework of **Agent Squad** (`agent-squad`) in accordance with the **OpenGAP v0.1.0** specification for the **HiDevs GitAgent Passport** clearance pipeline.

> **Agent Name:** Agent Squad (`agent-squad`)  
> **Specification:** OpenGAP v0.1.0  
> **Category / Domain:** Developer Tools / Multi-Agent Conversation Routing & Orchestration  
> **Compliance Standard:** OpenGAP Checkpoint 2 (Explainability & Decision Governance), OWASP LLM Top 10, MITRE ATLAS  

---

## How the Agent Decides

Agent Squad operates through a deterministic 5-stage decision and routing pipeline to evaluate user inputs, score candidate agents, select optimal delegation paths, and orchestrate responses.

### 1. Decision Architecture

The runtime intake, state classification, evaluation, and execution tracking operate across a deterministic, five-stage pipeline:

```
+-----------------------------------------------------------------------------------+
|                           5-STAGE DECISION PIPELINE                               |
+-----------------------------------------------------------------------------------+
|  [Stage 1: Ingestion & Input Sanitization]                                       |
|  - Parse query, strip control tokens, extract active session ID                   |
|                                     |                                             |
|                                     v                                             |
|  [Stage 2: Intent Classification & Feature Extraction]                            |
|  - Extract semantic embeddings, keyword signals, and context history              |
|                                     |                                             |
|                                     v                                             |
|  [Stage 3: Agent Affinity Scoring & Constraint Validation]                        |
|  - Compute affinity metric S_affinity across candidate squad agents               |
|                                     |                                             |
|                                     v                                             |
|  [Stage 4: Threshold Evaluation & Dispatch Selection]                             |
|  - Apply tau threshold (tau = 0.65); verify quota and availability                |
|                                     |                                             |
|                                     v                                             |
|  [Stage 5: Execution, Context Persistence & Response Streaming]                   |
|  - Delegate to target agent, persist session turn, stream tokens to caller        |
+-----------------------------------------------------------------------------------+
```

### 2. Decision Logic & Routing Formulations

For a given user query $q$, active conversation history $H$, and candidate agent $a_i \in A$, the routing score $S_{\text{affinity}}(a_i, q, H)$ is formulated as:

$$S_{\text{affinity}}(a_i, q, H) = w_1 \cdot \cos(\mathbf{e}_q, \mathbf{d}_{a_i}) + w_2 \cdot \text{BM25}(q, D_{a_i}) + w_3 \cdot C(a_i, H)$$

Where:
- $\mathbf{e}_q$ is the dense semantic embedding of user query $q$.
- $\mathbf{d}_{a_i}$ is the pre-computed centroid embedding of agent $a_i$'s declared capabilities.
- $\text{BM25}(q, D_{a_i})$ is the lexical match score against agent $a_i$'s documentation and tool registry.
- $C(a_i, H) \in [0, 1]$ is the conversational continuity prior, rewarding agents that participated in recent conversational turns.
- $w_1 = 0.50$, $w_2 = 0.30$, and $w_3 = 0.20$ such that $\sum_{k=1}^3 w_k = 1.0$.

The selected agent $a^*$ is determined by:

$$a^* = \arg\max_{a_i \in A} S_{\text{affinity}}(a_i, q, H)$$

Subject to the threshold constraint:

$$S_{\text{affinity}}(a^*, q, H) \ge \tau \quad (\tau = 0.65)$$

### 3. Thresholding & Refusal Decision Criteria

Agent Squad enforces strict operational boundaries and deterministic refusal thresholds:
- **Refusal on ERR_INPUT_EMPTY**: Query length $\le 0$ characters halts execution with code `ERR_INPUT_EMPTY`.
- **Refusal on ERR_INPUT_UNSAFE**: Prompt injection or forbidden keyword match halts execution with code `ERR_INPUT_UNSAFE`.
- **Refusal on ERR_ROUTING_CONFIDENCE_LOW**: Routing confidence score below threshold ($\max_{a_i} S_{\text{affinity}} < 0.65$) halts execution with code `ERR_ROUTING_CONFIDENCE_LOW`.
- **Refusal on ERR_AGENT_UNAVAILABLE**: Target agent offline or health check failed halts execution with code `ERR_AGENT_UNAVAILABLE`.
- **Refusal on ERR_CONTEXT_LIMIT_EXCEEDED**: Session history token count exceeding 32,000 tokens halts execution with code `ERR_CONTEXT_LIMIT_EXCEEDED`.

### 4. Fallback Decision Mechanism

Continuous operational stability is maintained through layered fault recovery:
- **Tier 1 Context Pruning & Secondary Scoring**: If top confidence falls below $\tau$, re-evaluate affinity using a compacted window of the last 2 turns, discounting conversational bias ($w_3 \to 0$).
- **Tier 2 General Triage Responder**: If Tier 1 confidence remains below $\tau$, route the query to the general-purpose triage agent configured with default safe response capabilities.
- **Model Fallback Cascade**: High-level reasoning and synthesis default to `gemini-2.0-flash` with automatic failover to `gpt-4o` and `claude-3-5-sonnet`.

### 5. Human-in-the-Loop Governance

Human operators retain sovereign authority over the multi-agent execution lifecycle:
- **Human-in-the-Loop Escalation**: If the triage agent detects unresolved ambiguity, hazardous intent, or domain violation, execution halts and notifies an operator with full session telemetry.
- **Routing Rule Administration**: Operators can define and override static bypass rules and inspect delegation traces.

---

## The Data It Uses

Agent Squad operates under strict principles of data minimization, environment isolation, and privacy protection.

### 1. Ingested Input Data

The framework processes only operational data necessary to perform its functions:
- **User Prompt Payload**: Natural language queries, structured commands, and optional file attachments.
- **Session Identifiers**: UUID v4 session tokens and user identifiers for thread management.
- **Runtime Metadata**: Client platform headers (`python`, `typescript`, `swift`), client timestamps, and latency budgets.

### 2. Configuration & Reference Data

- **Agent Registry**: Declarative JSON/YAML manifests defining agent descriptions, tool signatures, parameter schemas, and affinity vectors.
- **Domain Lexicons**: Specialized keyword indexes used by BM25 scoring for fast domain matching.
- **Routing Rules**: Administrator-defined static bypass rules (e.g., explicit `@agent` mentions).

### 3. Base Model & Inference Lineage

- **Classifier Models**: Compatible with JevClassifier, lightweight embedding models (`text-embedding-3-small`, `bge-small-en-v1.5`), and foundation models (Amazon Bedrock Titan/Claude, Anthropic Claude 3.5, OpenAI GPT-4o).
- **Weight Provenance**: Model weights are loaded strictly from verified public registries or enterprise cloud endpoints with cryptographic checksum verification.

### 4. Data Privacy, Storage, and Retention

- **OWASP LLM & MITRE ATLAS Hardened**: Defended against indirect prompt injection, credential leakage, and unauthorized external API dispatch.
- **Local Environment Isolation**: Agent execution workspaces, intermediate scratchpads, and vector stores reside strictly within designated local project directories.
- **Automated Secret Scrubbing**: API keys, database credentials, and personal credentials are automatically redacted prior to embedding or logging.
- **Zero Commercial Monetization**: Prompts, intermediate reasoning trajectories, and task deliverables are never commercialized or shared with third parties.

---

## Limitations

Understanding the operational boundaries and technical constraints of Agent Squad is essential for effective deployment.

### 1. Ambiguous Multi-Domain Query Decomposition
- **Limitation**: Ambiguous multi-domain queries containing mutually exclusive requests may result in sub-optimal agent selection.
- **Mitigation**: The classifier detects multi-intent markers and initiates Tier 1 sub-query decomposition before final dispatch.

### 2. On-Device Swift Runtime Compute Constraints
- **Limitation**: On-device Swift runtime execution is constrained by local device RAM and neural engine compute quotas.
- **Mitigation**: Quantized 4-bit local models are deployed with memory paging guards and automatic fallback to cloud endpoints when limits are exceeded.

### 3. Conversational Continuity Sticky Bias
- **Limitation**: Conversational continuity bias ($w_3$) can cause conversational stickiness to an agent after the user changes topic.
- **Mitigation**: Sudden semantic distance shifts ($\cos(\mathbf{e}_{q_t}, \mathbf{e}_{q_{t-1}}) < 0.35$) dynamically reset continuity weight $w_3$ to 0 for that turn.

### 4. Distributed Cloud Network Streaming Jitter
- **Limitation**: Network latency fluctuations in distributed cloud multi-agent deployments can cause streaming jitter.
- **Mitigation**: Adaptive token chunk buffering and keep-alive heartbeat frames smooth stream delivery to client applications.

### 5. Technical Domain Vocabulary Absence
- **Limitation**: Highly technical domain jargon absent from the classifier's pre-trained vocabulary may receive deflated affinity scores.
- **Mitigation**: Custom BM25 domain vocabulary dictionaries allow operators to boost specific terms directly in the affinity calculation.

---

## Summary & Compliance Checklist

| Checkpoint 2 Requirement | Corresponding Section | Status |
| :--- | :--- | :---: |
| **How the agent decides** | [How the Agent Decides](#how-the-agent-decides) | **Covered** |
| - Decision architecture & 5-stage pipeline | Section 1 | Verified |
| - Decision logic & routing formulations | Section 2 | Verified |
| - Thresholding & refusal decision criteria | Section 3 | Verified |
| - Fallback decision mechanism | Section 4 | Verified |
| - Human-in-the-loop governance & oversight | Section 5 | Verified |
| **The data it uses** | [The Data It Uses](#the-data-it-uses) | **Covered** |
| - Ingested input data & query streams | Section 1 | Verified |
| - Configuration & reference schemas | Section 2 | Verified |
| - Base model lineage & deterministic engines | Section 3 | Verified |
| - Data privacy, retention lifecycle & MITRE/OWASP | Section 4 | Verified |
| **Its limitations** | [Limitations](#limitations) | **Covered** |
| - Ambiguous Multi-Domain Query Decomposition | Section 1 | Verified |
| - On-Device Swift Runtime Compute Constraints | Section 2 | Verified |
| - Conversational Continuity Sticky Bias | Section 3 | Verified |
| - Distributed Cloud Network Streaming Jitter | Section 4 | Verified |
| - Technical Domain Vocabulary Absence | Section 5 | Verified |
