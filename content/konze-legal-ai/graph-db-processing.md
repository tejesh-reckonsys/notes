---
date: 2025-04-07 09:25:08+05:30
title: Graph DB Processing
---

# Processing Legal Document Links and Creating a Graph Database

## Initial Data Loading (Run for each version)

### 1. Load and Filter Source Data
```sql
-- Load acts and regulation file data
CREATE TABLE scraped_data AS (
    WITH combined_sources AS (
        SELECT url, text, unnest(split(file_name, ',')) AS unnested_file_name
        FROM read_csv('acts_07_02_2025.csv')
        WHERE file_name IS NOT NULL
        UNION ALL
        SELECT url, text, unnest(split(file_name, ',')) AS unnested_file_name
        FROM read_csv('regulations_07_02_2025.csv')
        WHERE file_name IS NOT NULL
    )
    SELECT url, text, unnested_file_name[10:] AS file_name
    FROM combined_sources
);

CREATE TABLE valid_files AS SELECT * FROM scraped_data WHERE (contains(url, '/acts/') OR contains(url, '/regs/')) AND NOT contains(file_name, '/legislation/');
-- COPY valid_files TO 'valid_links_07_02_2025.csv' WITH CSV HEADER;
```
This creates a consolidated table from both acts and regulations files, then filters for valid URLs containing '/acts/' or '/regs/' but not '/legislation/'.

### 2. Process Legislation Data
```sql
CREATE TABLE li_scraped_data_20_12_2024_31_12_2024 AS (
     WITH combined_legislation_data AS (
         SELECT process_url, title, NOT contains(force, 'No longer in force') AS in_force,
                file_name, unnest(split(file_name, ',')) AS unnested_file_name, 'act' AS source_type
         FROM read_csv('legislation_acts_20_12_2024_31_12_2024(in).csv')
         WHERE is_download AND file_name <> 'nan' AND version <> 'Superseded version'
         UNION ALL
         SELECT process_url, title, NOT contains(force, 'No longer in force') AS in_force,
                file_name, unnest(split(file_name, ',')) AS unnested_file_name, 'regulation' AS source_type
         FROM read_csv('legislation_regulations_20_12_2024_31_12_2024(in).csv')
         WHERE is_download AND file_name <> 'nan' AND version <> 'Superseded version'
     )
     SELECT replace(process_url, 'http://', 'https://') AS url, title, in_force, unnested_file_name[10:] AS file_name
     FROM combined_legislation_data
     WHERE unnested_file_name <> 'nan'
);

copy li_scraped_data_20_12_2024_31_12_2024 to 'li_scraped_data_20_12_2024_31_12_2024.csv';
```
This processes legislation data, combining acts and regulations, and includes information about whether items are in force.

## Main Processing Pipeline (Run once)

### 3. Extract Links from PDF Files
```bash
python -m ragcore.knowledge_graph.extract_links --output all_extracted_links.csv ./Immi_legend_data/
```
This command extracts all links from PDF files located in the `**/pdf/*.pdf` pattern within the specified directory and outputs them to a CSV file.

### 4. Process Extracted Links
```sql
-- Processing extracted links
CREATE TABLE all_extracted_links AS FROM read_csv('all_extracted_links.csv');

-- Add version to the extracted links
ALTER TABLE all_extracted_links ADD COLUMN version TEXT;
UPDATE all_extracted_links SET version='07_12_2024_13_12_2024' WHERE contains(file_path, '07_12_2024_13_12_2024');
UPDATE all_extracted_links SET version='14_12_2024_16_12_2024' WHERE contains(file_path, '14_12_2024_16_12_2024');
UPDATE all_extracted_links SET version='17_12_2024_19_12_2024' WHERE contains(file_path, '17_12_2024_19_12_2024');
UPDATE all_extracted_links SET version='20_12_2024_31_12_2024' WHERE contains(file_path, '20_12_2024_31_12_2024');
UPDATE all_extracted_links SET version='01_01_2025' WHERE contains(file_path, '01_01_2025');
UPDATE all_extracted_links SET version='07_02_2025' WHERE contains(file_path, '07_02_2025');

-- Add from_kind to the extracted links
ALTER TABLE all_extracted_links ADD COLUMN from_kind TEXT;
UPDATE all_extracted_links SET from_kind='regulations' WHERE starts_with(file_path, 'regulations');
UPDATE all_extracted_links SET from_kind='acts' WHERE starts_with(file_path, 'acts');
```
This loads all extracted links and adds metadata:
- `version` based on date patterns in file paths
- `from_kind` to categorize source types (acts/regulations)

