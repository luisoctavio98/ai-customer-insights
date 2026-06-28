# Development Logbook - Phase 0: Setup and Observability

## 1. Action Plan

**Phase Objective:** Establish the baseline project infrastructure, isolating dependencies within a virtual environment, configuring strict environment variables, and ensuring LangSmith telemetry is fully operational before authoring business logic.

### Architectural Approach & Methodology (Top-Down Pedagogy)

Every structural decision in this phase follows our mandatory architectural framework: explaining production necessity, evaluating alternatives, and establishing strict boundaries between mandatory and optional implementations.

#### 1. Git Isolation (`feature/bootstrap-core-observability`)

* **Mandatory Production Necessity:** Working directly on `main` introduces immediate risks of deploying broken code or conflicting dependencies to production. Isolating work within a feature branch ensures `main` remains stable and clean.
* **Alternatives:**
  * *More Complex:* Adding artificial ticketing numbers like `feature/INFRA-1001/bootstrap-core-observability`. While used in corporate Jira environments, this adds unnecessary visual nesting and clutter for a standalone repository.
  * *Simpler:* Working directly on `main` or using a weak, tutorial-like name (`phase-0-setup`).
* **Decision & Optionality:** We chose a clean, professional feature branch (`feature/bootstrap-core-observability`). Using the exact prefix `feature/` is **MANDATORY** per Rule #5. The exact branch suffix `bootstrap-core-observability` is **OPTIONAL** but highly recommended to maintain a rigorous production-grade standard without artificial clutter.

#### 2. Dependency Isolation (`pyproject.toml` & `.venv`)

* **Mandatory Production Necessity:** Global package installations lead to version conflicts across projects. Isolating dependencies via `pyproject.toml` ensures deterministic, repeatable builds across developer machines and CI/CD pipelines.
* **Alternatives:**
  * *More Complex:* Containerized dev environments (Docker Compose + DevContainers).
  * *Simpler:* Relying on global Python environments or loose `requirements.txt` files.
* **Decision & Optionality:** We chose Python's native `venv` combined with a declarative `pyproject.toml`. Using `.venv` and `pyproject.toml` is **MANDATORY**.

#### 3. Secrets Management (`.env`)

* **Mandatory Production Necessity:** Hardcoding API keys in source code is a critical security vulnerability. Decoupling secrets from code via environment variables is required for secure deployment.
* **Alternatives:**
  * *More Complex:* Fetching secrets dynamically at runtime from AWS Secrets Manager or HashiCorp Vault.
  * *Simpler:* Hardcoding keys directly in Python scripts (extreme security risk).
* **Decision & Optionality:** We chose a local `.env` file loaded via `pydantic-settings`. Keeping `.env` out of version control (via `.gitignore`) and storing only authorized keys is **MANDATORY**.

#### 4. Centralized Telemetry (`src/core/tracing.py`)

* **Mandatory Production Necessity:** Deploying LLMs without immediate observability creates a black box where debugging hallucinations, tracking token spend, and monitoring latency become impossible. Initializing LangSmith centrally ensures every chain is automatically traced.
* **Alternatives:**
  * *More Complex:* Configuring OpenTelemetry collectors exporting to self-hosted Jaeger/Prometheus in a private VPC.
  * *Simpler:* Relying on basic `print()` statements or standard `logging` without capturing LLM DAGs or token metrics.
* **Decision & Optionality:** We chose LangSmith tracing initialized via `pydantic-settings`. Validating environment variables at startup and enabling LangSmith tracing is **MANDATORY** per Rule #3.

---

### Step-by-Step Implementation Guide for the Developer

#### Step 0.0: Git Feature Branch Creation (Step Zero)

* **Understanding `feature/` (It is NOT a folder):** Because you just created the repository, it is critical to know that `feature/` in the branch name is **NOT a physical folder or directory**. It is simply a text naming tag (a prefix string) used inside Git's internal database to categorize this virtual timeline. When you eventually merge this branch into `main`, Git will not create a "feature" folder; it will seamlessly apply your code directly into the main project root.

1. Open your terminal in the project root and execute the following commands to ensure you are on `main`, pull any changes, and generate your isolated feature branch:

   ```bash
   git checkout main
   git pull origin main
   git checkout -b feature/bootstrap-core-observability
   ```

