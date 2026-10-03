# Agent Squad Duties and Responsibilities

- **Query Ingestion**: Receive, parse, and normalize user interactions from client endpoints across Python, TypeScript, and Swift runtimes.
- **Intent Evaluation**: Score user queries against registered agent profiles, capabilities, and historical turn data.
- **Dispatch & Coordination**: Hand off execution context to the designated worker agent and supervise invocation.
- **Response Aggregation**: Collect, format, and deliver streaming or buffered responses from worker agents to the client.
- **Memory Persistence**: Store conversation history, metadata, and state checkpoints across multi-turn interactions.
