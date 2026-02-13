# Software Design Document (SDD): BMA SBA Assessment Model (Julia Version)

## 1. Introduction

### 1.1. Purpose

This document provides a detailed technical design for the BMA SBA Assessment Model, adapted for a Julia-based technology stack. It describes the system architecture, module design, data design, and the technology stack that will be used to build the model. This document is intended for the development team and will serve as a guide for the implementation of the system.

### 1.2. Scope

This document covers the design of all components of the BMA SBA Assessment Model as defined in the Product Requirements Document (PRD).

## 2. System Architecture

A microservices-based architecture is proposed for the BMA SBA Assessment Model. This architecture will provide the required scalability, flexibility, and maintainability. Each major component of the system will be implemented as a separate service, with well-defined APIs for communication between services.

The system will be deployed on a cloud platform (e.g., AWS, Azure, or Google Cloud) to ensure scalability and high availability.

[System Architecture Diagram - *To be hosted internally. This diagram typically depicts the microservices (Core Scenario Engine, Scenario Library, Stochastic Modeling, Data Input, Cash Flow Projection, Capital Assessment, UI) communicating via APIs, with shared database and cloud infrastructure components.*]

## 3. Module Design

### 3.1. Core Scenario Engine

*   **Description:** This service will be responsible for generating the prescribed scenarios from the BMA rules.
*   **Technology:** Julia, using libraries like `DataFrames.jl` for data manipulation.
*   **API:** A RESTful API that allows other services to request the generation of scenarios.

### 3.2. Scenario Library and Plugin Architecture

*   **Description:** A service that manages a library of both prescribed and non-prescribed scenarios. It will expose an API for adding new scenarios as plugins, primarily through a formal "Scenario Manifest" structure. Each plugin will consist of a folder containing:
    *   **`scenario.yaml` (The Manifest):** A metadata file defining the scenario's ID, name, description, type (e.g., 'Prescribed', 'Regulatory', 'Extreme Event'), and the specific `effects` it has on model parameters (e.g., `equity_shock`, `interest_rate_shock`, `credit_spread_shock`). This YAML file will define the shock parameters or point to external data/logic.
    *   **`data_files/` (Optional):** A sub-folder for CSVs or other files containing time series data relevant to the scenario (e.g., historical index levels for an "AI Bubble" scenario).
    *   **`logic.jl` (Optional):** A Julia script for scenarios that require complex, programmatic logic beyond simple parameter shifts. This script will expose a well-defined entry point function for the scenario engine to call.
*   **Technology:** Julia with a `Genie.jl` or `HTTP.jl` framework for the API. Scenario definitions (including YAML and Julia logic) will be version-controlled and stored in the database.
*   **API:** A RESTful API for adding, retrieving, and managing scenarios, including uploading new scenario plugin packages.

### 3.3. Stochastic Modeling Component

*   **Description:** A service for running stochastic simulations. This service will be designed to be highly scalable to handle a large number of simulations.
*   **Technology:** Julia with libraries like `Distributions.jl` for statistical modeling. For performance, we will leverage Julia's built-in distributed computing capabilities (`Distributed.jl`).
*   **API:** An API to trigger simulation runs with specified parameters and retrieve results.

### 3.4. Data Input and Validation Module

*   **Description:** A service that handles the intake and validation of data from insurance companies.
*   **Technology:** A web service built with Julia (`Genie.jl`/`HTTP.jl`) that handles file uploads and data parsing. Data validation rules will be rigorously implemented using machine-readable data schemas (e.g., JSON Schema) in conjunction with Julia's strong type system.
*   **API:** A RESTful API for uploading data files and getting validation results.

### 3.5. Cash Flow Projection Module

*   **Description:** A service that takes scenario and company data as input and projects asset and liability cash flows. This is the core computational engine where Julia's performance will be critical.
*   **Technology:** Julia, leveraging its native high-performance array computations and `DataFrames.jl`.
*   **API:** An internal API to be called by the Capital Assessment module.

### 3.6. Capital Assessment and Reporting Module

*   **Description:** A service that orchestrates the overall assessment process. It will call other services to get scenario data, run cash flow projections, and then calculate capital requirements. It will also generate reports.
*   **Technology:** Julia. Report generation can use libraries for templating and PDF generation.
*   **API:** A RESTful API for initiating assessments and retrieving reports.

### 3.7. User Interface (Web-based dashboard)

*   **Description:** A single-page application (SPA) that provides the user interface for the BMA analysts. This component remains unchanged from the Python-stack proposal.
*   **Technology:** React with a state management library like Redux. The UI will communicate with the backend services via their REST APIs.

## 4. Data Design

### 4.1. Database Schema

A PostgreSQL database will be used to store the application's data. The schema will be designed to be normalized and will include tables for:

*   Users and Roles
*   Insurance Companies
*   SBA Submissions (including submitted data files, with `version` and `approval_status` fields to track changes and approval status)
*   Scenarios and Scenario Parameters (with `version` and `approval_status` fields for each scenario definition)
*   Model Runs and Results (with explicit linkage to the `version` of the scenario, data submission, and core model components used for reproducibility)
*   Audit Logs

### 4.2. Data Models

Data models for key entities like `Scenario`, `Submission`, and `ModelRun` will be defined in the application code. This can be handled with Julia structs and a query builder like `FunSQL.jl` or direct database interaction libraries.

### 4.3. Data Flow

[Data Flow Diagram - *To be hosted internally. This diagram typically illustrates the movement of data from external insurance groups (via the Data Input/Validation module) through the various microservices (Scenario Library, Core Scenario Engine, Cash Flow Projection, Stochastic Modeling, Capital Assessment) to the database and finally to the UI/Reporting module.*]

## 5. Technology Stack

*   **Backend:** Julia 1.9+, Genie.jl / HTTP.jl
*   **Frontend:** React, Redux, TypeScript
*   **Database:** PostgreSQL
*   **Cloud Platform:** AWS (using services like EKS for Kubernetes, S3 for storage, and RDS for the database)
*   **CI/CD:** Jenkins, GitHub Actions

## 6. Security Design

*   **Authentication and Authorization:** User authentication will be handled using OAuth 2.0 and OpenID Connect. Role-based access control (RBAC) will be implemented to restrict access to different parts of the system.
*   **Data Encryption:** All data at rest will be encrypted using industry-standard encryption algorithms. Data in transit will be encrypted using TLS.
*   **Vulnerability Scanning:** The application code and its dependencies will be regularly scanned for security vulnerabilities.

## 7. Deployment and Operations

*   **Containerization:** All services will be containerized using Docker.
*   **Orchestration:** Kubernetes will be used to orchestrate the deployment and scaling of the services.
*   **Monitoring:** The system will be monitored using a combination of tools like Prometheus for metrics, Grafana for dashboards, and the ELK stack (Elasticsearch, Logstash, Kibana) for logging.
