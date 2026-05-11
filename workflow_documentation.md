# My Name in History
## Workflow Documentation — HUM Hackathon 2026

*How we traced Estonian family names from 19th-century court records to published authorship*

---

**Authors:** HUM Hackathon 2026 Team  
**Licence:** CC BY 4.0  
**Date:** April 2026  
**Datasets:** Parish Court Records (National Archives of Estonia) · ENB Books Dataset (National Library of Estonia) · ENB Persons Dataset (National Library of Estonia) · Wikidata (Wikimedia Foundation)

**Keywords (content):** genealogy · national awakening · cultural heritage · historical records · Estonian identity

**Keywords (TaDiRAH):** Named entity recognition · Record linkage · Data enrichment · Exploratory analysis · Visualization

**Data types:** Structured tabular (CSV, TSV) · Unstructured text (HTML) · Linked open data (Wikidata)

---

## Overview

This workflow documents how we built a name disambiguation engine that bridges two historical datasets — Estonian parish court records from the 19th century and the Estonian National Bibliography — through the shared axis of personal names. The driving question was deceptively simple: can we trace a family name from a court ledger in 1850 to a title page in 1895? The answer turned out to require a pipeline crossing orthographic history, probabilistic matching, and Wikidata enrichment, before arriving at a public-facing heritage product.

> *"The court records show Estonia before it had a voice. The bibliography shows Estonia finding one."*
> — The core insight that shaped every decision in this project

### Scale of the data

| Dataset | Size | Scope |
|---|---|---|
| Parish court records | 3,990,000 rows | 1821 – 1925 |
| ENB books | 321,000 rows | All periods |
| ENB persons | 117,964 rows | All periods |
| Estonian-era persons (matching pool) | ~14,900 | Born 1750–1950 |
| Average name occurrences per court record | ~7 | — |
| ENB persons with a Wikidata ID | ~60% | — |

---

## Workflow steps

### 1. Understand the datasets and define the research question

**Keywords:** Discovering · Contextualizing · Information retrieval

#### Goal

Before writing a single line of code, we spent time understanding what was actually in each dataset. The parish court records are a pipe-delimited CSV of nearly four million rows, each representing a single court session entry. Names are not in a clean column — they are embedded in HTML-annotated text fields and in a semi-structured `jury` string encoding each participant's role. The `person` column, which one might have hoped would be the simplest route to names, turned out to be entirely empty across the whole file.

The ENB books dataset is a rich TSV with over 300,000 bibliographic entries. Its `creator` field is null in 57% of records, but `contributor` is much better populated. The persons dataset — 117,964 individuals — became the primary matching target, particularly because it already included Wikidata identifiers (`wkp_id`) for 60% of persons, and birth/death year information that would later power era filtering.

The research question crystallised through this inspection: **can we link names appearing in court records to persons in the ENB, and use that link to tell the story of a name moving from legal subject to published author?**

#### Relation to the workflow

This step set the constraints for everything that followed. Knowing that names were in HTML and semi-structured strings — not clean columns — meant the extraction layer would be more complex than a simple lookup. Knowing that the persons dataset had birth years meant we could build a temporal plausibility check into matching. And knowing that 60% of persons had Wikidata IDs opened the door to enrichment with portraits and geographic data that the bibliographic files alone couldn't provide.

---

### 2. Extract names from court records

**Keywords:** Named entity recognition · Preprocessing · Transformation

#### Goal

Names live in two places in the court records, and each requires a different extraction strategy. The `text` field contains a diplomatic transcription of each court session, formatted in HTML. Names have been annotated using `<person>` tags with a `title` attribute. However, the same attribute is also used for place annotations prefixed with `Koht:` (meaning "place") and person annotations prefixed with `Isik:` (meaning "person"). We parsed the HTML using BeautifulSoup, filtered out all location annotations, stripped the `Isik:` prefix where present, and returned the cleaned name strings.

The `jury` field encodes the court composition as a semi-structured string in the format `Name _NormalizedVariant_-Role; Name-Role`. We split on semicolons, extracted the normalised variant from underscores when available, and parsed the role from the trailing hyphen suffix. Common roles included *Peakohtumees* (chief judge), *Kohtumees* (judge), and *Kirjutaja* (clerk).

