# REQUIREMENTS.md

## 1. Overview
**ragbench** is a self‑hosted, zero‑trust benchmarking framework for Retrieval‑Augmented Generation (RAG) systems. It must enable developers to run reproducible, schema‑validated evaluations on‑premise while guaranteeing data sovereignty, security, and extensibility.

---

## 2. Functional Requirements

| ID | Description |
|----|-------------|
| **FR‑1** | **CLI Entry Point** – Provide a top‑level command `ragbench` with sub‑commands `run`, `list`, `validate`, and `report`. The CLI must parse a YAML/JSON configuration file and expose `--config`, `--output`, `--log-level` flags. |
| **FR‑2** | **Zero‑Trust Execution** – All benchmark runs must execute inside an isolated environment (Docker container or sandboxed subprocess) without external network access unless explicitly allowed in the config. |
| **FR‑3** | **Schema‑Driven Output Validation** – For each test case, validate the model’s JSON response against a Pydantic v2 schema defined in the test definition. Return detailed validation errors per request. |
| **FR‑4** | **Benchmark Suite Loader** – Dynamically discover benchmark suites under `benchmarks/` (each suite is a Python module exposing a `TestCase` class). Allow users to add new suites via a plug‑in directory specified in the config. |
| **FR‑5** | **Metric Computation Engine** – Compute built‑in metrics: `exact_match`, `f1_score`, `rouge_l`, `retrieval_precision@k`, and `latency_ms`. Provide a hook API (`Metric.register`) for custom metric plugins. |
| **FR‑6** | **Result Persistence** – Store run results in a JSON Lines file (`results_<timestamp>.jsonl`) and optionally in a SQLite DB (`--db-path`). Each record must contain: test_id, input, model_output, validation_status, metric scores, execution timestamps, and container metadata. |
| **FR‑7** | **Reporting Dashboard** – Generate an HTML report (`--report`) summarizing overall scores, per‑suite breakdowns, and validation failures. Include interactive tables (DataTables) and charts (Chart.js). |
| **FR‑8** | **Configuration Schema** – Validate the user‑provided config file against a Pydantic schema that includes fields: `model_endpoint`, `auth_token`, `allowed_networks`, `benchmarks`, `metrics`, `output_dir`, `container_image`. |
| **FR‑9** | **Authentication & Secrets Handling** – Support loading secrets from environment variables, `.env` files, or HashiCorp Vault (via optional plugin). Secrets must never be written to logs or result files. |
| **FR‑10** | **Testing Harness** – Provide a pytest suite covering CLI parsing, schema validation, metric calculations, and container isolation. Achieve ≥ 90 % code coverage. |
| **FR‑11** | **Versioning & Compatibility** – Expose `ragbench --version` and embed the semantic version in the result metadata. Ensure backward compatibility for config files from the previous major release (v1.x). |
| **FR‑12** | **Logging** – Emit structured JSON logs to stdout/stderr with fields: timestamp, level, component, message, request_id. Support configurable log level via `--log-level`. |
| **FR‑13** | **Graceful Shutdown** – On SIGINT/SIGTERM, allow in‑flight benchmark tasks to finish within a configurable timeout (default 30 s) before terminating containers. |
| **FR‑14** | **Resource Limits** – Allow the user to specify CPU/memory limits for the sandbox container (`cpu_quota`, `mem_limit`). The framework must enforce these limits and abort runs that exceed them, recording a `resource_exceeded` status. |

---

## 3. Non‑Functional Requirements

