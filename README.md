# My Name in History

**Tracing Estonian family names from 19th-century court records to published authorship**

Built at HUM Hackathon 2026 · CC BY 4.0

---

## What this project does

This project builds a name disambiguation engine that links two major Estonian 
cultural heritage datasets — parish court records from the 19th century and the 
Estonian National Bibliography — through the shared axis of personal names.

A user enters a family name and receives a narrative arc: how many times it 
appeared in court records, in which counties and decades, and whether it can be 
matched to a person who later published books in the Estonian National Bibliography. 
The crossing point between the two datasets — the moment a name moves from legal 
subject to published author — is the story of the Estonian National Awakening, 
visible in data.

---

## Repository structure
notebook/         Google Colab notebooks 
docs/             Workflow documentation, pipeline diagram
outputs/          Lookup table and Wikidata enrichment cache (small files)
presentation/     Hackathon presentation (PPTX)
data/             Instructions for accessing the raw datasets

## How to run

### Step 1 — Get the data
The three input datasets are not in this repository. See `data/README.md` for 
access instructions and links.

### Step 2 — Run the pipeline
Open `notebooks/01_pipeline.ipynb` in Google Colab. Mount your Google Drive, 
update the file paths in Cell 4, and run all cells top to bottom. Cell 12 
(disambiguation) takes several hours — plan accordingly.

### Step 3 — Run the demo app
Once the pipeline has produced `slim_enriched.csv`, open 
`notebooks/02_demo_app.ipynb` in Google Colab and run both cells. 
The second cell launches a Gradio app and prints a public URL.

---

## Datasets used

| Dataset | Source | Licence | Access |
|---|---|---|---|
| Parish Court Records | National Archives of Estonia / KIRMUS | Open | https://liilia.kirmus.ee/s/SK4gTCBxH9QGqWn |
| ENB Books | National Library of Estonia | CC0 1.0 | https://doi.org/10.5281/zenodo.8228794 |
| ENB Persons | National Library of Estonia | CC0 1.0 | https://doi.org/10.5281/zenodo.8228794 |
| Wikidata | Wikimedia Foundation | CC0 1.0 | https://www.wikidata.org/ |

The large output datasets (slim_enriched.csv, court_records_enriched.csv) are 
hosted on Google Drive:
**[Download outputs →](PASTE YOUR GOOGLE DRIVE FOLDER LINK HERE)**

---

## Technical stack

Python · pandas · rapidfuzz · BeautifulSoup4 · Wikidata SPARQL ·
Gradio · Plotly · Folium · Google Colab · Google Drive

---

## Key outputs

- `slim_enriched.csv` — 4M rows, 34 columns: court record fields + ENB match + 
  Wikidata biographical data. The primary research contribution.
- `name_lookup_table.csv` — every unique court name with its best ENB match 
  and confidence score
- `wikidata_enrichment.csv` — cached Wikidata results: portraits, coordinates, 
  Wikipedia sitelinks

---

## Workflow documentation

Full methodology documentation is in `docs/workflow_documentation.md`, 
following the HUMAL data lab workflow format. It covers dataset descriptions, 
extraction methods, normalisation rules, disambiguation signals, output schemas, 
known limitations, and planned enhancements.

---

## Known limitations

- Confidence scores are probabilistic — high confidence does not mean certain
- The German→Estonian name mapping covers common transformations but is not 
  exhaustive; rarer dialectal forms may be missed
- The disambiguation loop takes several hours at full scale
- The Wikidata layer requires Wikimedia Commons URL reformatting to render 
  portraits correctly in browsers
- Integration of the 1921 Estonianisation database (ra.ee/apps/onomastika/) 
  is planned but not yet implemented

---

## Planned enhancements

- Integrate the National Archives onomastics database for historically attested 
  German→Estonian surname mappings
- Social network graph of names co-occurring in the same court record
- Parish-level cultural biography template
- Lag map of Estonia: years between court appearance and first publication, 
  by county

---

## Citation

If you use this project or its outputs, please cite as:

> Bhumika Bhattacharyya. *My Name in History: Tracing Estonian Family Names 
> from Parish Court Records to Published Authorship.* HUM Hackathon 2026. 
> https://github.com/eurydice-1/My-Name-in-History. CC BY 4.0.

---

## Contact

Issues and pull requests welcome. For institutional enquiries about deploying 
this as a public tool, contact via GitHub Issues.

# Data

The raw datasets used in this project are not stored in this repository.
They are large files owned by their respective institutions and available 
under open licences at the links below.

## How to get the data

**1. Parish Court Records**
Download from the KIRMUS crowdsourcing project:
https://liilia.kirmus.ee/s/SK4gTCBxH9QGqWn
Filename to expect: `parish_courts_full_dataset.csv`
Format: CSV, pipe (|) delimited, UTF-8
Size: ~400MB

**2. ENB Books Dataset**
Download from Zenodo:
https://doi.org/10.5281/zenodo.8228794
Filename to expect: `enb_books.tsv`
Format: TSV, tab delimited, UTF-8
Size: ~240MB

**3. ENB Persons Dataset**
Download from Zenodo (same DOI as above):
https://doi.org/10.5281/zenodo.8228794
Filename to expect: `persons.tsv`
Format: TSV, tab delimited, UTF-8
Size: ~21MB

## Where to put them

Upload all three files to a folder in your Google Drive. Then update the 
file paths in Cell 4 of `notebooks/01_pipeline.ipynb` to point to that folder.

## Large output files

The processed output files are too large for GitHub. Download them here:


Files available:
- `slim_enriched.csv` — 4M rows, 34 columns, ~500MB [https://drive.google.com/file/d/154p6eluO3TXF4_XxBo4FRKnnQbOiYrno/view?usp=drive_link]
- `court_records_enriched.csv` — full enriched dataset [https://drive.google.com/file/d/1mwEwOx419GKe5shyvbxQhS4OU9XAYU0a/view?usp=drive_link]
- `name_lookup_table.csv` [https://drive.google.com/file/d/1aqFY9vMfD07kBok-ssmGoV_tWE448VXK/view?usp=drive_link]
- `wikidata_enrichment.csv` [https://drive.google.com/file/d/1xGbE54TlbhCYNgeExkaMQ20v_crcLOif/view?usp=drive_link]
