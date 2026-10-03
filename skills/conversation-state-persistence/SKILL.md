---
name: "conversation-state-persistence"
description: "Persist, synchronize, and retrieve conversational state across distributed agent turns."
---

# Conversation State Persistence Skill

Maintains coherent conversational memory and state across multiple participating agents.

## Core Capabilities
- Cross-agent session synchronization and memory compaction.
- Pluggable storage support for in-memory, DynamoDB, Redis, and SQLite.
- Automatic sliding-window compaction to satisfy model context boundaries.