| ID | Category | Requirement |
|----|----------|-------------|
| **NFR‑1** | **Performance** | Average latency measurement per request must be recorded with ±5 ms accuracy. Whole‑suite execution time must not exceed 2× the sum of individual request latencies (allowing for container overhead). |
| **NFR‑2** | **Scalability** | The framework must support parallel execution of up to **32** concurrent test cases on a single host, configurable via `--parallelism`. |
| **NFR‑3** | **Security** | All data (inputs, model outputs, secrets) must remain on the host machine; no outbound network traffic is permitted unless explicitly whitelisted. Containers run as non‑root users with read‑only filesystem mounts for code and read‑write only on a temporary `/tmp` volume. |
| **NFR‑4** | **Reliability** | Benchmark runs must be resumable. If a run crashes, a `--resume` flag should pick up from the last successfully persisted result line. |
| **NFR‑5** | **Compliance** | The solution must be compatible with GDPR and CCPA data‑processing requirements: no personal data is logged, and all persisted results can be encrypted at rest via an optional `--encryption-key` flag. |
| **NFR‑6** | **Portability** | The core library must run on Linux (kernel ≥ 5.4) and macOS (≥ 12). Docker‑based isolation must be optional; a pure‑Python sandbox fallback using `multiprocessing` is required for environments without Docker. |
| **NFR‑7** | **Maintainability** | Codebase must follow PEP 8, use type hints throughout, and include docstrings for all public objects. CI pipeline must run `black`, `ruff`, and `mypy` on every PR. |
| **NFR‑8** | **Usability** | CLI help (`ragbench --help`) must display examples for each sub‑command and exit with status 0. Errors must exit with non‑zero codes and provide a machine‑readable JSON error payload (`--json-errors`). |
| **NFR‑9** | **Observability** | Emit Prometheus‑compatible metrics (run duration, success/failure counts, container restarts) on a local endpoint (`--metrics-port`). |
| **NFR‑10** | **Documentation** | Provide a `docs/` folder with a Sphinx site, including a quick‑start guide, configuration reference, and plugin development tutorial. The README must contain a one‑minute “Getting Started” snippet. |

---

## 4. Constraints

1. **Language & Runtime** – Must be implemented in Python 3.11+; no external binaries other than Docker (optional).  
2. **Licensing** – All code must be released under the MIT license; third‑party dependencies must be compatible (Apache‑2.0, MIT, BSD).  
3. **Data Size** – Benchmark input datasets will not exceed **5 GB** per run; the system must stream data to avoid loading the entire set into memory.  
4. **Dependency Footprint** – Core package size (wheel) must stay under **15 MB** to facilitate easy distribution in air‑gapped environments.  
5. **Version Pinning** – Critical libraries (pydantic, click, pytest, docker) must be pinned to versions verified as CVE‑free at release time.  

---

## 5. Assumptions

| ID | Assumption |
|----|------------|
| **A‑1** | Users have Docker installed and the current user belongs to the `docker` group, or they will opt‑in to the pure‑Python sandbox mode. |
| **A‑2** | Model endpoints expose a standard HTTP/JSON API compatible with the request schema defined in each test case. |
| **A‑3** | All benchmark suites are authored in Python and follow the `TestCase` interface (methods: `load_input()`, `run(model)`, `expected_schema`). |
| **A‑4** | The host environment provides sufficient resources (CPU, RAM) to satisfy the requested parallelism and container limits. |
| **A‑5** | Secrets management plugins (e.g., Vault) will be installed separately; ragbench only provides the integration hook. |
| **A‑6** | Users will run the CLI from a terminal that supports UTF‑8; no Windows‑specific path handling is required (Linux/macOS only). |
| **A‑7** | Network latency measurements are taken from the host’s perspective; internal container networking overhead is negligible (< 2 ms). |

---

## 6. Acceptance Criteria

- All functional requirements **FR‑1 – FR‑14** are implemented and pass unit/integration tests.  
- Non‑functional thresholds (**NFR‑1 – NFR‑10**) are verified via automated performance and security test suites.  
- Documentation builds without errors and includes usage examples for each major feature.  
- The CI pipeline reports ≥ 90 % code coverage and zero linting/type errors.  
- A release candidate can be installed via `pip install .` in an air‑gapped environment and execute a full benchmark run on a sample config without external network access.  

--- 

*Document version: 1.0 – 2026‑06‑11*
