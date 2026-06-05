# Assignment 01 — Intelligent Document Analyzer


---

## Features

| Feature | Method | Libraries |
|---|---|---|
| **Summarization** | LexRank (TF-IDF PageRank) + Frequency-based fallback | `nltk`, `scikit-learn`, `numpy` |
| **Heading Extraction** | Font-size heuristics (PDF) + Style detection (DOCX) + Regex patterns | `PyMuPDF`, `python-docx` |
| **Semantic Search** | TF-IDF vectorization + Cosine similarity | `scikit-learn` |
| **UI** | Gradio — works locally and in Google Colab | `gradio` |

---


## Quick Start

**1. Install dependencies**

```bash
pip install streamlit pymupdf python-docx nltk scikit-learn numpy
```

**2. Download NLTK data** (first run only)

```python
import nltk
nltk.download('punkt')
nltk.download('punkt_tab')
nltk.download('stopwords')
```

**3. Launch**

```bash
streamlit run app.py
```

Open `http://localhost:8501` in your browser.

---

## Notebook Structure

The Colab notebook has **5 cells** — run them in order:

| Cell | Purpose |
|---|---|
| **1 — Install** | Installs all 6 packages via `subprocess` (reliable in Colab) |
| **2 — Imports** | Every `import` statement in one place; also downloads NLTK data |
| **3 — Logic** | All classes and functions; no imports here — uses Cell 2's namespace |
| **4 — UI** | Gradio interface + `demo.launch(share=True)` |

Keeping all imports in a single cell (Cell 2) prevents `ModuleNotFoundError` crashes that happen when imports are scattered across cells that run before packages are fully installed.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Gradio UI  /  Streamlit UI                 │
│   ┌──────────────┐  ┌───────────────┐  ┌────────────┐  │
│   │  Tab 1       │  │  Tab 2        │  │  Tab 3     │  │
│   │  Summarize   │  │  ToC / Index  │  │  Search    │  │
│   └──────┬───────┘  └──────┬────────┘  └─────┬──────┘  │
└──────────┼─────────────────┼─────────────────┼──────────┘
           │                 │                 │
     summarize()      extract_headings()     TFIDFSearcher
           │                 │                 │
           └─────────────────┴─────────────────┘
                             │
                          extract()
                    ┌────────┴────────┐
               extract_pdf()    extract_docx()
               (PyMuPDF)        (python-docx)
```

---

## Summarization

Two approaches are implemented, tried in order:

**LexRank — default**

Graph-based extractive summarization implemented from scratch using `sklearn` and `numpy` (no `sumy` dependency — compatible with Python 3.12+).

1. Vectorize all sentences with TF-IDF
2. Compute pairwise cosine similarity → sentence graph
3. Row-normalize into a Markov transition matrix
4. Run PageRank (power iteration, 100 steps, damping = 0.85)
5. Select top-N highest-scoring sentences in document order

**Frequency-based — fallback**

1. Tokenize text and remove stopwords
2. Build a normalized term-frequency table
3. Score each sentence by the sum of its token frequencies
4. Select top-N sentences in document order

Both methods enforce the assignment's **≥ 10% length requirement**. If the primary method produces a shorter summary, the frequency-based method re-runs at 20%.

---

## Heading Detection

**PDFs** — uses PyMuPDF font metadata per span:

| Condition | Level |
|---|---|
| Average font size ≥ 18 pt, or bold + size ≥ 16 pt | H1 |
| Average font size ≥ 14 pt, or bold + size ≥ 13 pt | H2 |
| Bold + line length < 80 chars | H3 |
| Numbered pattern `1.`, `1.1.`, `1.2.3` | H1 / H2 / H3 |
| ALL CAPS, length 4–79 chars | H1 |

**DOCX** — reads native paragraph style names (`Heading 1`, `Heading 2`, `Heading 3`, `Title`, `Subtitle`) first, then falls back to the same regex patterns above.

Detected headings are assigned hierarchical outline numbers (1, 1.1, 1.2.3, …) and rendered as a structured table of contents.

---

## Semantic Search

1. All non-heading paragraphs are split into sentences with NLTK
2. `TfidfVectorizer` (unigrams + bigrams, sublinear TF, English stopwords) is fitted on all sentences
3. The user's query is transformed with the same vectorizer
4. Cosine similarity ranks every sentence against the query
5. Top-K results are returned with a similarity score (%) and page number

---

## Dependencies

| Package | pip name | Purpose |
|---|---|---|
| `fitz` | `pymupdf` | PDF text + font metadata extraction |
| `docx` | `python-docx` | DOCX paragraph + style extraction |
| `nltk` | `nltk` | Sentence tokenization, stopwords |
| `sklearn` | `scikit-learn` | TF-IDF, cosine similarity |
| `numpy` | `numpy` | Matrix ops for LexRank PageRank |
| `gradio` | `gradio` | Web UI (Colab-compatible) |

> **Note on `sumy`:** An earlier version used `sumy` for LexRank. It was removed because `sumy` depends on Python's `imp` module, which was deleted in Python 3.12, causing a `ModuleNotFoundError` in Colab. LexRank is now implemented directly using `sklearn` and `numpy`.

---

## Challenges

- **PDF heading detection** is heuristic-only — PDFs have no semantic structure, so font size and bold flags vary widely across documents, requiring a multi-tier fallback strategy.
- **Summary length guarantee** — LexRank can under-select on very short or repetitive documents; the frequency-based fallback and a 20% expansion pass ensure the 10% floor is always met.
- **Python 3.12 compatibility** — `sumy`'s use of the removed `imp` module required replacing it with a from-scratch LexRank implementation.
- **Colab import ordering** — Scattering imports across cells caused crashes when cells ran before packages were fully installed; consolidating all imports into one cell after the install cell fixed this.

---

## Evaluation Criteria Coverage

| Criteria | Weight | Implementation |
|---|---|---|
| Document Summarization Quality | 25% | LexRank (PageRank on TF-IDF sentence graph) + frequency fallback, ≥10% ratio enforced |
| Heading & Subheading Extraction | 20% | Font metadata (PDF) + style names (DOCX) + regex patterns; hierarchical numbering |
| Semantic Search Quality | 25% | TF-IDF bigram vectors + cosine similarity; score % + page number displayed |
| Code Quality & Architecture | 15% | Modular design, dataclasses, clear separation of concerns |
| UI/UX | 10% | Gradio tabs; copy buttons; live score bars; stats displayed per result |
| Creativity / Bonus | 5% | Dual summarization methods; graceful fallbacks; Colab + local support |
