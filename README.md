# BMA SBA Assessment Model

## Project Overview

This repository houses the foundational planning and design documents for the Bermuda Monetary Authority's (BMA) Scenario-Based Approach (SBA) Assessment Model. The primary goal of this project is to develop a state-of-the-art, robust, and flexible software tool that will enable the BMA to effectively assess the soundness and compliance of insurance groups' SBA implementations, as well as to conduct its own independent stress testing and scenario analysis.

The model is designed to be the BMA's definitive tool for ensuring financial stability within the Bermudian insurance market by providing comprehensive insights into insurers' liability valuations and capital requirements under various economic and market conditions.

## Key Goals & Features

*   **Robust Scenario Engine:** Capable of generating and applying the nine prescribed BMA interest rate scenarios.
*   **Flexible Plugin Architecture:** An extensible scenario library allowing for the easy addition of new, non-prescribed scenarios (e.g., from other regulators like NAIC, PRA), custom extreme events (e.g., "AI bubble," pandemics), and scenarios with complex, correlated stresses. This is managed via a formal "Scenario Manifest" structure.
*   **Stochastic Modeling:** Includes a component for generating large numbers of economic scenarios to assess tail risk.
*   **Rigorous Data Validation:** Implements strict validation of submitted data from insurance companies, enforcing machine-readable data schemas (e.g., JSON Schema) to ensure completeness, accuracy, and consistency.
*   **Comprehensive Capital Assessment & Reporting:** Calculates capital requirements and generates detailed reports, dashboards, and comparative analyses.
*   **Auditability & Versioning:** Ensures all actions, scenarios, model components, and data submissions are logged, auditable, and version-controlled for reproducibility.
*   **User-Friendly Interface:** A web-based dashboard for BMA analysts to manage scenarios, analyze results, and generate reports.

## Technology Stack

The model is being developed using a modern microservices architecture with the following core technologies:

*   **Backend:** Python 3.9+ (Flask/FastAPI)
*   **Frontend:** React, Redux, TypeScript
*   **Database:** PostgreSQL
*   **Cloud Platform:** AWS (with Kubernetes for orchestration, S3 for storage, RDS for database)
*   **CI/CD:** Jenkins, GitHub Actions
*   **Key Libraries:** NumPy, pandas (for calculations); scipy.stats (for stochastic modeling); Cerberus/Pydantic (for data validation); Jinja2, WeasyPrint (for reporting).

## Repository Structure

*   `README.md`: This overview of the project.
*   `BMA_SBA_Consolidated_Guide.md`: A comprehensive guide to the BMA's Scenario-Based Approach (SBA) regulation, synthesizing information from official BMA documents.
*   `Model_doc/`: Contains detailed planning and design documents for the SBA Assessment Model:
    *   `BMA_SBA_Model_PRD.md`: Product Requirements Document
    *   `BMA_SBA_Model_SDD.md`: Software Design Document
    *   `BMA_SBA_Model_Project_Plan.md`: Project Plan
    *   `BMA_SBA_Model_Test_Plan.md`: Test Plan

## Current Status

The project is in the detailed planning and design phase. All foundational requirements, software design, project management, and testing strategies have been documented and refined. The next stage involves the commencement of model building and implementation based on these approved plans.

## Next Steps

*   Commence development of the core microservices.
*   Implement the scenario plugin architecture.
*   Develop robust data ingestion and validation mechanisms.
*   Build the user interface and reporting capabilities.
*   Execute the defined test plan to ensure quality and compliance.
