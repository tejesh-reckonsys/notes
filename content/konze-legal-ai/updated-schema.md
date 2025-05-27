---
date: 2025-03-25 11:02:17+05:30
title: Updated Schema
---

# Vector DB

**Metadata:**
```json
{
	"names": ["abc.pdf"],
	"start_date": 8780804,
	"end_date": 9707947, // Dates in ordinal format
	"start_page": 1,
	"end_page": 2,
	"summary": "Chunk summary (Optional)",
	"title": "Page title (Optional)",
	"kind": "regulations|acts|li",
	"hash": "7f533a447ac499e2998bdd7a589b626afd7171cde5dc48217c9a8e8c40cf09a6"
}
```


# Graph DB
## Node Structure

The node structure appears to be well-defined with clear properties:

```
Node {
  file_name: String,
  url: String,
  title: String,
  kind: String // enum: ["acts", "regulations", "LI", "other"]
}
```

## Relationship Structure

The "references" relationship is defined with directional flow and properties:

```
(Original:Node)-[r:REFERENCES {
  pages: Json.stringified [Integer],
  details: Json.stringified [Map]
}]->(Referenced:Node)
```

Where `details` is an array of maps containing:
- hashtag: String
- searchstring: String
- anchor_text: String
- page_no: int