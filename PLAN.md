# Execution Plan: AI Customer Insights

**Repository:** `ai-customer-insights`
**Objective:** Develop two AI chains (Car Reviews Classification and Clothing Reviews Semantic Search) grouped into a single FastAPI Microservice.
**State Architecture:** `STATELESS` (Requests are independent; they do not need to remember past interactions. LangSmith handles telemetry natively).
> **LLMOps Disclaimer (Data Retention):** LangSmith's free tier automatically deletes traces after 14 days. This is the expected behavior for operational observability. If permanent log retention is desired (e.g., for future *Fine-Tuning*), the industry standard architecture would be to implement an asynchronous queue to a Data Lake (AWS S3 + Athena). We will not implement this here to keep the scope bounded.
**Total Estimated Time:** 8 to 10 hours (Adjusted for RAG Offline Evaluation script and Security Phase).

---

## Directory Structure to Implement

```text
ai-customer-insights/
├── .env                       <-- Google API & LangSmith Keys
├── pyproject.toml             <-- FastAPI, LangChain, Boto3, FAISS dependencies
├── RULES.md                   
├── PLAN.md                    
├── README.md                  <-- Portfolio-specific documentation
├── logbook/                   <-- Detailed logs for each development phase
├── data/                      <-- Original and processed CSVs
│   └── raw/
├── src/
│   ├── main.py                <-- FastAPI Gateway
│   ├── core/                  <-- Shared Infrastructure
│   │   ├── config.py
│   │   ├── tracing.py
│   │   └── aws_client.py      
│   ├── agent_car_reviews/     <-- Zero-Shot Logic
│   │   ├── schemas.py
│   │   ├── prompts.py
│   │   └── classification.py
│   └── agent_clothing/        <-- RAG Logic
│       ├── pinecone_client.py <-- Vector DB Connection
│       └── retrieval_chain.py
└── tests/
```

---

## Architectural Trade-offs
*   **Gemini Flash vs Local Models:** Gemini Flash will be used for Zero-Shot classification. **Trade-off:** We sacrifice total independence (which a local HuggingFace model would provide) in exchange for gaining millisecond latency and avoiding hosting costs thanks to the Google AI Studio Free Tier.
*   **Pinecone vs Local FAISS:** We will use Pinecone (Cloud Serverless). **Trade-off:** It adds network latency when querying an external API compared to a local in-memory index, but we gain cloud persistence, Enterprise scalability, and avoid loading heavy indexes into our server's RAM.
*   **Direct Boto3 vs Heavy Frameworks:** We will connect AWS Titan via Boto3 / LangChain. **Trade-off:** Requires local AWS CLI profiles, but ensures security (no keys in `.env`) and granular cost control for Bedrock.

---

## Logbook Instructions
**MANDATORY FOR THE AI:** The development is NOT autonomously orchestrated by the AI; the AI acts as an Architect and the User as an Implementer. Upon starting any Phase, the AI **MUST** create a Markdown file inside the `logbook/` directory (e.g., `logbook/phase_0_setup.md`).
This file has a two-stage lifecycle:
1. **Phase Start (Action Plan):** The AI writes the detailed technical step-by-step guide here that the Developer must implement.
2. **During and End of Phase (Historical Record):** As the Developer implements the code and asks the AI for corrections, the logbook is updated with the errors found, solutions provided, and architectural decisions. At the end of the phase, the file remains as an immutable record of what actually happened.
**Rule:** Never dictate code to start a phase without first having written the "Action Plan" in its corresponding logbook file.

---

## Granular Execution Checklist

### Phase 0: Setup and Observability (Est. 1 Hour)
| # | Task | Folder/File | Status |
|---|---|---|---|
| 0.1 | **MANDATORY**: Create a virtual environment in the `.venv` folder and configure `pyproject.toml` fixing dependencies to avoid future breaks. | `pyproject.toml` | `[ ]` |
| 0.2 | Create an `.env` file containing **only** `LANGCHAIN_API_KEY` and `GEMINI_API_KEY`. | `.env` | `[ ]` |
| 0.3 | Build the base folder structure (`src`, `data`, `tests`). | `/` | `[ ]` |
| 0.4 | **MANDATORY**: Implement observability initialization. | `src/core/tracing.py` | `[ ]` |
| 0.5 | Write a test script to validate trace sending to LangSmith. Do not advance if it fails. | `tests/test_tracing.py` | `[ ]` |

