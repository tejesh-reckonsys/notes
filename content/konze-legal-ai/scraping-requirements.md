---
date: 2025-03-21 14:54:45+05:30
title: Scraping Requirements
---

**1. In hierarchical structure, maintaining table of content structure from legend.com. Folder structure should follow legend URL.**
Example:
![Pasted image 20250321142956.png](/notes/konze-legal-ai/pasted-image-20250321142956.png) 
This page should be in following folder structure: `migration`,`2021-2024`, `2025`, `07-03-2025` > `acts` > `Pages` > `document00000` > `level%20100000.html`. 

**2. For each page, save as html only containing `title`,  `breadcrumps` and `page_content`**. (Do not include other elements such as table of contents and navigation buttons)
For the above example. html should contain head with the `title`. In the body, Only include the div containing `breadcrumps`, and the main `page_content` div.
```html
<html>
<head>
  <title>...</title>
</head>
<body>
	<div class="breadcrumps">...</div>
	<div class="page_content">
		...
	</div>
</body>
</html>
```

**3. Elements to include:**
- anchor elements with href.
- Element structure inside page_content should be maintained.
- Include previous and next buttons in page_content.

**4. Elements to not include:**
- CSS styles. Do not remove other attributes.
- Table of contents.

**5. Include legal instruments in a different folder.**