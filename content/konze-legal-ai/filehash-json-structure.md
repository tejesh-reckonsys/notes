---
date: 2025-04-04 10:18:35+05:30
title: Filehash JSON structure
---

This file is used to store and identify duplicate files in dataset.
## Structure

```json
{
	"<sha256_file_hash>": [
		"<file_1>.pdf",
		"<file_2>.pdf",
		...
	]
}
```

**Keys**: SHA256 hash generated for the pdf file
**Values**: list of file names that have this hash (meaning exactly same content)

*Note:* Hash will be different even if only spacing changes in the pdf. **So it does not represent text.**

### File names

File names contain the folder as well. [Check this for folder structure they provided](/notes/konze-legal-ai/dataset-structure).
Example filename in file_hash.json: `"/acts_07_02_2025/legislation/C2004A01237.pdf"`
(Includes leading `/`)