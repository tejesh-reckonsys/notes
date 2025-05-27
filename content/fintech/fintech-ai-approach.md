---
date: 2025-02-14 14:57:15+05:30
title: Fintech AI Approach
---

```mermaid
flowchart

A(User Query) --> B(Call AI with Columns data)
B -->|Enough data| X(Generate response)
B -->|Not enough data| C(call function for more data)
C --> B
X -->|Table response| Y(Generate SQL for duckdb and send table)
X -->|Calculation response| Z(Calculate response and send formula & calculation)
```

#### Things missing in PoC:
- ***Comparing peers data.*** Can implement by adding a function for getting peers data
- ***Returning charts/graphs:*** Currently only returning tables or text. Can inegrate other output formats by giving context to AI.

**NOTE:**
- In PoC, it might interpret question somewhat incorrectly. Need to provide more context for better results.