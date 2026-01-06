# Insurance-Claim-Agent-HOL
## Insurance Claims Agent – Hands on Lab Steps (Sections 1–7)

# Lab Instructions
---

## 1. Parse Documents with Cortex (PARSE_DOCUMENT & EXTRACT_ANSWER)

### Goal
- Convert uploaded claim artifacts (notes, guidelines, invoices) in the `LOSS_EVIDENCE` stage into **queryable tables**:
  - `PARSED_CLAIM_NOTES`
  - `PARSED_GUIDELINES`
  - `PARSED_INVOICES`
- Extract **claim numbers** from notes and invoices for joining to structured data.

### Concepts
- **`SNOWFLAKE.CORTEX.PARSE_DOCUMENT`**  
  - Reads files from a Snowflake stage (e.g. `@INSURANCE_CLAIMS_DEMO.LOSS_CLAIMS.LOSS_EVIDENCE`).  
  - Uses OCR / layout‑aware parsing to produce text content.  
  - Returns a `VARIANT`; the lab converts `:content` to `VARCHAR`.

- **`SNOWFLAKE.CORTEX.EXTRACT_ANSWER`**  
  - Accepts free‑form text and a **natural language question** (e.g. *“What is the claim number?”*).  
  - Returns candidate answers with `answer` and `score`.  
  - In the notebook, results are FLATTENed and filtered to `score >= 0.5`, then stored as `CLAIM_NO`.

### What to run in the notebook
In `insurance_claims_lab.ipynb`:
1. **Section 3 markdown** – read the explanation of PARSE_DOCUMENT and EXTRACT_ANSWER.
2. Run the three SQL cells:
   - Create and populate `PARSED_CLAIM_NOTES` (notes + extracted `CLAIM_NO`).
   - Create and populate `PARSED_GUIDELINES` (guideline text).
   - Create and populate `PARSED_INVOICES` (invoices + extracted `CLAIM_NO`).

> Docs reference: `https://docs.snowflake.com` → search for  
> **“SNOWFLAKE.CORTEX.PARSE_DOCUMENT”** and **“SNOWFLAKE.CORTEX.EXTRACT_ANSWER”**.

---

## 2. Chunk Documents and Build Cortex Search Indexes

### Goal
- Break long parsed documents into **small, overlapping text chunks**.
- Attach **presigned URLs** so users and the agent can open originals.
- Build **Cortex Search Services** over notes and guidelines.

### Concepts
- **Chunking with `SPLIT_TEXT_RECURSIVE_CHARACTER`**  
  - Produces 200‑character chunks with 30‑character overlap for better recall.  
  - Used to populate:
    - `NOTES_CHUNK_TABLE`
    - `GUIDELINES_CHUNK_TABLE`

- **Presigned URLs with `GET_PRESIGNED_URL`**  
  - Generates time‑limited URLs back to stage files for drill‑through.

- **Cortex Search Service (`CREATE CORTEX SEARCH SERVICE`)**  
  - Indexes the `chunk` column using `snowflake-arctic-embed-l-v2.0`.  
  - Exposes attributes like `file_url`, `claim_no`, and `filename` in search results.

### What to run
In the notebook:
1. Read **Section 4 markdown** for Cortex Search overview.
2. Run SQL cells to:
   - Create `NOTES_CHUNK_TABLE` from `PARSED_CLAIM_NOTES`.
   - Create `GUIDELINES_CHUNK_TABLE` from `PARSED_GUIDELINES`.
   - Create two **Cortex Search Services**:
     - `INSURANCE_CLAIMS_DEMO_claim_notes`
     - `INSURANCE_CLAIMS_DEMO_guidelines`

> Docs reference: `https://docs.snowflake.com` → search for  
> **“Cortex Search”** and **“SPLIT_TEXT_RECURSIVE_CHARACTER”**.

---

## 3. Helper AI Functions and Procedures

### Goal
Create reusable **SQL functions** and a **stored procedure** that the agent and analysts can call:
- Document classification
- Document parsing helper
- Image summarization
- Call transcription

### Objects
- `CLASSIFY_DOCUMENT(FILE_NAME, STAGE_NAME)`  
  - Uses **`AI_EXTRACT`** to classify documents into types (Invoice, Evidence Image, etc.).  
  - Returns an OBJECT with classification, description, business context, purpose, and confidence.

- `PARSE_DOCUMENT_FROM_STAGE(FILE_NAME)`  
  - Thin wrapper over **`AI_PARSE_DOCUMENT`** for the `LOSS_EVIDENCE` stage.  
  - Returns a `VARIANT` representation of the parsed document.

- `GET_IMAGE_SUMMARY(IMAGE_FILE, STAGE_NAME)`  
  - Uses **`SNOWFLAKE.CORTEX.COMPLETE`** (model `claude-3-5-sonnet`) to generate a ~100‑word image summary.

- `TRANSCRIBE_AUDIO_SIMPLE(FILE_NAME, STAGE_NAME)`  
  - Uses **`AI_TRANSCRIBE`** with speaker‑level timestamps.  
  - Wraps the result plus error handling into a single OBJECT.