The `person` column was 100% empty across the entire file. We discarded it entirely and noted this as a known data limitation.

#### Relation to the workflow

This step produced the raw name-occurrence table — every name, in every record, with its court year, county, parish, municipality, case type, and role. It is the foundation on which all matching is built. We accepted some noise (partial names, initials, occasional place names that slipped through) in exchange for high recall, knowing that the confidence scoring in step 5 would down-weight poor matches.

---

### 3. Build the name normalisation layer

**Keywords:** Transformation · Preprocessing · Linguistic analysis

#### Goal

The single biggest technical challenge in this project is orthographic discontinuity. Court records were kept in German administrative language throughout the 19th century, and Estonian names were written in German-influenced spelling. The ENB bibliography uses modern Estonian. The 1921 Name Estonianisation movement added a third layer: German surnames were replaced wholesale with Estonian ones. A single family might appear as *Jürri Bergmann* in an 1855 court record, as *Jüri Bergmann* in an 1890 register, and as *Jüri Mägi* in a 1930 bibliography — three different strings for the same lineage.

We built a two-layer normalisation approach:

**Layer 1 — First-name lookup table (30+ mappings)**

| German form | Estonian form | Type |
|---|---|---|
| Jürri | Jüri | Phonetic |
| Johann | Juhan | Lookup table |
| Hans | Jaan | Lookup table |
| Tönno | Tõnu | Phonetic + table |
| Maddis | Madis | Phonetic |
| Jacob | Jaak | Lookup table |
| Michael | Mihkel | Lookup table |
| Wilhelm | Vilhelm | Phonetic |

**Layer 2 — Phonetic rules applied to all name strings**

- `w` → `v`
- `ö` → `õ`
- `ae` → `ä`
- `oe` → `õ`
- `ue` → `ü`
- Double consonants collapsed (e.g. `dd` → `d`, `rr` → `r`)
- `ph` → `f`
- `ck` → `k`

**Surname transformations (1921 Estonianisation patterns)**

- `Bergmann` → `Mägi` (German *berg* = mountain, Estonian *mägi* = hill)
- `Waldmann` → `Metsamees` (German *wald* = forest, Estonian *mets* = forest)

#### Relation to the workflow

Without normalisation, searching for *Jaan* would miss every record where the same person is called *Johann*, *Hans*, or *Hannes*. The normalisation layer is what makes the engine behave like a knowledgeable human researcher rather than a keyword search. It also created an interesting by-product: the lookup table itself is a small piece of computational linguistic history, encoding decades of German-Estonian administrative practice into a reusable resource.

---

### 4. Build the ENB persons index and aggregate publication statistics

**Keywords:** Indexing · Preprocessing · Aggregation

#### Goal

The ENB persons dataset became our matching target. We applied the same two-layer normalisation to every person's canonical name and known name variant forms, and prepared in-memory lists for fast candidate retrieval using rapidfuzz. We also created a prioritised subset of approximately 14,900 persons: those with a geographic ISO code of `ee` (Estonia) or with birth years between 1750 and 1950.

In parallel, we parsed the ENB books dataset to aggregate publication statistics per person:

- Total publication count
- Earliest and latest publication year
- Most common language of publication
- Most frequent topic keywords
- Fiction vs. non-fiction breakdown
- Most active publication decade

#### Relation to the workflow

This step is the bridge between the two datasets. The publication statistics aggregation transforms a bare person record into a cultural biography — the difference between "Tamm, Jaan" and "Tamm, Jaan, who published 12 works on folk culture and agriculture between 1887 and 1921."

---

### 5. Build and run the name disambiguation engine

**Keywords:** Record linkage · Probabilistic matching · Computational analysis

#### Goal

The core of this project is a multi-signal confidence scoring engine. For every court record name, we retrieve the top 20 candidate persons from the ENB index using rapidfuzz's token sort ratio, then score each candidate against five signals.

**Confidence scoring — signal weights**

| Signal | Weight | Description |
|---|---|---|
| Exact normalised match | 50% | Strongest signal — identical strings after normalisation |
| Fuzzy string match | 25% | rapidfuzz token sort ratio on normalised names |
| Phonetic normalisation match | 15% | Catches spelling variants that fuzzy matching misses |
| Name variant form match | 5% | Fuzzy match against ENB `name_varform` field |
| Estonian geography bonus | 5% | +bonus if Wikidata confirms person is Estonian |

