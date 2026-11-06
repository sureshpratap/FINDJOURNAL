# JournalMatch: find authentic Scopus & SCIE journals for your manuscript

Paste a manuscript **title and/or abstract**. JournalMatch finds the most similar published papers
and ranks the journals that publish them, as [JANE](https://jane.biosemantics.org/) does. It then
**verifies every suggested journal against the official Scopus and Web of Science (SCIE) lists**,
so you are not steered towards predatory, discontinued or hijacked titles.

![Journal suggestions](docs/screenshot-results.png)

*(Screenshot uses demo data with invented journal names.)*

## Features

| | |
|---|---|
| **Find journals** | Ranked by similarity to your text. Each journal shows a 0–100 match score, the number of similar articles, and the articles themselves. |
| **Verified indexing badges** | ✓ Scopus (active) · ✓ SCIE · ESCI (not SCIE) · ⚠ Scopus discontinued · Not found. Matching uses the ISSN and eISSN from the official lists. |
| **Filters** | Scopus or SCIE · Scopus only · SCIE only · both · show all. Article time window (3/5/10 years/any). Open-access preference. |
| **Journal metrics** | h-index, 2-yr mean citedness (an impact-factor-like figure from OpenAlex, *not* the official JIF), APC, DOAJ status. |
| **Find articles / authors** | The 50 most similar papers, and the researchers closest to your topic (useful when a journal asks you to suggest reviewers). |
| **Verify a journal** | Enter any ISSN to check Scopus/SCIE status, including discontinued titles. Links go to the official Scopus and Master Journal List pages. |
| **Anti-hijacking guidance** | Users are pointed to the official index pages rather than search-engine results. |

## How it works

```
title + abstract
   │
   ├─► keyword extraction ──► OpenAlex keyword search  (≤100 papers) ─┐
   └─► OpenAlex semantic (embedding) search            (≤50 papers)  ─┤
                                                                      ▼
                  TF-IDF cosine similarity (60%) + semantic score (40%) per paper
                                                                      ▼
            sum similarity of the top-100 papers per journal  ──►  journal score
                                                                      ▼
     ISSN lookup in local index built from official Scopus + Clarivate lists
                                                                      ▼
                         filter (Scopus / SCIE / both)  ──►  ranked suggestions
```

* **Article data:** [OpenAlex](https://openalex.org), an open (CC0) catalogue of 250M+ works across *all*
  disciplines. JANE uses PubMed, which covers biomedicine only, so OpenAlex suits engineering and other fields.
* **Indexing data:** the Scopus Source Title List and the Clarivate Master Journal List, which
  you download and load yourself. See [`data/README.md`](data/README.md).

## Quick start

```bash
git clone https://github.com/<you>/journalmatch.git
cd journalmatch
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env          # then paste your free OpenAlex key into OPENALEX_API_KEY
# download the official lists into data/raw/ (see data/README.md), then:
python scripts/build_index.py \
    --scopus data/raw/ext_list_October_2026.xlsx \
    --wos SCIE=data/raw/wos-core_SCIE.csv

uvicorn app.main:app --reload
# open http://localhost:8000
```

**OpenAlex API key:** since 13 Feb 2026 OpenAlex requires a free key. Get one at
<https://openalex.org/settings/api>. The free tier gives US$1 of usage per day. One manuscript
search costs about US$0.011 (keyword search + semantic search + one journal lookup), so you get
**about 90 searches/day**. Set `ENABLE_SEMANTIC_SEARCH=false` to raise that to about 900/day at
some cost in match quality. Identical searches are cached for 6 hours and don't use credits.

The key stays on the server. The browser never sees it.

## Deploy

**Docker**

```bash
docker compose up -d --build       # mounts ./data so the index can be rebuilt without rebuilding the image
```

**Render / Railway / Fly.io / any VPS:** run
`uvicorn app.main:app --host 0.0.0.0 --port $PORT`, set `OPENALEX_API_KEY`, and give the
service a persistent disk or volume for `data/journal_index.json`.

GitHub Pages cannot host this app, because it needs a server to keep the API key secret.

## API

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/suggest` | Body: `{title, abstract, years, open_access: any\|only\|exclude, indexing: any\|scopus\|scie\|both\|all, semantic}` → journals, articles, authors |
| `GET` | `/api/journal/{issn}` | Indexing status of one ISSN |
| `GET` | `/api/status` | Index size, build date, configuration |
| `GET` | `/docs` | Interactive OpenAPI docs |

## Configuration (`.env`)

| Variable | Default | |
|---|---|---|
| `OPENALEX_API_KEY` | – | Required for real use |
| `CONTACT_EMAIL` | – | Optional contact address sent to OpenAlex |
| `ENABLE_SEMANTIC_SEARCH` | `true` | Embedding search (10× cost of keyword search) |
| `JOURNAL_INDEX_PATH` | `data/journal_index.json` | Built index |
| `CACHE_TTL_SECONDS` | `21600` | Result cache lifetime |
| `RATE_LIMIT_PER_MINUTE` | `20` | Uncached searches per IP per minute |

## Project layout

```
app/
  main.py          FastAPI app and the search pipeline
  openalex.py      OpenAlex client (keyword, semantic, sources)
  text.py          keyword extraction, abstract rebuild, TF-IDF similarity
  ranking.py       article scoring, journal and author aggregation
  index_store.py   Scopus / WoS index lookups and filters
  issn.py          ISSN normalisation and check-digit validation
  static/          single-page frontend (HTML/CSS/JS, no build step)
scripts/build_index.py   builds the index from the official downloads
tests/                   pytest suite (OpenAlex mocked; runs offline)
```

## Development

```bash
pip install -r requirements-dev.txt
pytest -q && ruff check .
```

CI runs both checks on every push (`.github/workflows/ci.yml`).

## Limitations & responsible use

* Suggestions show where *similar work* has been published. They do not predict acceptance.
  Always read the journal's aims & scope.
* Indexing status is only as fresh as the lists you loaded. Rebuild the index monthly.
* A genuine ISSN does not prove a website is genuine. Hijacked journals clone real titles. Reach a
  journal through its Scopus or Master Journal List page.
* 2-yr mean citedness and h-index come from OpenAlex. They are not Clarivate's JIF or Elsevier's CiteScore.

## Acknowledgements

Inspired by JANE (Schuemie & Kors, *Bioinformatics* 2008). Article data from OpenAlex (CC0).

## License

MIT. See [LICENSE](LICENSE). The Scopus and Web of Science lists are **not** covered by this
licence and must not be committed to the repository.