### What to run
In the notebook:
1. Read **Section 5 markdown** for a summary of each function/procedure.
2. Run the SQL cells to create:
   - `CLASSIFY_DOCUMENT`
   - `PARSE_DOCUMENT_FROM_STAGE`
   - `GET_IMAGE_SUMMARY`
   - `TRANSCRIBE_AUDIO_SIMPLE`

> Docs reference: search in `https://docs.snowflake.com` for  
> **“AI_EXTRACT”**, **“AI_PARSE_DOCUMENT”**, **“SNOWFLAKE.CORTEX.COMPLETE”**, and **“AI_TRANSCRIBE”**.

---

## 4. Cortex Analyst Semantic View

### Goal
Define **`CA_INSURANCE_CLAIMS_DEMO`** – a semantic view that tells Cortex Analyst:
- Which tables it can use.
- How to **join** them.
- Which columns are facts, dimensions, and time dimensions.

### Highlights
- Tables: `AUTHORIZATION`, `CLAIMS`, `CLAIM_LINES`, `FINANCIAL_TRANSACTIONS`, `INVOICES`.
- Relationships:
  - Links claim headers to lines, lines to financials, and lines to invoices and authorization.
- Facts:
  - E.g., `FIN_TX_AMT`, `INVOICE_AMOUNT`, `FROM_AMT`, `TO_AMT`.
- Dimensions:
  - Claim identifiers, statuses, dates, vendors, currencies, etc., with rich synonyms.
- Extension metadata:
  - **Sample values** and **verified queries** for common audit questions, e.g.:
    - “Was a payment made in excess of the performer authority?”
    - “Was a payment issued to the vendor 30+ days after the invoice date?”

### What to run
In the notebook:
1. Read **Section 6 markdown** to understand the semantic view concept.
2. Run the SQL cell that creates `CA_INSURANCE_CLAIMS_DEMO`.

> Docs reference: `https://docs.snowflake.com` → search for  
> **“Cortex Analyst”** and **“semantic model”**.

---

## 5. Create the Insurance Claims Agent (Snowflake Intelligence)

### Goal
Create `CLAIMS_AUDIT_AGENT`, a **Snowflake Intelligence Agent** that:
- Understands the claims domain via your semantic view.
- Uses Cortex Search to pull relevant notes and guidelines.
- Calls helper functions for classification, parsing, imaging, transcription, and redaction.

### Key pieces in the agent spec
- **Models**: orchestration set to `auto`.
- **Instructions**:
  - Persona: “You are an insurance claims agent.”
  - Priorities: use Cortex Analyst (`CA_INS`) for quantitative questions, Cortex Search for notes/guidelines, include charts where possible.
  - Claim completeness criteria: claim header, claim lines, financials, and claim notes present.
- **Sample questions**:
  - Settlement window compliance, reserve rationale, payments over reserve, transcript and intent for a call, image summary, claim completeness, etc.
- **Tools**:
  - `TEXT2SQL` (Cortex Analyst over your semantic model).
  - `SEARCH_CLAIM_NOTES` and `SEARCH_GUIDELINES` (Cortex Search).
  - Generic tools for `CLASSIFY_DOCUMENT`, `PARSE_DOCUMENT_FROM_STAGE`, `GET_IMAGE_SUMMARY`, `TRANSCRIBE_AUDIO_SIMPLE`, and email redaction.
- **Tool resources** map tool names to actual functions, procedures, and search services in `INSURANCE_CLAIMS_DEMO.LOSS_CLAIMS`.

### What to run
In the notebook:
1. Read **Section 7 markdown** for the high‑level picture of Snowflake Intelligence Agents.
2. Run the SQL cell that issues `CREATE OR REPLACE AGENT ... FROM SPECIFICATION $$ { ... } $$`.

> Docs reference: `https://docs.snowflake.com` → search for  
> **“Snowflake Intelligence”** and **“Agent specification”**.

---

## 6. Register and Validate the Agent in Snowflake Intelligence

### Goal
Make `CLAIMS_AUDIT_AGENT` available in the **Snowflake Intelligence** UI and APIs.

### What happens
- A helper procedure `ADD_AGENT_TO_INTELLIGENCE()`:
  - Drops the agent from the default Intelligence object if it already exists (ignoring errors).
  - Adds `CLAIMS_AUDIT_AGENT` back to `SNOWFLAKE_INTELLIGENCE_OBJECT_DEFAULT`.
  - Returns a status message.

### What to run
In the notebook:
1. Ensure you are using **`ACCOUNTADMIN`** (or equivalent).
2. Run the SQL cell in **Section 9** to:
   - Create `ADD_AGENT_TO_INTELLIGENCE`.
   - Execute `CALL INSURANCE_CLAIMS_DEMO.LOSS_CLAIMS.ADD_AGENT_TO_INTELLIGENCE();`.

### Validate in Snowsight
1. Open **Snowflake Intelligence** under AI/ML section in Snowsight.
2. Locate and open the `Insurance Claims Agent` / `CLAIMS_AUDIT_AGENT`.
3. Try sample questions from the notebook:
   - “Was a payment made in excess of the performer authority?”
   - “Was a payment issued to the vendor 30+ calendar days after the invoice was received?”
   - “Is claim 1899 complete?”

If responses reference claim data, claim notes, guidelines, and documents correctly, your end‑to‑end Insurance Claims Agent is working.


