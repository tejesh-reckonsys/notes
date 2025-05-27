---
date: 2025-04-04 11:24:28+05:30
title: Project Structure
---

# Project Structure Overview

## Overview
### Core Components
- **`ragcore/`** - Main source code
- **`logging_handler/`** - [Service for handling logs by pushing to MongoDB](/notes/konze-legal-ai/logging-handler-service)
- **`streamlit_graph.py`** - Current Streamlit entrypoint ([graph approach](/notes/konze-legal-ai/graph-approach))
- **`streamlit_agentic.py`** - Streamlit entrypoint for [agentic approach](/notes/konze-legal-ai/finite-state-machine-based-agentic-approach)

### Support Components
- **`logs/`** - Directory for storing logs
- **`notebooks/`** - Python notebooks for Zed
- **`scripts/`** - Helper scripts

### Configuration Files
- `compose.yml` - Main Docker Compose file
- `Dockerfile` - Main project Dockerfile
- `pyproject.toml` - Python project configuration
- `uv.lock` - Dependency lock file

### Data Files
- `bm25_encoder_weights.json` - Used by BM25 sparse embedding
- `file_hash.json` - [Stores file hashes for PDF duplicate identification](/notes/konze-legal-ai/filehash-json-structure) 
- `log_processor_state.json` - Maintains logging service state
- `new_file_hash.json` / `old_file_hash.json` - Snapshots of file hash states

### Legacy Files
- `streamlit.py` - Old Streamlit entrypoint
- `OLDREADME.md` - Previous documentation


## Main source code

```
ragcore/
├── *agentic*
├── *ai_models*
├── *conversation*
├── *filehash*
├── *knowledge_graph*
├── *markdown_chunking*
├── *processing*
├── *retrieval*
├── *sources*
├── *utils*
├── embedding.py
├── __init__.py
├── logging.py
├── process_chunks.py
├── process_query_agentic.py
├── process_query_gpt_4_5.py
├── process_query_graph.py
├── process_query.py
├── retriever.py
├── settings.py
├── upload_documents.py
└── upload_to_pinecone.py
```