**Era penalty**

If the court year falls outside the matched person's plausible lifespan — defined as birth year + 10 at the earliest, birth year + 90 at the latest — the confidence score is multiplied by **0.3**. This eliminates a large proportion of false matches from common names appearing across many decades.

**Wikidata post-match boosts**

- +5% if person's birthplace is confirmed as Estonian
- +3% if person has more than 5 Wikipedia sitelinks

**Confidence thresholds used in the product**

| Score | Interpretation |
|---|---|
| ≥ 70% | High confidence — surfaced as primary match |
| 50–70% | Probable — shown with confidence label |
| < 50% | Weak — shown but flagged as uncertain |

**Efficiency note:** To make the pipeline feasible at scale, we deduplicated the name-occurrence table by (normalised name × decade) before running the engine. The same name in the same decade is disambiguated once, not thousands of times, and the result is joined back to all matching records.

#### Relation to the workflow

The disambiguation engine is what makes this project technically original. It is a transparent, weighted scoring function that acknowledges uncertainty and communicates it to the user through the confidence score. We made an explicit decision to show confidence to the user rather than hide it, because heritage products that present uncertain matches as definitive facts erode trust and can mislead genealogical researchers.

---

### 6. Enrich matched persons via Wikidata

**Keywords:** Data enrichment · Linked open data · API integration

#### Goal

For every matched person with a Wikidata ID (`wkp_id`), we queried the Wikidata SPARQL endpoint in batches of 50 to fetch:

- Full birth and death dates (day, month, year — not just year)
- Birth place name
- Birth place geographic coordinates (latitude and longitude)
- Portrait image URLs from Wikimedia Commons
- Number of Wikipedia language editions covering this person (cultural fame signal)

All Wikidata results were cached to a CSV file on Google Drive to avoid re-querying on every session restart. The Wikidata layer is entirely dependent on the ENB persons dataset already having `wkp_id` populated — approximately 60% of persons had this, giving us a good hit rate without requiring any additional entity resolution.

#### Relation to the workflow

Wikidata transforms the product from a data-linking exercise into a genuine heritage experience. The difference between seeing "Tamm, Jaan, born 1847" and seeing a photograph of Jaan Tamm alongside "born 14 March 1847 in Tartu — documented in 23 Wikipedia editions worldwide" is emotional, not just informational. It is the layer that makes a user feel they have found something rather than retrieved something.

---

### 7. Produce the combined output datasets

**Keywords:** Aggregation · Publishing · Disseminating

#### Goal

The pipeline produces four output files, each serving a different audience.

| File | Description | Size |
|---|---|---|
| `name_lookup_table.csv` | Every unique court name with its best ENB match and confidence score | Moderate |
| `court_records_enriched.csv` | All 4M name occurrences with ENB metadata joined in | Large |
| `slim_enriched.csv` | 34-column product-ready dataset: court fields + ENB match + Wikidata | Large |
| `wikidata_enrichment.csv` | Cached Wikidata results: portraits, coordinates, sitelinks | Small |

The slim combined dataset (`slim_enriched.csv`) is self-contained enough that someone with no knowledge of the pipeline can open it and immediately understand what they are looking at. Every column has a meaningful name, every ID field is paired with its human-readable equivalent, and the match confidence is always present.

**Key statistics of the final output:**

- 4,097,046 rows in the enriched dataset
- 34 columns in `slim_enriched.csv`
- 3 source datasets merged through one pipeline

#### Relation to the workflow

The slim_enriched.csv is a resource that did not exist before this project: a file linking 4 million Estonian legal records to bibliographic persons via confidence-scored name matches, enriched with Wikidata geographic and biographical data. It is ready for further research without requiring the raw datasets or the full pipeline to be re-run.

---

### 8. Analyse patterns — what the data reveals about the National Awakening

**Keywords:** Exploratory analysis · Visualization · Interpretation

#### Goal

With the combined dataset in hand, we looked beyond individual name matches and asked questions about the dataset as a whole. Two patterns stood out immediately.