### Phase 1: Car Reviews Agent (Zero-Shot Classification) (Est. 2 Hours)
| # | Task | Folder/File | Status |
|---|---|---|---|
| 1.1 | **MANDATORY**: Define Pydantic schemas (`ReviewInput`, `ReviewOutput` with sentiment/translation). | `src/agent_car_reviews/schemas.py` | `[ ]` |
| 1.2 | Design System Prompt using static Few-Shot (2 calibration examples). | `src/agent_car_reviews/prompts.py` | `[ ]` |
| 1.3 | Instantiate Gemini 1.5 Flash client. | `src/core/config.py` | `[ ]` |
| 1.4 | Build the LangChain chain connecting Prompt + LLM + OutputParser. | `src/agent_car_reviews/classification.py` | `[ ]` |
| 1.5 | Test locally. Check LangSmith: **Goal:** Latency < 800ms. | `tests/test_car_agent.py` | `[ ]` |

### Phase 2: Clothing Reviews Agent (Embeddings & RAG) (Est. 3-4 Hours)
| # | Task | Folder/File | Status |
|---|---|---|---|
| 2.1 | Load "Women's Clothing" CSV and clean nulls with Pandas. | `data/etl_pipeline.py` | `[ ]` |
| 2.2 | **MANDATORY**: Instantiate AWS Titan V2 using local profiles. *Justification: Bidirectional encoders are mathematically optimal for Embeddings.* | `src/core/aws_client.py` | `[ ]` |
| 2.3 | Apply chunking to reviews and configure free Serverless index in Pinecone. | `src/agent_clothing/pinecone_client.py` | `[ ]` |
| 2.4 | Send Embedding batches (Upsert) to Pinecone. | `src/agent_clothing/pinecone_client.py` | `[ ]` |
| 2.5 | Build RAG Chain: User Query -> Embedding -> Top-K Search -> Context to Gemini Flash. | `src/agent_clothing/retrieval_chain.py` | `[ ]` |
| 2.6 | **MANDATORY (Core - Eval):** Create offline evaluation script to measure RAG precision/recall against a test dataset. | `tests/evaluate_rag.py` | `[ ]` |

### Phase 3: Gateway and Deployment (FastAPI) (Est. 1 Hour)
| # | Task | Folder/File | Status |
|---|---|---|---|
| 3.1 | Create base FastAPI application. | `src/main.py` | `[ ]` |
| 3.2 | Expose endpoint: `POST /v1/car-reviews/classify` (connected to Phase 1 chain). | `src/main.py` | `[ ]` |
| 3.3 | Expose endpoint: `POST /v1/clothing/search` (connected to Phase 2 RAG chain). | `src/main.py` | `[ ]` |
| 3.4 | **MANDATORY**: Verify Swagger UI (`/docs`) and confirm that Pydantic schemas are documented correctly. | `src/main.py` | `[ ]` |

---

### Project Retrospective (Upon completion)

| Question | Answer |
|----------|-----------|
| What did I learn about the latency difference between Encoders and Decoders? | _(note at the end)_ |
| How useful was having LangSmith trace every RAG step? | _(note at the end)_ |
| Did we meet the budget and estimated time? | _(note at the end)_ |

### Bonus Phase (Optional - Periphery): Security and Scalability
| # | Task | Folder/File | Status |
|---|---|---|---|
| B.1 | Add an API Key authentication Middleware to protect endpoints. | `src/main.py` | `[ ]` |
| B.2 | Add Rate Limiting (`slowapi`) to prevent classification API abuse. | `src/main.py` | `[ ]` |
| B.3 | Document in a comment block why using a synchronous endpoint to classify massive arrays of texts is an anti-pattern and the theoretical need for an asynchronous worker (e.g., Celery). | `src/main.py` | `[ ]` |
