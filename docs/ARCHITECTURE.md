# Referral Review Assistant: architecture, workflow and safety

**Challenge 1: Referral intake.** For the referral coordinator at Illumination Health + Home (IHH).
The app inside is **Careward**, built by Team 1 (Aaron Zendejas, Harjeev, Vibhu Arya, Neetal).

## 1. High level

Hospitals send referral packets by fax and email all day: a referral form plus pages of chart. The
coordinator must read each packet against IHH's admission criteria and answer within a two-hour
target: yes, no with a reason, or a list of what is missing. The form is one person's summary and
can be wrong, so the chart must be checked too.

Referral Review Assistant turns a packet into **searchable, page-linked evidence**, lines each
finding up with an admission rule and the exact page it came from, surfaces conflicts and unknowns,
and drafts a clarification request. **The coordinator decides; the tool never approves or denies.**

| | |
|---|---|
| **User** | IHH referral coordinator (intake staff) |
| **Input** | A referral packet PDF, often an image-only fax |
| **Output** | Page-linked findings per admission rule (meets / unmet / unknown / conflict / clinical review), open questions for the referrer, an unsent clarification draft, a reviewed export |
| **AI** | OCR (Tesseract) by default; optional Claude extraction behind a strict validation boundary, **off by default** |
| **Runs** | Static demo on GitHub Pages (synthetic referrals); full processing on a local FastAPI service |

## 2. Workflow

```mermaid
flowchart TD
    A([Referral packet arrives]) --> B[Inbox: practice timer<br/>toward 2-hour target]
    B --> C[Open packet PDF]
    C --> D[Validate: type, size ≤25 MB,<br/>≤60 pages, not encrypted]
    D --> E[Render pages + native text<br/>OCR scanned pages with Tesseract]
    E --> F{Optional analysis<br/>configured?}
    F -- no, default --> G[Manual review:<br/>search text, read original pages]
    F -- yes --> H[Claude extracts fields + findings<br/>each with page + exact quote]
    H --> V{Validator: known rule IDs,<br/>real pages, quotes in page text,<br/>no decision keys}
    V -- fails --> G
    V -- passes --> I[Draft findings, labelled 'needs review']
    G & I --> J[Coordinator reviews each finding<br/>against the source page]
    J --> K{Gaps or conflicts?}
    K -- yes --> L[Unsent clarification draft<br/>to the referrer]
    K -- no --> M[Confirm next step<br/>rationale + reviewer + attestation]
    L --> M
    M --> N([Reviewed export / CRM handoff file<br/>approval and denial stay with staff])
```

## 3. Architecture

```mermaid
flowchart TB
    subgraph FE["Frontend (frontend/), static"]
        INBOX[Referral inbox + practice timer]
        REVIEW[Evidence review workspace<br/>field-review, referral-flow]
        PDF[pdf.js page preview]
        ONB[Onboarding / tour]
    end
    subgraph BE["Backend (backend/), FastAPI, localhost only"]
        API[app.py: upload, jobs, search,<br/>page images, reviews, outcome, export]
        OCR[PyMuPDF render + Tesseract OCR]
        AR[analysis.py: adapter boundary<br/>timeouts, retries, validate_analysis_result]
        EXT[extraction.py:<br/>ClaudeExtractionAdapter]
        EVAL[evaluation.py: blind offline eval<br/>suggest / score kept separate]
        ACC[access.py + shared.py:<br/>roles, default deny, institution scope]
        FU[referral_followup.py, timing.py]
    end
    CRIT[(Admission criteria<br/>JSON + markdown, rule IDs)]
    LLM[Claude via Anthropic API<br/>no default model, key server-side]
    TMP[(Temp dir outside repo<br/>+ in-memory records,<br/>removed on shutdown)]
    PAGES[GitHub Pages:<br/>synthetic demo only] -.serves.-> FE
    FE -- same-origin localhost only --> API
    API --> OCR --> TMP
    API --> AR
    AR -- only if IHH_ANALYSIS_ADAPTER set --> EXT --> LLM
    CRIT --> EXT
    CRIT --> AR
    LLM -- JSON --> AR
    EVAL -.offline.-> LLM
    API --> ACC
    API --> FU
    style LLM stroke-dasharray: 5 5
    style EXT stroke-dasharray: 5 5
```

Dashed boxes are optional and off unless explicitly configured.

**Where AI is used, and where it isn't**

