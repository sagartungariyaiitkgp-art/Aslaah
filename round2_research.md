# Round 2 Research Compilation — Valmo / DICE Challenge S3

> **Important note:** The only PDF in `C:\Users\SAGAR\Desktop\meesho` is **`DICE Challenge S3  Valmo Case studies.pdf`** — this is a **Valmo (healthcare claims)** case study, **not Meesho**. All analysis below follows the Valmo problem statement. If the actual round‑2 brief is a different Meesho document, replace the domain examples while keeping the architecture.

---

## 1. Problem Summary (from Valmo PDF)

| Pain Point | Current State | Target for Prototype |
|------------|---------------|----------------------|
| Pre‑authorization | 2‑5 days, manual | <15 min triage, same‑day decision |
| Claim settlement | 30‑45 days | First‑pass approval ↑ 20‑30 % |
| Denial rate | 15‑20 % (doc/coding mismatch) | ≤8 % (target for demo) |
| Hospital rework | High, staff burnout | Missing‑doc checklist + auto‑suggestions |
| Patient anxiety | No visibility | Real‑time WhatsApp/SMS/email updates |
| Payer ops cost | High manual adjudication | Rule‑based + RAG decision engine with audit trail |

**Required capabilities** (explicit in brief):
1. Automate pre‑auth
2. Reduce denials
3. Improve patient communication
4. Operational analytics
5. Human‑in‑the‑loop, ethical/regulatory guardrails

---

## 2. Winning Concept — “ArogyaFlow / Valmo Claims Copilot”

**Narrow, high‑impact workflow** (not a generic chatbot):

```
Upload scanned pre‑auth packet
      ↓
Multilingual OCR + layout parsing (PaddleOCR‑VL‑1.6 / PP‑OCRv6)
      ↓
Structured extraction → ABDM/NHCX FHIR Claim Bundle
      ↓
HBP/STG rules engine + RAG over NHA guidelines
      ↓
ICD‑10 / HBP code suggestions with evidence spans
      ↓
Pre‑auth readiness decision (approve / clarify / reject) + citations + missing‑doc list
      ↓
Human review (approve/edit) → audit log
      ↓
Patient comms (EN/HI/Hinglish templates) + denial‑risk/root‑cause dashboard
```

**Why this scores:**
- Solves the *exact* quantified pains (TAT, denials, patient comms)
- Demonstrates **India‑specific** standards (NHA HBP, STG, NHCX FHIR)
- Shows **safety**: every AI output grounded, cited, human‑approved
- End‑to‑end demo in one synthetic case (cataract / normal delivery / diabetic foot)
- Clear business metrics for pilot pitch

---

## 3. Data Strategy — What’s Actually Usable

| Dataset | Access | What It Gives | Use in Prototype |
|---------|--------|---------------|------------------|
| **NHA HBP 2.2** (`nha.gov.in/img/resources/HBP-2.2-manual.pdf`) | Public | Package list, rates, exclusions, pre‑auth rules | Core rules engine + RAG corpus |
| **NHA STG Manual** (`nha.gov.in/img/pmjay-files/STG-Manual-Booklet-final.pdf`) | Public | Disease‑specific treatment pathways | Guideline retrieval for RAG |
| **NHCX FHIR IG & Examples** (`nrces.in/ndhm/fhir/r4/…`) | Public | Claim, Coverage, ClaimResponse, Bundle profiles, JSON examples | FHIR normalization target & validation |
| **Synthea** (`github.com/synthetichealth/synthea`) | Open (Apache‑2.0) | Synthetic patients (FHIR R4, CCDA, CSV) — 100 % privacy‑free | Demo corpus, synthetic pre‑auth packets |
| **MIMIC‑IV‑Note** (PhysioNet, credentialed) | DUA + CITI training | 357k discharge summaries, 2.47M radiology reports | *Only if team already has access* — coding evidence eval |
| **MDACE** (`github.com/3mcloud/MDACE`) | Requires MIMIC access | 302 inpatient + 52 pro‑fee charts with coder‑annotated evidence spans | Gold‑standard coding‑evidence benchmark |
| **PMJAY Dashboard / data.gov.in** | Public aggregate | Fund releases, admissions, top procedures | Analytics dashboard seed data |
| **CMS SynPUF** (synthetic US Medicare) | Public | Claims‑structure reference | Fallback if NHCX examples insufficient |

