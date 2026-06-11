# TECH_SPEC.md – ragbench  

**Document version:** 1.0 – 2026‑06‑11  
**Owner:** Senior Product/Engineering Lead, AxentX  
**Repository:** https://github.com/axentx/ragbench  

---  

## 1. Overview  

`ragbench` is a **zero‑trust, self‑hosted benchmarking framework** for Retrieval‑Augmented Generation (RAG) pipelines. It provides:

* **Isolated execution** – benchmarks run inside a Docker‑based sandbox with no outbound network unless explicitly allowed.  
* **Schema‑driven validation** – model outputs are automatically validated against user‑defined JSON schemas (Pydantic v2).  
* **Extensible metric engine** – plug‑in architecture for custom evaluation metrics (e.g., R‑Precision, Faithfulness, Latency).  
* **CLI‑first workflow** – a single `ragbench` command line tool orchestrates configuration loading, execution, and reporting.  
* **Enterprise‑grade compliance** – data never leaves the host environment; all artefacts are stored encrypted at rest.

The technical specification below details the architecture, components, data model, public APIs, technology stack, dependencies, and deployment model required to ship a production‑ready release.

---  

## 2. Architecture Overview  

```
+-------------------+        +-------------------+        +-------------------+
|   User / CI/CD    |  CLI   |   ragbench Core   |  API   |   Metric Plugins  |
|  (bash, GitHub)   |<------>|  (Python pkg)    |<------>| (Python modules) |
+-------------------+        +-------------------+        +-------------------+
          |                         |                               |
          |                         |                               |
          v                         v                               v
+-------------------+   Docker   +-------------------+   Docker   +-------------------+
|   Host Machine    |<---------->|   Sandbox Runner  |<---------->|   Model Service   |
| (Linux/macOS/Win) |   (isol.) |  (Docker image)   |   (API)   |  (any LLM/Vector) |
+-------------------+            +-------------------+           +-------------------+

```

* **CLI Layer** – `ragbench` entry point built with **Click**. Parses a YAML/JSON config, resolves plugins, and launches the **Sandbox Runner**.  
* **Sandbox Runner** – Docker container (`axentx/ragbench-runner:<semver>`) that contains the core benchmark engine, the user‑provided model client, and metric plugins. The container runs with `--network none` by default; network can be whitelisted via config for on‑prem vector stores.  
* **Core Engine** – orchestrates the evaluation loop:  
  1. Load test cases (queries + ground‑truth).  
  2. For each case, invoke the **Model Service** (HTTP or local Python callable).  
  3. Validate the raw response against the **JSON Schema** (Pydantic).  
  4. Feed the validated payload to each **Metric Plugin**.  
  5. Store per‑case results in an SQLite DB (encrypted with SQLCipher).  
* **Metric Plugins** – Python packages that expose a `Metric` subclass with `compute(reference, prediction) → float`. Discovered via entry‑point group `ragbench.metrics`.  
* **Reporting** – After execution, the core engine aggregates metrics and renders a static HTML dashboard (Chart.js + Bootstrap) saved under `output/`.  

---  

## 3. Component Details  

### 3.1 CLI (`ragbench/__main__.py`)  

| Command | Options | Description |
|---------|---------|-------------|
| `run` | `--config <path>` (required) <br> `--output <dir>` (default `./output`) <br> `--dry-run` | Execute a benchmark suite. |
| `list-metrics` | – | Show all discovered metric plugins. |
| `validate-schema` | `--schema <path>` `--sample <json>` | Validate a sample payload against a schema. |

The CLI uses **Click** for argument parsing and automatically injects a `--log-level` flag (DEBUG/INFO/WARN/ERROR).

### 3.2 Core Engine (`ragbench/engine.py`)  

* **Config Loader** – YAML/JSON parsed with **PyYAML**; schema validated via a Pydantic `BenchmarkConfig` model.  
* **TestCase Provider** – Abstract base class `TestCaseSource`; built‑in implementations: `FileSource` (JSONL), `SQLSource`.  
* **Model Adapter** – Interface `ModelClient` with method `generate(prompt: str, **kwargs) -> dict`. Implementations:  
  * `OpenAIClient` (calls `https://api.openai.com/v1/chat/completions`) – **only for local dev** (network must be whitelisted).  
  * `LocalHFClient` – loads a HuggingFace model via **vLLM** or **SGLang** (both are verified frameworks in the company knowledge base).  
* **Validator** – `ResponseValidator` wraps a Pydantic model generated from the user‑supplied JSON schema. Returns a `ValidatedResponse` or raises `ValidationError`.  
* **Metric Engine** – Dynamically loads all entry‑points under `ragbench.metrics`. Each metric receives the **ground‑truth** and **validated model output**.  
* **Result Store** – SQLite (`results.db`) encrypted with a per‑run key derived from the CLI `--output` directory name (PBKDF2‑SHA256).  

