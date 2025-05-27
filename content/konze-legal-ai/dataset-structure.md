---
date: 2025-04-04 10:17:12+05:30
title: Dataset Structure
---

## Initially provided dataset
```
├── 2024
│   ├── acts_07_12_2024_13_12_2024
│   ├── acts_14_12_2024_16_12_2024
│   ├── acts_17_12_2024_19_12_2024
│   ├── acts_20_12_2024_31_12_2024
│   ├── regulations_07_12_2024_13_12_2024
│   ├── regulations_14_12_2024_16_12_2024
│   ├── regulations_17_12_2024_19_12_2024
│   └── regulations_20_12_2024_31_12_2024
└── 2025
    ├── acts_01_01_2025
    └── regulations_01_01_2025
```

Each folder here represents a version.

For example, `acts_07_12_2024_13_12_2024` is valid from 12th December 2024 to 13th December 2024. `acts_01_01_2025` is valid from 1st January 2025 until the next version.

**Note**: Each version has acts and regulations.

## Legislation vs. PDF

Each folder contains legislation and pdf folders:
```
acts_01_01_2025/
├── legislation
└── pdf
```

- **Legislation**: These are legal instruments (li) files downloaded as PDF.
- **PDFs**: Legend.com webpages downloaded as PDF.

### Legislation
- LI files officially released by Australian Government.
- Does not contain any hyperlinks.

### PDFs
- These are same as legend.com webpages.
- Contains hyperlinks to identify interlinks.