# Product Requirements Document (PRD)
## Project: ragbench

### Problem Statement

Developers working with Retrieval-Augmented Generation (RAG) systems face challenges in evaluating their models' performance and ensuring data sovereignty. Current evaluation tools often rely on cloud-based services, compromising data security and compliance. Additionally, model outputs may not meet schema requirements, leading to downstream errors.

### Target Users

* RAG engineers and developers
* Data scientists and researchers
* Compliance and security teams
* Enterprise organizations requiring secure and private evaluation environments

### Goals

1. **Self-hosted, Zero-Trust Evaluation**: Enable developers to run benchmarks on-premises or in private clouds, ensuring data sovereignty and compliance.
2. **JSON Normalization & Validation**: Guarantee model outputs meet schema requirements, reducing downstream errors.
3. **Compliance-First Architecture**: Design a secure and private evaluation framework that meets enterprise-grade security and privacy controls.
4. **Extensible Benchmark Suite**: Provide a modular framework for adding custom evaluation metrics and test cases.
5. **Developer-Friendly CLI**: Offer a streamlined command-line interface for effortless benchmark execution and reporting.

### Key Features (Prioritized)

1. **Zero-Trust Benchmarking**: Execute evaluations in isolated, secure environments.
2. **JSON Schema Validation**: Automatically validate model outputs against defined schemas.
3. **On-Premise Execution**: Fully self-hosted for compliance-sensitive deployments.
4. **Customizable Metrics**: Extendable framework for domain-specific evaluation criteria.
5. **CLI Interface**: Simple terminal commands for running and managing benchmarks.
6. **Reporting Dashboard**: Visualize results with built-in reporting capabilities.

### Success Metrics

1. **Number of self-hosted deployments**: Track the number of organizations running ragbench on-premises.
2. **Schema validation accuracy**: Measure the percentage of model outputs validated against defined schemas.
3. **Custom metric adoption**: Monitor the number of custom evaluation metrics added to the framework.
4. **User satisfaction**: Collect feedback from users on the ease of use and effectiveness of the CLI interface.
5. **Reporting dashboard engagement**: Track user interaction with the reporting dashboard.

### Scope

* Develop and maintain the ragbench framework, including the CLI interface and reporting dashboard.
* Provide documentation and support for users.
* Continuously update and improve the framework based on user feedback and emerging requirements.

### Out-of-Scope

* Developing and maintaining RAG models or datasets.
* Providing cloud-based evaluation services.
* Integrating with third-party tools or services not explicitly required for the framework's operation.
* Conducting extensive security audits or penetration testing (beyond standard security best practices).