| Component | Technology | AI? |
|---|---|---|
| Text from scanned faxes | Tesseract OCR via PyMuPDF page renders | Machine learning (OCR) |
| Field and finding extraction | Claude, one JSON object per packet, prompt `extract-draft-1`, admission rule IDs only | **LLM, optional, off by default** |
| Validating model output | `validate_analysis_result`: schema, known rule IDs, page ranges, quotes present in page text, no decision keys | No |
| Offline evaluation | `evaluation.py suggest` (model) and `score` (no model, only part that reads historical labels) | LLM in `suggest` only |
| Search, review, versioning, guards, export | Deterministic Python and JavaScript | No |
| Admit / deny | The coordinator | Human |

## 4. How it was implemented

- **Team build at the Orange County Claude Impact Lab (October 17, 2026),** with Claude and other
  coding assistants, split into backend tickets (BE-01 to BE-06, LOOP-05/06) and frontend lanes.
- **Frontend:** vanilla JavaScript modules, pdf.js for page preview, synthetic referral packets for
  practice, an allowlisted static build deployed to GitHub Pages after 34 frontend tests pass.
- **Backend:** FastAPI + PyMuPDF + Tesseract, pinned dependencies, one processing worker, bounded
  pages, OCR time and text length; versioned reviews where stale writes are rejected.
- **LLM boundary:** the adapter module is found but not executed at startup; the Claude client is
  built only on the inference path; per-call timeout and bounded retries; any invalid, untraceable
  or decision-style output becomes `failed`, never a result. Page text is wrapped as untrusted
  evidence and the model is told to ignore instructions inside it.
- **Evaluation:** blind and reproducible. `suggest` never opens the labels file; results are cached
  by packet, criteria, prompt, model and schema hash; agreement with past decisions is explicitly
  not called accuracy.
- **Tests:** 156 backend tests (with subtests) and 34 frontend tests, all invented data.

## 5. Governance and safety

```mermaid
flowchart TB
    subgraph D["Data"]
        D1[Real packets never in git<br/>synthetic packets for demo]
        D2[Temp storage outside repo,<br/>in-memory, cleared on shutdown]
        D3[No OCR text in logs<br/>static error messages]
    end
    subgraph M["Model"]
        M1[Off by default<br/>no default model]
        M2[Key server-side only]
        M3[Untrusted-evidence prompt<br/>no decision output]
        M4[Validator rejects unknown rules,<br/>bad pages, unquoted claims]
    end
    subgraph H["Human"]
        H1[Every finding reviewed<br/>against its page]
        H2[Approve / deny always<br/>rejected by server]
        H3[Rationale + reviewer +<br/>attestation to confirm]
        H4[Drafts never sent<br/>no email or fax]
    end
    subgraph A["Access"]
        A1[Localhost + Host/Origin guards]
        A2[Default deny, roles,<br/>institution scoping]
        A3[OIDC blocked until IHH<br/>chooses an identity provider]
    end
    D --> M --> H
    A -.applies throughout.- H
```

| Risk | Control |
|---|---|
| Admitting someone the site can't safely care for, or turning away someone who needed the bed | The tool never decides: the server rejects approve/deny in this version; every finding links to a page and quote; missing or redacted evidence is "unknown", never "meets"; conflicts are shown, not resolved silently; internal criteria take precedence over the referral form |
| Being blamed for a machine's mistake | AI output is labelled draft and "needs review"; confirmation requires rationale, reviewer label and attestation; editing a field invalidates the earlier confirmation |
| Patient privacy | Real referral packets never committed (CI checks the deploy contains only allowlisted synthetic files); processing on localhost only; temp files outside the repo; no OCR text in logs; data not durable |
| Model invents evidence | Quotes must appear in the page text; pages must exist; rule IDs must be real; failures become `failed`, not results |
| Prompt injection from packet text | Page text is wrapped as untrusted evidence; instructions inside it are ignored |
| Secrets | `ANTHROPIC_API_KEY` server-side only, never in git, issues, PRs or the frontend; `.env` git-ignored |
| Overclaiming | Practice timer is illustrative, not a service deadline; no measured time savings; agreement with historical labels is not accuracy |
| Access | Shared-record service off without identity; dev sessions are rehearsal-only; OIDC blocked until IHH decides |

**Known limits.** The model-backed extractor targets a draft contract and is not tested against a
real model. OCR orientation, handwriting and checkbox accuracy are outstanding. There is no
production authentication, durable audit or compliance certification. The follow-up and
shared-record modules are tested but not yet wired into one persistent frontend workflow.