### 3.3 Metric Plugin API  

```python
# ragbench/metrics/__init__.py
from .base import Metric

class Metric(ABC):
    name: str                     # human readable
    description: str

    @abstractmethod
    def compute(self, reference: dict, prediction: dict) -> float:
        """Return a scalar score for a single test case."""
```

Plugins are discovered via `importlib.metadata.entry_points(group="ragbench.metrics")`.  

**Bundled metrics (v0.1.0)**  

| Plugin | Metric | Formula |
|--------|--------|---------|
| `ragbench.metrics.rouge` | ROUGE‑L | Standard ROUGE‑L F1 |
| `ragbench.metrics.factuality` | Factuality | Binary exact‑match against `reference["facts"]` |
| `ragbench.metrics.latency` | Latency | `end_time - start_time` (seconds) |

### 3.4 Reporting Dashboard (`ragbench/report.py`)  

* Generates `index.html` with:  
  * Summary table (overall averages, pass/fail counts).  
  * Interactive line/bar charts per metric (Chart.js).  
  * Downloadable CSV of raw per‑case scores.  
* All assets are bundled in the Docker image under `/usr/share/ragbench/static`.  

---  

## 4. Data Model  

### 4.1 Configuration (`BenchmarkConfig`)  

```yaml
benchmark:
  name: "my-rag-eval"
  description: "Evaluation of internal RAG pipeline"
  version: "1.0"
  schema: "./schemas/answer_schema.json"
  model:
    type: "local_hf"
    path: "/models/tiiuae/falcon-40b"
    parameters:
      temperature: 0.0
      max_new_tokens: 256
  test_cases:
    source: "file"
    path: "./data/test_cases.jsonl"
  metrics:
    - "rouge"
    - "factuality"
    - "latency"
output:
  dir: "./output"
  encrypt: true
```

*All fields are validated by `BenchmarkConfig` (Pydantic).*

### 4.2 Test Case (JSONL line)  

```json
{
  "id": "tc-001",
  "query": "What are the privacy guarantees of ragbench?",
  "ground_truth": {
    "answer": "ragbench runs in an isolated Docker container with no outbound network...",
    "facts": ["isolated Docker", "no outbound network"]
  }
}
```

### 4.3 Result Record (SQLite schema)  

| Column | Type | Description |
|--------|------|-------------|
| `run_id` | TEXT | UUID for the benchmark execution |
| `case_id` | TEXT | Test case identifier |
| `prompt` | TEXT | Prompt sent to model |
| `raw_output` | TEXT | Raw JSON string returned by model |
| `validated` | BOOLEAN | Whether schema validation succeeded |
| `validation_error` | TEXT | Error message if validation failed |
| `metric_name` | TEXT | Name of metric (e.g., `rouge`) |
| `metric_score` | REAL | Computed score |
| `latency_sec` | REAL | Generation latency |
| `timestamp` | DATETIME | Execution timestamp |

---  

## 5. Key APIs / Interfaces  

| Interface | Module | Method Signature | Purpose |
|-----------|--------|------------------|---------|
| `ModelClient` | `ragbench/model.py` | `generate(prompt: str, **kwargs) -> dict` | Invoke the underlying LLM and return raw JSON payload |
| `TestCaseSource` | `ragbench/cases.py` | `iter_cases() -> Iterator[TestCase]` | Stream test cases to the engine |
| `ResponseValidator` | `ragbench/validation.py` | `validate(payload: dict) -> ValidatedResponse` | Enforce JSON schema via Pydantic |
| `Metric.compute` | `ragbench/metrics/base.py` | `compute(reference: dict, prediction: dict) -> float` | Compute a scalar evaluation metric |
| `ReportGenerator` | `ragbench/report.py` | `generate(run_id: str, output_dir: Path) -> None` | Produce HTML dashboard and CSV |

All public objects are exported in `ragbench.__init__` for external tooling (e.g., CI integration).

---  

## 6. Technology Stack  

| Layer | Technology | Version (as of 2026‑06‑11) | Rationale |
|-------|------------|---------------------------|-----------|
| Language | Python | 3.11+ | Modern typing, async support |
| Schema Validation | Pydantic | v2.7 | Fast, declarative, supports JSON schema generation |
| CLI | Click | v8.1 | Mature, composable, auto‑help |
| Container Runtime | Docker | 24.0+ | Industry‑standard isolation |
| Model Serving (optional) | vLLM | `vllm-project/vllm` (commit a1b2c3) | High‑throughput inference, verified in company repo |
| Structured Generation (optional) | SGLang | `sgl-project/sglang` (commit d4e5f6) | Supports token‑level control, also verified |
| Database | SQLite + SQLCipher | 3.45 + 4.5 | Lightweight, encrypted at rest |
| Reporting UI | Chart.js, Bootstrap | 3.9 / 5.3 | Zero‑dependency static assets |
| Testing | pytest | 8.2 | Standard Python test runner |
| CI/CD | GitHub Actions | – | Already used in repo (badge) |
| Packaging | setuptools + `pyproject.toml` | – | Editable install (`pip install -e .`) |

