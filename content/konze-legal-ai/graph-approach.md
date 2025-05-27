---
date: 2025-04-04 11:24:21+05:30
title: Graph Approach
---

This is currently used approach.

## Overview

```mermaid
flowchart TD
    A[User Query] --> B[Initial Hybrid Retrieval]
    B --> C[Retrieve Initial Chunks]
    C --> D[Get Connected Files via Graph DB]
    D --> E[Generate Targeted Queries using LLM]
    E --> F[Retrieve Additional Chunks]
    F --> D
    
    F -->|max iterations| I[Combine All Retrieved Chunks]
    I --> J[Generate Final Response]
    J --> K[Return Response to User]
    
    style A fill:#f9d5e5,stroke:#333,stroke-width:2px
    style J fill:#ade8f4,stroke:#333,stroke-width:2px
    style K fill:#f9d5e5,stroke:#333,stroke-width:2px

```

**User Query**: The process begins when a user submits a question or query to the system.

**Initial Hybrid Retrieval**: The system performs an initial search using a combination of methods (Vector + Sparse search) to find relevant information. **Currently, retrieves top 5**

**Get Connected Files via Graph DB**: Using a knowledge graph database, the system identifies all the outgoing references from retrieved chunks.

**Generate Targeted Queries using LLM**: A Large Language Model analyzes the current context and generates more specific, focused follow-up queries to do on top of each related file.

**Retrieve Additional Chunks**: These targeted queries are used to retrieve additional information chunks that might have been missed in the initial retrieval. **Currently retrieves top 3 for each targeted query**

**Recursive Loop**: The system cycles through retrieving connected files, generating targeted queries, and retrieving additional chunks. **(This is repeated twice)**

**Combine All Retrieved Chunks**: After completing the recursive retrieval process, all the gathered information chunks are combined into a comprehensive context.

**Generate Final Response**: The LLM uses all retrieved information to craft a comprehensive and accurate response to the original query.

**Return Response to User**: The final answer is presented to the user who submitted the original query.


### Hybrid retrieval process

```mermaid
flowchart TD
        START --> D1[Dense Retrieval]
        START --> S1[Sparse Retrieval]
        D1 --> DR1[TOP_K Dense Results]
        S1 --> SR1[TOP_K Sparse Results]
        DR1 --> CR1[Combined Results]
        SR1 --> CR1
        CR1 --> RR1[Reranker]
        RR1 --> FR1[Final Ranked Results for Query]
```

[Check this for more info on how chunks are created.](/notes/konze-legal-ai/chunking-strategy)
## LLM agents

- We are using 2 LLM agents. One for generating targeted queries. Another for final response generation.
### Targeted Query Agent

(Defined in `ragcore/agentic/fsm_knowledge_graph.py`)
- The agent accepts user query, and a list of interconnections that are retrieved from graph db.
- Returns specific search queries for each selected file.
- The output is a **structured CSV list** of filename-query pairs.

### Response generation agent

(Defined in `ragcore/agentic/fsm_knowledge_graph.py`)

- Accepts retrieved chunks formatted as xml, user query, and conversation history.
- Generates the final response.