**The 20–30 year gap.** Court records peak dramatically in the 1870s and 1880s, while ENB publications begin accelerating from the 1900s onward. The gap between these two curves corresponds almost exactly to the Estonian National Awakening. We could see the Awakening in the data.

**Geographic convergence.** Tartu county dominates both datasets — it has the most court records and the most bibliography-linked persons. Tartu was the seat of the university and the intellectual engine of the Awakening. The two datasets do not just overlap in time; they converge in space.

**Thematic continuity.** The top case types in court records — loan disputes, land and buildings disputes, property crimes — are the anxieties of a peasant economy. These are precisely the themes that early Estonian-language books began to address: land rights, agricultural practice, family law.

**Records by decade (sample of 100,000 records):**

| Decade | Court records | ENB publications |
|---|---|---|
| 1820s | 1,021 | ~26 |
| 1840s | 2,067 | ~56 |
| 1860s | 14,403 | ~127 |
| 1870s | 30,542 | ~142 |
| 1880s | 39,517 | ~178 |
| 1900s | 1,425 | ~550 |
| 1920s | — | ~898 |
| 1930s | — | ~1,290 |

#### Relation to the workflow

This step transformed the pipeline from a technical exercise into a historical argument. The patterns we found suggest that the National Awakening was not a spontaneous cultural flowering but a response to accumulated legal and economic pressure visible in court record data — a finding that connects this hackathon product to genuine scholarly questions.

---

### 9. Build the product — My Name in History

**Keywords:** Visualization · Prototyping · Disseminating

#### Goal

The final step was turning the dataset into something a non-specialist could use and feel something from. We built a Gradio web application with the following components:

- **Name search** — word-boundary matching across 4 million records (prevents partial hits: "Lee" does not match "Leenik")
- **Court records summary** — count, year range, county, parish, role breakdown, most common case types
- **Historical arc** — Plotly timeline with court appearances below a horizontal line, publication dates above
- **ENB biography** — matched author with full birth/death dates, profession, publication count, languages, topic keywords
- **Narrative closer** — a paragraph generated dynamically from real data
- **Interactive map** — Folium map with county circles sized by record frequency and a birthplace pin from Wikidata
- **Wikidata portrait** — photograph of the matched person where available
- **Surprise Me button** — draws from a pre-computed list of names with strong data behind them for reliable live demonstrations

**Technical stack:** Python · pandas · rapidfuzz · BeautifulSoup · Wikidata SPARQL · Gradio · Plotly · Folium · Google Colab + Drive

#### Relation to the workflow

The product is the proof that the pipeline worked. It is also the argument that this kind of digital humanities infrastructure — bridging legal records, bibliographic data, and linked open data — can produce experiences that feel personally meaningful to the people it is built for.

---

### 10. Document and reflect

**Keywords:** Publishing · Contextualizing · Disseminating

#### Goal

Throughout the project we maintained documentation recording dataset descriptions, extraction methods, normalisation rules, matching signals, output schemas, known limitations, and planned enhancements.

**Known limitations:**

- Confidence scores are probabilistic — high confidence does not mean certain
- The German-to-Estonian name mapping is not exhaustive; rarer dialectal forms may be missed
- The `person` column in court records is 100% empty, limiting extraction routes
- The disambiguation loop (Cell 12) took several hours to run and would require vectorisation to be practical at larger scale
- Wikidata coverage depends on ENB persons having `wkp_id` populated — approximately 40% of persons have no Wikidata link

**Planned enhancements:**

- Integrate 1921 Name Estonianisation records as a third dataset for higher-precision surname mapping
- Build a social network graph from names co-occurring in the same court record
- Parish-level "cultural biography" template — the full story of one community from disputes to books
- Lag map of Estonia: how many years passed between a name appearing in court records and first appearing in print, by county
- Geographic co-occurrence as a disambiguation signal: same parish = higher confidence boost

#### Relation to the workflow

Documentation is not a postscript — it is part of the research output. The slim_enriched.csv dataset has scholarly value beyond this hackathon, and for any researcher who wants to use it they need to understand what decisions were made, where the uncertainty lies, and what the known failure modes are.

---

*My Name in History · HUM Hackathon 2026 · Data: National Archives of Estonia · National Library of Estonia · Wikidata · CC BY 4.0*