### 5. Filter Valid Links
```sql
-- Get valid links FROM Step 1
CREATE TABLE valid_links AS (
    SELECT * FROM read_csv('valid_links_01_01_2025.csv')
    UNION ALL SELECT * FROM read_csv('valid_links_07_12_2024_13_12_2024.csv')
    UNION ALL SELECT * FROM read_csv('valid_links_20_12_2024_31_12_2024.csv')
    UNION ALL SELECT * FROM read_csv('valid_links_17_12_2024_19_12_2024.csv')
    UNION ALL SELECT * FROM read_csv('valid_links_14_12_2024_16_12_2024.csv')
    UNION ALL SELECT * FROM read_csv('valid_links_07_02_2025.csv')
);

-- Get only links from valid files
CREATE TABLE valid_extracted_links AS SELECT * FROM all_extracted_links WHERE file_path IN (SELECT file_name FROM valid_links);
```
This combines valid links from all versions and filters the extracted links to include only those from valid files.

### 6. Normalize and Categorize Links
```sql
-- Replace comlaw.gov.au with legislation.gov.au because this is a redirect
UPDATE valid_extracted_links SET url = replace(url, 'http://www.comlaw.gov.au', 'https://www.legislation.gov.au');
UPDATE valid_extracted_links SET url = replace(url, 'https://www.comlaw.gov.au', 'https://www.legislation.gov.au');

-- Annotate to_kind based on the URL pattern
ALTER TABLE valid_extracted_links ADD COLUMN to_kind TEXT;
UPDATE valid_extracted_links SET to_kind='li' WHERE starts_with(url, 'https://www.legislation.gov.au');
UPDATE valid_extracted_links SET to_kind='acts' WHERE contains(url, '/acts/') AND to_kind IS NULL;
UPDATE valid_extracted_links SET to_kind='regulations' WHERE contains(url, '/regs/') AND to_kind IS NULL;
UPDATE valid_extracted_links SET to_kind='policy' WHERE contains(url, '/policy/') AND to_kind IS NULL;
UPDATE valid_extracted_links SET to_kind='amendments' WHERE starts_with(url, '/Amendments/') AND to_kind IS NULL;
UPDATE valid_extracted_links SET to_kind='same_file' WHERE (url IS NULL OR starts_with(url, '/tmp')) AND to_kind IS NULL;
UPDATE valid_extracted_links SET to_kind='external' WHERE to_kind IS NULL;

-- Take links that lead to li or acts or regulations
CREATE TABLE file_inter_connections AS SELECT * FROM valid_extracted_links WHERE to_kind IN ('acts', 'regulations', 'li');
```
This standardizes URLs and categorizes links based on their target type, then selects only links that connect to legislative material.

### 7. Process URL Components
```sql
-- Split link into hashtag and search string
ALTER TABLE file_inter_connections ADD COLUMN hashtag VARCHAR;
ALTER TABLE file_inter_connections ADD COLUMN searchstring VARCHAR;
ALTER TABLE file_inter_connections ADD COLUMN new_url VARCHAR;

-- Extract hashtag component
UPDATE file_inter_connections SET
    new_url = split_part(url, '#', 1),
    hashtag = NULLIF(split_part(url, '#', 2), '');

-- Update URL and extract search string
UPDATE file_inter_connections SET
    new_url = split_part(new_url, '?searchstring=', 1),
    searchstring = NULLIF(url_decode(split_part(new_url, '?searchstring=', 2)), '');

-- URL decode the search string (where not NULL)
UPDATE file_inter_connections SET
    searchstring = url_decode(searchstring)
    WHERE searchstring IS NOT NULL;

-- Replace original URL with the processed URL
ALTER TABLE file_inter_connections DROP COLUMN url;
ALTER TABLE file_inter_connections RENAME new_url TO url;

-- Update URL with processed URL
UPDATE file_inter_connections SET url='https://legend.online.immi.gov.au'||url
WHERE to_kind IN ('acts', 'regulations') AND starts_with(url, '/');
```
This extracts components from complex URLs:
- Hashtags (text after '#')
- Search strings (text after '?searchstring=')
- Normalizes relative URLs by adding the base domain

