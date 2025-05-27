---
date: 2025-03-31 10:47:57+05:30
title: Knowledge Graph Implementation for Document Retrieval
---

## Overview

Our system implements a graph-based approach to enhance document retrieval by leveraging the semantic connections between documents. Here's how it works:

## Graph Construction Phase

1. **Data Processing**: We analyzed PDF files and scraped URL data to identify interlinks between documents.
2. **Metadata Extraction**: For each interlink, we captured rich metadata including:
   - Anchor text (the visible, clickable text)
   - Page number (source location)
   - Link hashtags (for section references)
   - Other relevant connection data

3. **Neo4j Storage**: All connections and metadata were stored in Neo4j graph database, creating a navigable knowledge structure representing document relationships.

## Query Processing and Retrieval

When a user submits a question, our system follows a multi-step retrieval process:

1. **Initial Vector Search**: We retrieve the top 5 most relevant chunks from our vector database.
2. **Interlink Extraction**: From these initial chunks, we identify all interlinked documents.
3. **Expansion - First Iteration**:
   - We select the most promising interlinks
   - Generate targeted search queries for each link
   - Retrieve top chunks for each of these queries

4. **Expansion - Second Iteration**:
   - We repeat the process with interlinks discovered in the first iteration
   - This creates a second layer of relevant content

5. **Consolidated Analysis**: All retrieved chunks from the initial search plus both expansion iterations are passed to the LLM.

6. **Response Generation**: The LLM synthesizes information from all retrieved chunks to generate a comprehensive response.

This graph-based approach allows us to follow semantic connections between documents, mimicking how a human researcher would explore related materials to build a complete understanding of a topic.