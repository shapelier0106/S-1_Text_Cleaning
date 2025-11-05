# S-1 Text Cleaning & Section Extraction

This document describes the pipeline used to extract target sections from SEC Form S-1 filings, transform HTML-like sources into clean TXT, and filter climate-related sentences for labeling and modeling.

---

## 1. Target sections extractions

1.1 Extract the first `<TEXT> ... </TEXT>` block as the main body in Form S-1.  
1.2 Capture the Table of Contents (TOC): find a standalone “Table of Contents” line, then take the first `<TABLE>…</TABLE>` after it; if multiple TOC tables are adjacent, keep swallowing them until they stop.  
1.3 Parse TOC text into entries: section title, corresponding page number (start page), and the next section’s start page (as the end page).  
1.4 Use standalone numeric lines as page markers to split the full text into per-page segments and build a map from printed page numbers to indices in the page list.  
1.5 For each target TOC entry, jump to its start page and locate the section heading on that page.  
1.6 Extract the section content:
   1) From the end of the heading on the start page, collect body text.  
   2) If there’s no “next section,” keep collecting to the end of the file.  
   3) If there is a “next section,” add all full pages between start and end pages; on the end page, cut off at the next section’s heading if found.  
   4) Merge everything collected as the section’s content.  

1.7 Iterate over all target TOC entries and gather all target sections.

---

## 2. Transforming HTML Code to TXT

2.1 Extract the first `<TEXT>…</TEXT>` block from the source as the main content.  
2.2 Table detection: find `<TABLE>` tags. If the 300 characters before a table contain a standalone line “Table of Contents,” mark it as a TOC table; otherwise mark it as a normal table.  
2.3 For TOC tables: if following tables are separated only by whitespace/layout tags, merge them into one TOC. Reformat the merged table into a fixed-width text directory.  
2.4 For normal tables: parse rows/cells. If ≥30% of cells look numeric (money, percentages, bps, thousands separators), treat it as a data table and drop it; otherwise, join each row into a paragraph.  
2.5 HTML cleanup:
   1) remove `<HEAD>…</HEAD>`, `<IMG>`, and extra `<HR>` (turn to newlines), etc.;  
   2) convert all line breaks and block-level closing tags to `\n`;  
   3) remove all other tags except `<PAGE>`;  
   4) decode HTML entities;  
   5) remove `<PAGE>` tags, delete lines that are exactly “Table of Contents,” and collapse excessive blank lines.

---

## 3. Filtering out climate-related sentences

3.1 Within each section, split text into “sentences,” using numbers/list bullets and end punctuation as breakpoints.  
3.2 Normalize characters, then compute per-sentence stats: number of lines, word count, unique-word count, and letter ratio.  
3.3 Keep only sentences that meet thresholds for lines, words, unique-word share, and letter ratio.  
3.4 Do keyword matching with a tolerant regex built from your keyword list, supporting space/hyphen/underscore variants.  
3.5 Build a labeling pool, then split it into train/val/test sets in a 70/15/15 ratio.