---  

## 7. Dependencies  

`requirements.txt` (pinned to known‑good versions)

```
click>=8.1,<9
pydantic>=2.7,<3
pyyaml>=6.0,<7
sqlcipher3-binary>=0.5.0
pytest>=8.2
importlib-metadata>=6.0
```

Optional extras (installed via `pip install ragbench[local]`):

```
vllm>=0.4.0
sglang>=0.1.0
torch>=2.2.0
```

All dependencies are MIT or Apache‑2.0 compatible, matching the repository license.

---  

## 8. Deployment & Operations  

### 8.1 Docker Image  

**Dockerfile (simplified)**  

```Dockerfile
FROM python:3.11-slim

# System deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    git curl && rm -rf /var/lib/apt/lists/*

# Install Python deps
COPY pyproject.toml poetry.lock /app/
WORKDIR /app
RUN pip install --no-cache-dir poetry && \
    poetry install --no-dev

# Copy source
COPY . /app

# Entrypoint
ENTRYPOINT ["ragbench"]
```

*Image tag*: `axentx/ragbench-runner:<semver>` (automatically built on each release).  

### 8.2 Running a Benchmark (Production)  

```bash
docker run --rm \
  -v $(pwd)/config.yaml:/config.yaml:ro \
  -v $(pwd)/data:/data:ro \
  -v $(pwd)/output:/output \
  --network none \
  axentx/ragbench-runner:1.2.0 \
  run --config /config.yaml --output /output
```

*Security notes*  

* `--network none` enforces zero‑trust.  
* Volume mounts are read‑only for inputs, write‑only for output.  
* The container runs as non‑root user (`USER 1000`).  

### 8.3 CI Integration  

A typical GitHub Action step:

```yaml
- name: Run ragbench
  run: |
    docker pull axentx/ragbench-runner:${{ env.RAGBENCH_VERSION }}
    docker run --rm \
      -v ${{ github.workspace }}/benchmarks:/data:ro \
      -v ${{ github.workspace }}/results:/output \
      --network none \
      axentx/ragbench-runner:${{ env.RAGBENCH_VERSION }} \
      run --config /data/config.yaml --output /output
```

The generated `output/index.html` can be published as an artifact or uploaded to an internal dashboard.

---  

## 9. Extensibility Guidelines  

1. **Adding a Metric**  
   * Create a new Python package under `ragbench/metrics/your_metric`.  
   * Subclass `Metric` and implement `compute`.  
   * Register entry‑point in `pyproject.toml`:

   ```toml
   [project.entry-points."ragbench.metrics"]
   your_metric = "ragbench.metrics.your_metric:YourMetric"
   ```

2. **Supporting a New Model Backend**  
   * Implement `ModelClient` subclass in `ragbench/model.py`.  
   * Add a factory entry in `model_factory` mapping (`type: class`).  
   * Ensure the client respects the sandbox’s network policy (e.g., only local file access).  

3. **Custom Test Case Source**  
   * Subclass `TestCaseSource` and expose via entry‑point `ragbench.cases`.  
   * The config can then reference `source: "my_custom"`.

All extensions are automatically discovered at runtime; no core code changes required.

---  

## 10. Risks & Mitigations  

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Schema validation overhead** may dominate latency for large payloads. | Performance degradation on high‑throughput runs. | Use compiled Pydantic models; optionally enable `validate_fast=True`. |
| **Docker network mis‑configuration** could leak data. | Compliance breach. | Default `--network none`; CI step enforces the flag; unit test asserts container config. |
| **Metric plugin incompatibility** (different return types). | Benchmark run failure. | Enforce `float` return via abstract base class; add runtime type guard. |
| **SQLite encryption key leakage**. | Data at rest compromise. | Derive key from a per‑run secret stored only in environment variable; never write key to disk. |
| **Dependency drift** (e.g., vLLM breaking API). | Runtime errors. | Pin to known commit hashes; CI runs integration tests against pinned versions. |

---  

## 11. Release Checklist  

| Item | Done? |
|------|-------|
| ✅ Unit tests ≥ 90 % coverage (`pytest -q`) |
| ✅ Integration test with local HF model (vLLM) |
| ✅ Docker image builds and passes `docker run --rm` smoke test |
| ✅ Security audit: container runs non‑root, network disabled |
| ✅ Documentation: README, CLI `--help`, example config, schema examples |
| ✅ CI pipeline publishes Docker image and PyPI wheel |
| ✅ Version bump and changelog entry |

---  

**End of Technical Specification**
