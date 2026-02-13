# Software Design Document (SDD): BMA SBA Assessment Model (Hybrid Approach)

## 1. Introduction

### 1.1. Purpose

This document provides a detailed technical design for the BMA SBA Assessment Model based on a hybrid technology stack. It describes the system architecture, module design, data design, and technologies that will be used. This approach uses Python for application-level services and Julia for high-performance computational services, leveraging the strengths of both languages.

### 1.2. Scope

This document covers the design of all components of the BMA SBA Assessment Model as defined in the Product Requirements Document (PRD).

## 2. System Architecture

A microservices-based architecture is proposed. This architecture is ideal for a hybrid-language system, providing clear separation of concerns, independent scalability, and technological flexibility. Each service will communicate via well-defined RESTful APIs.

*   **Application & Orchestration Layer (Python):** Handles all external communication, user interaction, data intake, and orchestration of the modeling process.
*   **Computational Layer (Julia):** A set of specialized services that perform computationally intensive tasks, called by the Python layer.

The system will be deployed on a cloud platform (e.g., AWS, Azure, or Google Cloud) using containerization to manage the polyglot environment seamlessly.

[System Architecture Diagram - *To be hosted internally. This diagram would show a client interacting with a Python API Gateway. The gateway routes requests to various Python microservices (Data Input, Capital Assessment, etc.). The Capital Assessment service, in turn, makes API calls to separate Julia microservices (Cash Flow Projection, Stochastic Modeling).*]

## 3. Module Design

### 3.1. Data Input and Validation Module (Python Service)

*   **Description:** A service that handles the intake and validation of data from insurance companies.
*   **Technology:** **Python** with **FastAPI**. Data validation rules will be implemented using **Pydantic**.
*   **API:** A RESTful API for uploading data files and getting validation results.

### 3.2. Scenario Library and Plugin Architecture (Python Service)

*   **Description:** Manages the library of scenarios. The plugin logic itself (`logic.py`) would be executed within this Python service.
*   **Technology:** **Python** with **FastAPI**.
*   **API:** A RESTful API for adding, retrieving, and managing scenarios.

### 3.3. Capital Assessment and Reporting Module (Python Service)

*   **Description:** The central orchestrator. It receives a request to run an assessment, calls the Data Input and Scenario services, then calls the Julia computational services with the prepared data, receives the results, and generates the final reports.
*   **Technology:** **Python** with **FastAPI**. Report generation will use libraries like `Jinja2` and `WeasyPrint`.
*   **API:** The primary RESTful API for initiating assessments and retrieving reports.

### 3.4. Cash Flow Projection Module (Julia Service)

*   **Description:** A dedicated, high-performance service that takes scenario and company data as input and projects asset and liability cash flows.
*   **Technology:** **Julia**, using its native array computations and `DataFrames.jl`. The service will be exposed via a lightweight web framework like **HTTP.jl**.
*   **API:** An internal RESTful API that accepts projection parameters and data, and returns the resulting cash flows. This API is called by the Python Capital Assessment module.

### 3.5. Stochastic Modeling Component (Julia Service)

*   **Description:** A dedicated service for running a large number of stochastic simulations.
*   **Technology:** **Julia**, using libraries like `Distributions.jl` and built-in `Distributed.jl` for parallel processing. Exposed via **HTTP.jl**.
*   **API:** An internal API to trigger simulation runs and retrieve results, called by the Python Capital Assessment module.

### 3.6. User Interface (Web-based dashboard)

*   **Description:** A single-page application (SPA) that provides the user interface for the BMA analysts. It interacts exclusively with the Python backend.
*   **Technology:** React with a state management library like Redux.

## 4. Data Design

The data design remains the same as in the other proposals, with a central PostgreSQL database. The key difference is that only the Python services will have direct access to the database. The Julia services are treated as pure computational engines; they receive all necessary data via API calls and return their results, without a direct database connection.

*   **Database:** PostgreSQL
*   **ORM (Python):** SQLAlchemy

## 5. Technology Stack

*   **Orchestration & Application Layer:** Python 3.9+, FastAPI
*   **Computational Layer:** Julia 1.9+
*   **Frontend:** React, Redux, TypeScript
*   **Database:** PostgreSQL
*   **Cloud Platform & Deployment:** Docker, Kubernetes (e.g., AWS EKS), S3, RDS
*   **CI/CD:** GitHub Actions (with parallel workflows for Python and Julia testing/building)

## 6. Security Design

Security is handled at the Python application layer, which acts as the gateway to the entire system.

*   **Authentication and Authorization:** Handled by the Python services using OAuth 2.0 and RBAC. The internal Julia services will be on a private network, only accessible by the trusted Python services.
*   **Data Encryption:** TLS for all data in transit (including between Python and Julia services). Encryption at rest for the database.

## 7. Deployment and Operations

*   **Containerization:** All services (both Python and Julia) will be containerized using Docker. This is key to managing a polyglot environment effectively.
*   **Orchestration:** Kubernetes will be used to deploy and manage the heterogeneous containers as a single, cohesive application.
