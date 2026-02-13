# BMA SBA Assessment Model

## Project Overview

This repository houses the planning, design, and analysis documents for the Bermuda Monetary Authority's (BMA) Scenario-Based Approach (SBA) Assessment Model. The project's primary goal is to develop a state-of-the-art software platform that balances two critical needs: a robust, feature-rich web application for BMA analysts and a high-performance computational engine for intensive financial modeling.

To achieve this, the project has adopted a **hybrid microservice architecture**. The application layer, including the user interface, data validation, and reporting, will be built using **Python**, leveraging its mature and productive ecosystem. The computationally intensive core, responsible for cash flow projections and stochastic modeling, will be developed in **Julia** to achieve the highest level of performance. This dual approach ensures both rapid application development and world-class computational speed, positioning the BMA to effectively assess insurance group solvency and conduct independent stress testing.

## Key Goals & Features

*   **Robust Scenario Engine:** Capable of generating and applying the nine prescribed BMA interest rate scenarios.
*   **Flexible Plugin Architecture:** An extensible scenario library allowing for the easy addition of new, non-prescribed scenarios (e.g., from other regulators like NAIC, PRA), custom extreme events (e.g., "AI bubble," pandemics), and scenarios with complex, correlated stresses. This is managed via a formal "Scenario Manifest" structure.
*   **Stochastic Modeling:** Includes a component for generating large numbers of economic scenarios to assess tail risk.
*   **Rigorous Data Validation:** Implements strict validation of submitted data from insurance companies, enforcing machine-readable data schemas (e.g., JSON Schema) to ensure completeness, accuracy, and consistency.
*   **Comprehensive Capital Assessment & Reporting:** Calculates capital requirements and generates detailed reports, dashboards, and comparative analyses.
*   **Auditability & Versioning:** Ensures all actions, scenarios, model components, and data submissions are logged, auditable, and version-controlled for reproducibility.
*   **User-Friendly Interface:** A web-based dashboard for BMA analysts to manage scenarios, analyze results, and generate reports.

## Technology Stack

The model is being developed using a modern microservices architecture that separates application logic from high-performance computation.

*   **Application Layer (Python):**
    *   **Backend Framework:** FastAPI
    *   **Key Libraries:** pandas (data manipulation), Pydantic (data validation), Jinja2/WeasyPrint (reporting).
    *   **Database:** PostgreSQL
    *   **Cloud Platform:** AWS (Kubernetes, S3, RDS)
    *   **CI/CD:** Jenkins, GitHub Actions

*   **Computational Layer (Julia):**
    *   **Core:** High-performance services for cash flow projections and stochastic modeling.
    *   **Architecture:** Internal, containerized microservices called by the Python application layer.

*   **Frontend:**
    *   React, Redux, TypeScript

## Repository Structure

This repository contains the analysis and planning documents that led to the adoption of the hybrid architecture.

*   `README.md`: This project overview.
*   `BMA_doc/`: A collection of key reference documents, including a consolidated guide to the BMA's SBA regulation, illustrative calculations, and official source documents from the BMA.
*   `Python_Implementation/`: Contains the initial planning and design documents (PRD, SDD, etc.) for building the application layer in Python.
*   `Julia_Implementation/`: Contains the initial planning and design documents for building the computational engine in Julia.
*   `Hybrid_Implementation/`: Contains documents related to the hybrid strategy, including the final `Recommendation_Analysis.md` which outlines the strategic decision to adopt the Python + Julia model.
*   `Archive/`: Contains older summary documents that have been superseded by the materials in `BMA_doc/`.

## Current Status

The project has completed a crucial technology evaluation phase. After analyzing pure Python, pure Julia, and hybrid approaches, the decision has been made to proceed with a **hybrid Python/Julia microservice architecture**. The foundational planning and design documents for the separate components are in place.

## Next Steps

*   Commence development of the core Python/FastAPI microservices for the main application layer.
*   Begin implementation of the Julia microservices for the computational core.
*   Define and build the internal API for communication between the Python and Julia services.
*   Develop robust data ingestion and validation mechanisms in the Python layer.
*   Build the user interface and reporting capabilities.
*   Execute the defined test plans for each component to ensure quality and compliance.