### 8. Load Legislation Metadata
```sql
-- Load li scraped data
CREATE TABLE li_scraped_data AS (
    SELECT * FROM read_csv('li_scraped_data_01_01_2025.csv')
    UNION ALL SELECT * FROM read_csv('li_scraped_data_07_12_2024_13_12_2024.csv')
    UNION ALL SELECT * FROM read_csv('li_scraped_data_20_12_2024_31_12_2024.csv')
    UNION ALL SELECT * FROM read_csv('li_scraped_data_17_12_2024_19_12_2024.csv')
    UNION ALL SELECT * FROM read_csv('li_scraped_data_14_12_2024_16_12_2024.csv')
    UNION ALL SELECT * FROM read_csv('li_scraped_data_07_02_2025.csv')
);

-- Combine files
CREATE TABLE combined_scraped_data AS (
    SELECT url, text AS title, NULL AS in_force, file_name
    FROM valid_links
    WHERE (contains(url, '/acts/') OR contains(url, '/regs/')) AND NOT contains(file_name, '/legislation/')
    UNION ALL
    SELECT *
    FROM li_scraped_data
);
```
This combines legislation metadata from all time periods into a single comprehensive dataset.

### 9. Create Graph Database Structure
```sql
-- Create graph db table
CREATE TABLE graph_db_data AS SELECT
    fic.file_path AS file_name,
    fic.page_no,
    fic.anchor_text,
    fic.from_kind AS file_kind,
    fic.to_kind AS referenced_kind,
    fic.hashtag, fic.searchstring,
    fic.version AS version,
    fic.url,
    csd1.title AS referenced_title,
    csd1.in_force AS referenced_in_force,
    csd1.file_name AS referenced_file_name,
    csd2.url AS file_url,
    csd2.title AS file_title
FROM
    file_inter_connections fic
LEFT JOIN
    combined_scraped_data csd1
ON
    fic.url = csd1.url AND contains(csd1.file_name, fic.version)
LEFT JOIN
    combined_scraped_data csd2
ON
    fic.file_path = csd2.file_name;
```
This joins the interconnection data with metadata about both source and target files to create a complete graph data structure.

### 10. Generate Graph Database Tables
```sql
-- Generate graph db nodes
CREATE TABLE graph_db_nodes AS
SELECT DISTINCT * FROM (
    SELECT
        referenced_file_name AS file_name,
        referenced_kind AS kind,
        url,
        referenced_title AS title
    FROM graph_db_data
    WHERE referenced_file_name IS NOT NULL
    AND (referenced_in_force OR referenced_in_force IS NULL)
    UNION ALL
    SELECT
        file_name,
        file_kind,
        file_url,
        file_title
    FROM graph_db_data
    WHERE referenced_file_name IS NOT NULL
);

-- Generate graph db references
CREATE TABLE graph_db_references AS SELECT
    file_name,
    referenced_file_name,
    ARRAY_AGG(DISTINCT page_no) AS pages,
    ARRAY_AGG(
        {
            'hashtag': hashtag,
            'searchstring': searchstring,
            'anchor_text': anchor_text, 'page_no': page_no
        }
    ) AS details
    FROM
        graph_db_data
    WHERE
        referenced_file_name IS NOT NULL AND (referenced_in_force IS NULL OR referenced_in_force)
    GROUP BY
        file_name, referenced_file_name;

-- Export data to files
COPY graph_db_nodes TO 'all_graph_db_nodes.csv';
COPY graph_db_references TO 'all_graph_db_references.jsonl' (FORMAT json);
```
This creates the final graph database structure with:
1. `graph_db_nodes`: Unique entries for all documents (both source and referenced)
2. `graph_db_references`: Connections between documents with aggregated details including:
   - Arrays of page numbers where references occur
   - Detailed reference information (hashtags, search strings, anchor text)

The data is then exported to:
- A CSV file containing all nodes
- A JSONL file containing all references

To convert the references JSONL to CSV format, use:
```bash
python -m ragcore.knowledge_graph.references_jsonl_to_csv all_graph_db_references.jsonl all_graph_db_references.csv
```

The final structure supports graph-based analysis of document interconnections across the entire corpus of legal texts.