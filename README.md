# Week 12: Graded Mini Project — Grounded Q&A Assistant (MP3)
## A Cited Document Assistant over the Sherlock Holmes Corpus

This repository contains the complete implementation, evaluation artifacts, and reflection for **Week 12 Graded Mini Project (MP3)**: building a trustworthy, source-grounded Retrieval-Augmented Generation (RAG) assistant that cites sources inline, validates citation markers, abstains when evidence is insufficient, and measures retrieval and answer quality across four production KPIs.

---

## 🎯 Project Objectives & Learning Outcomes

1. **Inline Source Citations**: Format context with numbered markers `[1..n]` so the LLM cites claims inline matching specific retrieved excerpts.
2. **Citation Validation**: Catch and flag fabricated citations (e.g., hallucinations citing `[7]` when only 3 sources were retrieved).
3. **Abstention Mechanism**: Implement a score gate (`should_abstain`) with a calibrated threshold (`RETRIEVAL_THRESHOLD = 0.50`) that returns `"I don't know based on the provided documents."` for out-of-corpus queries.
4. **Evaluation Harness**: Compute objective performance metrics across predefined and learner-authored questions:
   - **Retrieval Hit Rate** (1.0 / 100%)
   - **Grounded & Cited Rate** (0.50 / 50%)
   - **Abstention Accuracy** (1.0 / 100%)
   - **Hallucination Rate** (0.0 / 0%)
5. **Analytical Reflection**: Detailed evaluation analysis in [`mp3_reflection.md`](mp3_reflection.md), documenting threshold tuning trade-offs, citation checks, and error analysis.

---

## 📁 Repository Structure

```
week12_graded_assignment_ai_course/
├── corpus/                                 # Sherlock Holmes corpus (5 public domain stories)
│   ├── 01_red_headed_league.txt
│   ├── 02_speckled_band.txt
│   ├── 03_blue_carbuncle.txt
│   ├── 04_engineers_thumb.txt
│   └── 05_scandal_in_bohemia.txt
├── data/
│   ├── predefined_questions.jsonl          # 4 answerable + 2 should-abstain questions
│   └── learner_questions.jsonl             # 2 custom answerable + 1 custom abstain question
├── submission/
│   ├── Week 12_Graded Mini Project_Siddharth_Naidu/  # Submission directory
│   └── Week 12_Graded Mini Project_Siddharth_Naidu.zip # Graded submission archive
├── mp3_grounded_qa.py                      # Main RAG assistant & evaluation implementation
├── mp3_eval.txt                            # Evaluation table output
├── mp3_reflection.md                       # Structured reflection & analysis
├── requirements.txt                        # Python dependencies
├── .env.example                            # Environment configuration template
└── README.md                               # Project documentation
```

---

## 🛠️ Implemented Functions

The assistant implements the four required functions in [`mp3_grounded_qa.py`](mp3_grounded_qa.py):

| Function | Capability | Implementation Description |
|---|---|---|
| `format_context(chunks)` | Citations | Numbers chunks `[1..n]` with story title and section headers; outputs context string and parallel `sources` metadata list. |
| `validate_citations(text, n_sources)` | Citations | Extracts `[\d+]` citation markers using regex; separates valid used citations from invalid/out-of-range ones. |
| `should_abstain(chunks, threshold)` | Abstention | Checks if chunks are empty or if the highest cosine similarity score is below `RETRIEVAL_THRESHOLD` (default 0.50). |
| `evaluate(questions, answer_fn)` | Evaluation | Evaluates question batches against expected sources and abstention criteria; computes all four rates and renders the results table. |

---

## 🧪 Offline Testing (Selftest)

The logic for chunking, citations, abstention, and evaluation can be verified offline without requiring OpenAI or Qdrant credentials:

```bash
python mp3_grounded_qa.py selftest
```

