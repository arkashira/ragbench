<h3 align="center">🔍 ragbench</h3>

<div align="center">

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Language](https://img.shields.io/badge/language-Python-blue.svg)](https://www.python.org/)
[![Build](https://img.shields.io/badge/build-passing-green.svg)](https://github.com/axentx/ragbench/actions)
[![Stars](https://img.shields.io/github/stars/axentx/ragbench.svg)](https://github.com/axentx/ragbench/stargazers)

</div>

---

# 🚀 ragbench

**Empower developers with zero-trust RAG evaluation tools that ensure data sovereignty and model output correctness.**

## Why ragbench?

- **Self-hosted, Zero-Trust Evaluation**: Run benchmarks on-premises or in private clouds, ensuring data sovereignty and compliance.
- **JSON Normalization & Validation**: Guarantees model outputs meet schema requirements, reducing downstream errors.
- **Built for RAG Engineers**: Designed for teams evaluating Retrieval-Augmented Generation systems in secure environments.
- **Schema-Driven Testing**: Enforces strict JSON schemas for consistent and reliable model responses.
- **Compliance-First Architecture**: Built with enterprise-grade security and privacy controls from the ground up.
- **Extensible Benchmark Suite**: Modular framework for adding custom evaluation metrics and test cases.
- **Developer-Friendly CLI**: Streamlined command-line interface for effortless benchmark execution and reporting.

## Feature Overview

| Feature                     | Description                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| Zero-Trust Benchmarking    | Execute evaluations in isolated, secure environments                        |
| JSON Schema Validation     | Automatically validate model outputs against defined schemas                |
| On-Premise Execution       | Fully self-hosted for compliance-sensitive deployments                        |
| Customizable Metrics       | Extendable framework for domain-specific evaluation criteria                  |
| CLI Interface              | Simple terminal commands for running and managing benchmarks                  |
| Reporting Dashboard        | Visualize results with built-in reporting capabilities                        |

## Tech Stack

- Python 3.11+
- Pydantic v2 for schema validation
- Click for CLI interface
- pytest for testing
- Docker for containerization (if applicable)

## Project Structure

```
.
├── business/
│   └── README.md
├── README.md
└── ...
```

## Getting Started

### Prerequisites

Ensure you have Python 3.11 or higher installed.

```bash
python --version
```

### Installation

Clone the repository and install dependencies:

```bash
git clone https://github.com/axentx/ragbench.git
cd ragbench
pip install -e .
```

### Running Benchmarks

Execute a basic benchmark using the CLI:

```bash
ragbench run --config ./examples/config.yaml
```

### Testing

Run unit tests:

```bash
pytest tests/
```

## Deploy

To deploy `ragbench`, containerize it using Docker:

```bash
docker build -t ragbench .
docker run ragbench
```

## Status

✅ Active Development  
Latest commit: `2149907` — Initial commit

## Contributing

We welcome contributions! Please see our [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.