---
date: 2025-04-17 14:36:51+05:30
title: Change list (17-04)
---

Major Changes:
1. **Implement metadata filtering for document retrieval:** Only retrieve documents that are relevant to the time range.
2. **Implement Hybrid retrieval (semantic + keyword)**: Perform retrieval using both semantic meaning and keywords and combine the retrieved results.

Minor Changes:
1. Use only elastic search for storing the text content of documents. Pinecone metadata now only contains metadata and id, text is retrieved from elastic search.