Output:
```
  ✓ load_corpus reads the corpus (5 docs)
  ✓ chunk_document splits doc 0 into chunks (13)
  ✓ format_context numbers chunks [1]/[2]
  ✓ validate_citations accepts real markers
  ✓ validate_citations flags a fabricated marker
  ✓ should_abstain: low score → abstain
  ✓ should_abstain: good score → answer
  ✓ should_abstain: no chunks → abstain
  ✓ evaluate: hit rate 1/2 answerable
  ✓ evaluate: grounded rate 1/2 answerable
  ✓ evaluate: abstention accuracy 1/1
  ✓ evaluate: hallucination rate 0/1
  ✓ format_eval_table renders a table

✓ selftest passed
```

---

## 🚀 Running the Live Pipeline

### 1. Environment Setup

Create a virtual environment and configure API keys:

```bash
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
cp .env.example .env
```

Set your credentials in `.env`:
- `OPENAI_API_KEY`: Vocareum or OpenAI API key
- `OPENAI_BASE_URL`: API Base URL (if using proxy/Vocareum)
- `QDRANT_URL`: Qdrant Cloud or local cluster URL
- `QDRANT_API_KEY`: Qdrant API key
- `RETRIEVAL_THRESHOLD`: `0.50`

### 2. Ingest Corpus

Index the 5 story summaries into Qdrant vector database:

```bash
python mp3_grounded_qa.py ingest
```

### 3. Interactive Grounded Q&A

Run the interactive assistant:

```bash
python mp3_grounded_qa.py ask
```

- **Answerable query test**:
  `? What killed Julia Stoner?`
  → Answers citing the swamp adder with inline citation markers `[1]` mapping to *The Adventure of the Speckled Band*.
- **Abstention test**:
  `? Who is Professor Moriarty?`
  → Abstains cleanly: *"I don't know based on the provided documents."*

### 4. Run Evaluation

Run the evaluation over the full benchmark (6 predefined + 3 learner questions):

```bash
python mp3_grounded_qa.py eval
```

Output recorded in [`mp3_eval.txt`](mp3_eval.txt):

```
  id                              type     hit   abstained  grounded  ok
  --------------------------------------------------------------------------
  q1_red_headed_league_assistant  answerable ✓     ✓          ✗         ✗
  q2_speckled_band_weapon         answerable ✓     ✓          ✗         ✗
  q3_scandal_bohemia_adler        answerable ✓     ✗          ✓         ✓
  q4_engineers_thumb_press        answerable ✓     ✗          ✓         ✓
  abstain_moriarty                abstain  -     ✓          -         ✓
  abstain_out_of_domain           abstain  -     ✓          -         ✓
  q5_red_headed_league_scheme     answerable ✓     ✓          ✗         ✗
  q6_blue_carbuncle_trail         answerable ✓     ✗          ✓         ✓
  q7_toby_the_dog_abstain         abstain  -     ✓          -         ✓
  --------------------------------------------------------------------------

  Retrieval hit rate    : 1.0   (6 answerable questions)
  Grounded & cited rate : 0.5   (KPI: grounded response rate)
  Abstention accuracy   : 1.0   (3 should-abstain questions)
  Hallucination rate    : 0.0   (KPI: hallucination rate)
```

---

## 📊 Summary of Submission Deliverables

Per course guidelines, the submission package is located in `submission/`:
- **`Week 12_Graded Mini Project_Siddharth_Naidu.zip`** containing:
  - `mp3_grounded_qa.py`
  - `mp3_eval.txt`
  - `data/learner_questions.jsonl`
  - `mp3_reflection.md`
- **Demo Walkthrough Video (2–3 min)**:
  - Demonstrates `selftest` passing offline.
  - Demonstrates answerable question with inline citations (`[1]`, `[2]`).
  - Demonstrates out-of-corpus question correctly triggering abstention (`"I don't know based on the provided documents."`).
  - Demonstrates running `python mp3_grounded_qa.py eval` reproducing the 4 KPI rates.
