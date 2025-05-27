---
date: 2025-02-11 12:19:18+05:30
title: Feature Description
---

### Questions:
1. Is this conversation based? That is, can the user ask follow up questions?
2. In some questions, they mentioned `peer data`. Is this also present in the company dataset? Otherwise how do we fetch this data?

### Data needed:
1. Fields present in master data
2. Widgets already present and their use-case
3. Common finance related terms (Not sure if AI can directly understand them)

### Approach:
- Create a base prompt that contains the details about the platform, such as available fields and widgets.
- Pass the user query to the LLM.
	- If there is existing widget for this, return that widget. Might need to find parameters such as date range.
	- If the computation is direct and can directly be displayed as text, generate a text as response.
- For deriving new data from existing data, there are 2 approaches:
	1. Can use duckdb, which supports sql queries on pandas dataframes. ***This might be ideal approach, need to test.***
	2. If the types of data is limited, can define functions for each type of calculation.
	3.  Can use pandas `Dataframe.query()`, by having LLM generate the query needed.

### Types of questions (Using examples)

1. Trend analysis: Comparing periodic changes in a specific field. **Output:** Graph/chart showing the change in data over time.
2. Relationship analysis: Comparing relationship between 2 or more fields in the data. **Output:** Graph showing the different fields for user to compare.
3. Derived field: Derive a field from existing fields.