---
date: 2025-03-05 14:29:27+05:30
title: Konze Legal AI Approaches
---

For our Retrieval Augmented Generation (RAG) implementation, we explored several different approaches to chunking and retrieval.

### Chunk Naming Convention
All approaches used the same naming convention for chunks. Each chunk was uniquely identified using a combination of:
- The hash of the original source file
- A sequential index number

This naming convention ensured:
1. Traceability back to source documents
2. Unique identification of each chunk
3. Ability to reconstruct document order when needed
### Approach 1 - Semantic Chunking (with only dense retrieval)
Our initial approach utilized semantic chunking to.

**Chunking process:**
1. Extract text content from PDF documents using pymupdf
2. Split text into chunks using RecursiveCharacterSplitter
3. Use voyage embeddings to merge adjacent chunks if they are semantically similar
4. Generate unique chunk ID based on source file hash and sequential index
5. Generate dense embeddings
6. Upload the data to pinecone

**Technical Information:**
- Implemented: RecursiveCharacterSplitter with voyage-law-2 embeddings
- Initial chunk size: 768 characters
- Target max size: 4096 characters
- Actual result: Most chunks remained at 768 characters due to implementation constraints
- Merge operation: Adjacent chunks intended to be combined but merge functionality did not achieve target size
- Retrieval: 25 results initially fetched and reranked using bge-reranker-base to get top 15 most relevant chunks

**Issues:**
- Chunks are smaller than expected, which made them not contain full information.
- Since we are parsing pdf documents, paragraphs were not identified, so semantic chunking was done on lines, which did not work.

### Approach 2 - Hybrid Dense and Sparse Retrieval
We extended the first approach by adding sparse retrieval alongside the dense embeddings.

**Chunking process:**
1. Use same chunks generated in Approach 1
2. Fit BM25 model on the chunks
3. Generate sparse embeddings using fitted BM25 model
4. Upload the data to pinecone

**Technical Information:**
- Implementation: BM25 encoder from pinecone-text library
- Dense retrieval: Maintained same voyage-law-2 embeddings approach
- Sparse retrieval: BM25 model fit on our specific document corpus
- Capabilities: Combined lexical (BM25) and semantic (embeddings) search
- Architecture: Hybrid system utilizing both retrieval methods in parallel
- Retrieval: Fetched 15 chunks from dense and 15 from sparse retrieval, reranked using bge-reranker-base to get top 15 most relevant chunks

**Improvements:**
- During retrieval, we had better results because of 2 types of retrieval.

**Issues:**
- We used same chunks as last approach, so they still did not contain full information.
- The BM25 model did not perform well with our specific document corpus.
### Approach 3 - Traditional Chunking with Hybrid Retrieval
We moved to a traditional chunking approach while maintaining the hybrid retrieval system.

**Chunking process:**
1. Convert PDF documents to structured markdown using pymupdf4llm to preserve formatting/structure
2. Split markdown into chunks using tiktoken tokenizer
3. Generate unique chunk ID based on source file hash and sequential index
4. Generate dense and sparse embeddings
5. Upload the data to pinecone

**Technical Information:**
- Chunk size: 800 tokens
- Overlap: 400 tokens between chunks
- Dense retrieval: voyage-law-2 embeddings
- Sparse retrieval: BM25 model with default weights
- Retrieval: Fetched 10 results each from dense and sparse retrieval, reranked using bge-reranker-base to get top 7 most relevant chunks

**Improvements:**
- Each chunk was larger, and had enough context to provide a comprehensive answer.
- Markdown conversion preserved document structure better than raw text extraction

**Issues:**
- Chunks are larger, so had to decrease the count of chunks retrieved and passed to LLM.

### Enhancements
We further improved the retrieval process through several iterations while keeping the same chunking approach.

#### Enhancement 1 - Conversation Memory & Query Rephrasing
**Technical Information:**
- Store summaries of previous conversation turns
- Use conversation context to rephrase user queries for better relevance
- Improved query understanding through historical context

**Improvements:**
- More contextually aware retrieval
- Better handling of follow-up questions
- Reduced redundancy in responses

#### Enhancement 2 - Multi-Query Generation
**Technical Information:**
- Multiple queries generated for each user question and processed in parallel
- Retrieval: 40-50 chunks retrieved across all queries
- Different perspectives captured through query variations

**Improvements:**
- More comprehensive retrieval through multiple query angles
- Better coverage of relevant information
- Reduced chance of missing important context

#### Enhancement 3 - Improved Reranking
**Technical Information:**
- Switched from bge-reranker-base to Pinecone reranker
- Final output: Top 7 most relevant chunks after reranking

**Improvements:**
- Faster reranking process with lower system resource usage
- Better quality of final chunk selection
- More efficient processing pipeline