**Recommendation:** Build the demo **entirely on NHA public docs + Synthea**. Use MIMIC/MDACE only for internal evaluation if credentials exist — do not ship them.

---

## 4. Model & Tooling Choices (2026‑current)

| Layer | Recommended | Why | Fallback |
|-------|-------------|-----|----------|
| **OCR / Layout** | **PaddleOCR‑VL‑1.6** (0.9B VLM, 96.3 % OmniDocBench) + PP‑OCRv6 (50 langs, 34.5M params) | SOTA open‑source doc parsing, Hindi/Devanagari, tables, seals, formulas | `mindee/doctr` |
| **Primary LLM** | **Qwen3‑8B‑Instruct** (Apache‑2.0, 33k/131k ctx, 4‑bit ≈5 GB VRAM) via Ollama / vLLM | Strong multilingual, tool‑calling, structured JSON, permissive licence | Qwen2.5‑7B‑Instruct / Llama‑3.1‑8B‑Instruct |
| **Hindi‑specific reasoning** | **HiMed‑8B** (research, LLaMA‑3.1‑8B base) | Hindi medical benchmarks ↑ | Not production‑ready; use Qwen3 + Hindi prompts |
| **Embeddings** | **BAAI/bge‑m3** (multilingual, 100+ langs, hybrid dense+sparse) | Proven on MTEB, works with Qdrant/Chroma | `intfloat/multilingual‑e5‑large` |
| **Vector DB** | **Qdrant** (local, filterable) or **Chroma** (zero‑config) | Fast hybrid search, metadata filtering | FAISS (if no metadata needed) |
| **Rules / Validation** | JSON policy + Pydantic models + `jsonschema` / `great_expectations` | Deterministic, auditable, versionable | — |
| **FHIR** | `fhir.resources` (Python) + NHCX example bundles | Direct mapping to ABDM/NHCX profiles | HAPI FHIR (Java) if team prefers |
| **UI / Dashboard** | **Streamlit** (fastest) → React if polish needed | Sub‑hour prototyping, Plotly charts | — |
| **Patient Comms** | Template‑based (Jinja2) + LLM polish, Twilio/WhatsApp sandbox mock | No PHI in prompts, language selectable | — |
| **Audit / Security** | Local encryption (cryptography), structured logs, role‑based view, redaction | Meets “human‑in‑loop + ethical” brief | — |

**Do NOT fine‑tune** for round‑2:
- Guidelines change annually → RAG keeps knowledge current
- No labelled denial/pre‑auth data → synthetic labels only
- Hallucination risk in clinical coding → constrained JSON + citations safer
- Fine‑tuning consumes GPU time better spent on retrieval + rules

---

## 5. MVP Scope (7‑9 Day Sprint)

| Day | Deliverable |
|-----|-------------|
| 1 | Finalize workflow, data dictionary, policy rules (HBP/STG JSON), UI wireframe |
| 2 | Generate 200‑500 synthetic patients (Synthea) + render 5‑doc pre‑auth packets (PDF) per case |
| 3 | OCR → structured JSON extraction (doc type, fields, confidence, bbox) |
| 4 | Map JSON → NHCX Claim Bundle; schema validation, FHIR round‑trip |
| 5 | RAG index (HBP/STG chunks) + rules engine → pre‑auth decision with citations & missing‑doc list |
| 6 | Denial‑risk classifier (XGBoost on synthetic features) + patient message templates (EN/HI) |
| 7 | Streamlit dashboard: TAT, first‑pass, denial reasons, override rate, audit trail |
| 8 | Evaluation on held‑out synthetic set: doc‑cls ≥95 %, field F1 ≥90 %, missing‑doc recall ≥90 %, 100 % cited outputs |
| 9 | Pitch deck (8 slides) + 5‑min demo script |

**Adjust days** if calendar tighter — drop denial‑risk classifier, keep core flow.

---

## 6. Demo Script (5 min)

1. **Problem slide** (30 s) — quantify pains from PDF
2. **Live upload** of one synthetic cataract packet (30 s)
3. **OCR → extraction → FHIR** (45 s) — show JSON + confidence
4. **Policy check** — missing consent form flagged, package rate auto‑checked (45 s)
5. **Code suggestion** — ICD‑10 + HBP code with evidence highlights (30 s)
6. **Human approve** — one click, audit entry appears (15 s)
7. **Patient WhatsApp** preview in Hindi/English (15 s)
8. **Dashboard** — before/after metrics (30 s)
9. **Architecture + safety** slide (30 s)
10. **Pilot ask** — 2 hospitals, 1 payer, 8 weeks (15 s)