#### Step 0.1: Virtual Environment and Dependencies

1. Now that you are safely on your feature branch, create the virtual environment in the project root:

   ```bash
   python -m venv .venv
   ```

2. Activate the virtual environment:

   ```bash
   # On Windows:
   .\.venv\Scripts\activate
   ```

3. Create the `pyproject.toml` file in the root directory with the following baseline configuration:

   ```toml
   [build-system]
   requires = ["setuptools>=61.0"]
   build-backend = "setuptools.build_meta"

   [project]
   name = "ai-customer-insights"
   version = "0.1.0"
   description = "FastAPI Microservice for Reviews Classification and Semantic Search"
   readme = "README.md"
   requires-python = ">=3.10"
   dependencies = [
       "fastapi>=0.110.0",
       "uvicorn>=0.28.0",
       "langchain>=0.1.13",
       "langchain-google-genai>=1.0.1",
       "langsmith>=0.1.31",
       "pydantic>=2.6.4",
       "pydantic-settings>=2.2.1",
       "boto3>=1.34.69",
       "pinecone-client>=3.1.0",
       "pandas>=2.2.1",
       "slowapi>=0.1.9"
   ]
   ```

4. Install the project and its dependencies in editable mode:

   ```bash
   pip install -e .
   ```

#### Step 0.2: Configuration File `.env`

1. Create a `.env` file in the root directory containing **only** the authorized keys:

   ```env
   LANGCHAIN_TRACING_V2=true
   LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
   LANGCHAIN_API_KEY=your_langsmith_api_key
   GEMINI_API_KEY=your_gemini_api_key
   ```

#### Step 0.3: Base Directory Structure

1. Create the base folder structure:

   ```text
   ai-customer-insights/
   ├── data/
   │   └── raw/
   ├── src/
   │   ├── core/
   │   ├── agent_car_reviews/
   │   └── agent_clothing/
   └── tests/
   ```

#### Step 0.4: Observability Initialization (`src/core/tracing.py`)

1. Create `src/core/tracing.py` and implement the LangSmith configuration using `pydantic_settings` for robust environment validation:

   ```python
   import os
   from pydantic_settings import BaseSettings
   from langsmith import traceable

   class TracingSettings(BaseSettings):
       langchain_tracing_v2: str = "true"
       langchain_endpoint: str = "https://api.smith.langchain.com"
       langchain_api_key: str
       gemini_api_key: str

       class Config:
           env_file = ".env"
           extra = "ignore"

   def init_tracing() -> TracingSettings:
       """
       Initializes and validates the observability environment variables required for LangSmith.
       """
       settings = TracingSettings()
       os.environ["LANGCHAIN_TRACING_V2"] = settings.langchain_tracing_v2
       os.environ["LANGCHAIN_ENDPOINT"] = settings.langchain_endpoint
       os.environ["LANGCHAIN_API_KEY"] = settings.langchain_api_key
       os.environ["GEMINI_API_KEY"] = settings.gemini_api_key
       return settings

   # Export traceable so other modules can import it cleanly from core
   __all__ = ["init_tracing", "traceable"]
   ```

#### Step 0.5: Tracing Validation Script (`tests/test_tracing.py`)

1. Create `tests/test_tracing.py` to verify trace delivery to LangSmith before proceeding to Phase 1:

   ```python
   from src.core.tracing import init_tracing, traceable

   # 1. Validate and initialize environment
   init_tracing()

   # 2. Define a dummy traceable function
   @traceable(name="dummy_calibration_trace")
   def dummy_trace_call(input_text: str) -> dict:
       return {"status": "success", "length": len(input_text), "echo": input_text}

   def test_langsmith_connection():
       print("Starting LangSmith trace delivery test...")
       result = dummy_trace_call("Initial LLMOps observability calibration")
       assert result["status"] == "success"
       print("Trace successfully sent! Check your LangSmith dashboard.")

   if __name__ == "__main__":
       test_langsmith_connection()
   ```

---

## 2. Historical Record

*(This section will be updated as the developer implements the steps, documenting any errors encountered, dependency adjustments, or architectural decisions made during the phase).*