---

## 7. Evaluation Targets (for judges)

| Metric | Target (demo) | How Measured |
|--------|---------------|--------------|
| Document classification accuracy | ≥95 % | 50 held‑out synthetic packets |
| Field extraction F1 | ≥90 % | Manual spot‑check 30 cases |
| Missing‑doc recall | ≥90 % | Rule‑based ground truth |
| Code suggestion top‑3 accuracy | ≥80 % | MDACE subset *if* accessible |
| Citation coverage | 100 % | Every AI output has source snippet |
| Human approval required | 100 % | No auto‑approve path |
| Pre‑auth TAT (simulated) | <15 min | Timestamp logs |
| Denial reduction (simulated) | 15‑20 % → ≤8 % | Compare rule‑only vs rule+RAG on synthetic |

*All targets are **demo goals**, not production guarantees.*

---

## 8. Pitch Deck Outline (8 Slides)

1. **Title** — ArogyaFlow: Evidence‑Grounded Pre‑Auth Copilot for India
2. **Problem** — Manual, slow, error‑prone, patient‑blind (numbers from PDF)
3. **User Journey** — Hospital → Payer → Patient (current vs proposed)
4. **Live Demo** — Screenshots + QR to running Streamlit
5. **Architecture** — OCR → FHIR → Rules/RAG → Human → Comms → Dashboard
6. **Data & Model Strategy** — Public NHA docs + Synthea + Qwen3 + bge‑m3 + PaddleOCR
7. **Safety & Compliance** — Citations, audit log, human gate, no PHI in prompts, encryption
8. **Pilot & Business Case** — 2 hospitals / 1 payer / 8 weeks → projected ₹X Cr savings, TAT ↓ 90 %

---

## 9. Key Source Links (for verification)

| Source | URL |
|--------|-----|
| NHA HBP 2.2 Manual | https://nha.gov.in/img/resources/HBP-2.2-manual.pdf |
| NHA STG Manual | https://nha.gov.in/img/pmjay-files/STG-Manual-Booklet-final.pdf |
| NHCX FHIR IG (v6.5) | https://nrces.in/ndhm/fhir/r4/index.html |
| NHCX Claim Examples | https://nrces.in/ndhm/fhir/r4/4.0.0/Claim-example-01.json.html |
| PaddleOCR‑VL‑1.6 Release | https://github.com/PaddlePaddle/PaddleOCR (v3.7.0, Jun 2026) |
| Qwen3‑8B‑Instruct | https://huggingface.co/Qwen/Qwen3-8B |
| bge‑m3 Embedding | https://huggingface.co/BAAI/bge-m3 |
| Synthea Generator | https://github.com/synthetichealth/synthea |
| MIMIC‑IV‑Note Access | https://physionet.org/content/mimic-iv-note/ |
| MDACE Dataset | https://github.com/3mcloud/MDACE |
| PMJAY Dashboard (aggregate) | https://dashboard.nha.gov.in/public |
| NHA Auto‑Adjudication Hackathon (2026) | https://aaehackathon.nhaad.in/hackathon |

---

## 10. Immediate Next Steps for You

1. **Confirm domain** — if the real round‑2 brief is Meesho, share that PDF; otherwise proceed with Valmo.
2. **Lock team roles** — OCR/extraction, FHIR/rules, LLM/RAG, UI/dashboard, pitch.
3. **Spin up Synthea** — `git clone https://github.com/synthetichealth/synthea && ./gradlew build && ./run_synthea -p 500` (adjust `-p` for patient count).
4. **Download NHA PDFs** — `wget` the HBP/STG/NHCX IG PDFs; extract text → chunk → index.
5. **Provision GPU** — 1× RTX 3090/4090 (24 GB) or Colab Pro+ for Qwen3‑8B‑4bit.
6. **Create repo** — `fastapi` backend, `streamlit` frontend, `docker-compose` for Qdrant + Ollama.
7. **Daily stand‑up** — 15 min, track against the 7‑day sprint table above.

---

*Generated 2026‑09‑24 from automated research (web search, PDF parse, public docs). All licences noted; respect PhysioNet DUA if you use MIMIC/MDACE.*