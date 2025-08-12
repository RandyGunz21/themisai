# Project ThemisAI: System Design & Architecture

**Version:** 2.1
**Last Updated:** August 12, 2025

## Table of Contents

1.  [Project Overview](#1-project-overview)
    *   [Vision](#vision)
    *   [Core Features](#core-features)
2.  [Python Package Structure](#2-python-package-structure)
    *   [Monorepo Layout](#monorepo-layout)
    *   [Individual Service Structure](#individual-service-structure)
3.  [System Architecture](#3-system-architecture)
    *   [High-Level Diagram](#high-level-diagram)
    *   [Architectural Principles](#architectural-principles)
4.  [Technology Stack](#4-technology-stack)
5.  [Core Components (Microservices)](#5-core-components-microservices)
    *   [I/O Agent (Frontend Gateway)](#io-agent-frontend-gateway)
    *   [Supervisor Agent (Central Orchestrator)](#supervisor-agent-central-orchestrator)
    *   [Legal Researcher Agent](#legal-researcher-agent)
    *   [Legal Documents Drafter Agent](#legal-documents-drafter-agent)
    *   [Legal Documents Reviewer Agent](#legal-documents-reviewer-agent)
6.  [Data Flow & Kafka Topics](#6-data-flow--kafka-topics)
7.  [The Advanced RAG Pipeline in Detail](#7-the-advanced-rag-pipeline-in-detail)
8.  [Real-time User Experience (UX) Flow](#8-real-time-user-experience-ux-flow)
9.  [Getting Started (Developer Guide)](#9-getting-started-developer-guide)
    *   [9.1 Local Development (using Docker Compose)](#91-local-development-using-docker-compose)
    *   [9.2 Production Deployment (using Kubernetes)](#92-production-deployment-using-kubernetes)

---

## 1. Project Overview

### Vision

Project **ThemisAI** is an initiative to build a state-of-the-art, AI-powered web application that serves as an intelligent legal assistant. Our vision is to democratize access to legal knowledge by providing intuitive, accurate, and instant tools for legal research, document drafting, and compliance review.

### Core Features

1.  **Legal Research:** Empowers users to pose complex legal questions in natural language and receive precise answers, substantiated by citations from valid and relevant legal sources.
2.  **Legal Document Drafting:** Automatically generates drafts of common legal documents (e.g., powers of attorney, non-disclosure agreements) by synthesizing user requirements and contextual legal research.
3.  **Legal Document Review:** Analyzes user-provided legal documents to assess compliance with current laws, identify potential risks or ambiguities, and offer actionable recommendations.

---

## 2. Python Package Structure

To ensure the project is maintainable, scalable, and provides a consistent developer experience, we will adopt a structured monorepo layout. This structure clearly separates concerns and standardizes the architecture of each microservice.

### Monorepo Layout

The root of the repository is organized to separate frontend code, backend services, and infrastructure configurations, facilitating independent development and deployment pipelines.

```plaintext
themisai/
├── .github/                  # CI/CD workflows (e.g., GitHub Actions)
├── services/                 # Parent directory for all backend microservices
│   ├── io_agent/             # I/O Agent microservice package
│   ├── supervisor_agent/     # Supervisor Agent microservice package
│   ├── researcher_agent/     # Researcher Agent microservice package
│   └── ...                   # Other agent services
├── frontend/                 # The Next.js application
├── infra/                    # Infrastructure as Code (IaC)
│   ├── docker-compose.yml    # For local development
│   └── helm/                 # Helm charts for Kubernetes deployment
│       └── themisai/
├── .env.example              # Example environment variables
├── .gitignore
└── README.md                 # This file
```

### Individual Service Structure

Each Python microservice within the `services/` directory will adhere to the following standardized structure, managed with `Poetry`. This consistency is key to reducing cognitive load and accelerating development.

The structure for `supervisor_agent` serves as the template:

```plaintext
services/supervisor_agent/
├── app/                      # Main application source code, importable as a package
│   ├── __init__.py
│   ├── api/                  # FastAPI endpoints and routes (if applicable)
│   │   └── __init__.py
│   ├── core/                 # Core logic, including configuration management
│   │   ├── __init__.py
│   │   └── config.py         # Pydantic settings for loading environment variables
│   ├── schemas/              # Pydantic models defining data contracts (e.g., Kafka messages)
│   │   ├── __init__.py
│   │   └── ticket.py
│   ├── services/             # Business logic handlers (e.g., Kafka consumer/producer logic)
│   │   ├── __init__.py
│   │   └── kafka_service.py
│   ├── agent/                # The core AI logic (e.g., LangGraph graph definition)
│   │   ├── __init__.py
│   │   └── graph.py
│   └── main.py               # FastAPI application entry point and lifecycle events
├── tests/                    # Unit and integration tests for the service
│   ├── __init__.py
│   └── test_kafka_service.py
├── Dockerfile                # Multi-stage Dockerfile for building a lean production image
├── poetry.lock               # Dependency lock file for reproducible builds
└── pyproject.toml            # Project metadata and dependencies (managed by Poetry)
```

#### Key Components Explained:

*   **`app/core/config.py`**: Provides a single, validated source of truth for all environment variables using Pydantic's `BaseSettings`.
*   **`app/schemas/`**: Defines the data contracts for all inputs and outputs, ensuring data integrity across the system.
*   **`app/services/`**: Encapsulates business logic decoupled from the web framework, such as processing Kafka messages.
*   **`app/agent/`**: Contains the specialized AI logic for the agent, such as the LangGraph definition or the LlamaIndex RAG pipeline.
*   **`pyproject.toml`**: The modern standard for Python project configuration, defining dependencies, metadata, and scripts for tools like `Poetry` and `pytest`.

---

## 3. System Architecture

### High-Level Diagram

```mermaid
graph TD
    subgraph "User's Browser"
        A[Next.js Frontend]
    end

    subgraph "I/O Agent (FastAPI)"
        B(API Endpoint)
        C(SSE Endpoint for Real-time Updates)
    end

    subgraph "Kafka Cluster"
        K1(ticket.new)
        K2(request.*)
        K3(result.*)
        K4(ticket.status.updates)
        K5(system.logs)
    end

    subgraph "AI Agentic System"
        D(Supervisor Agent - LangGraph)
        E(Legal Researcher Agent - LlamaIndex)
        F(Legal Drafter Agent)
        G(Legal Reviewer Agent)
    end

    subgraph "Data & External Services"
        H[PostgreSQL]
        I[Qdrant Vector DB]
        J[Cohere Rerank API]
        L[Cache - Redis]
    end

    A -- HTTP Request --> B
    B -- Publishes to --> K1
    K1 -- Consumed by --> D
    D -- Publishes to --> K2
    K2 -- Consumed by --> E
    K2 -- Consumed by --> F
    K2 -- Consumed by --> G
    E -- Publishes to --> K3
    F -- Publishes to --> K3
    G -- Publishes to --> K3
    K3 -- Consumed by --> D
    D -- Publishes to --> K4
    K4 -- Consumed by --> C
    C -- Pushes real-time status --> A
    D -- Writes state to --> H
    E -- Queries/Writes --> I
    E -- Reads from --> L
    E -- Calls API --> J
```

### Architectural Principles

Our architecture is guided by these core principles to ensure a robust and scalable system:

*   **Independent Microservices:** Each AI Agent (Supervisor, Researcher, etc.) is a **fully independent, containerized service**. This principle allows teams to develop, deploy, scale, and maintain each agent's functionality without impacting the rest of the system.
*   **Asynchronous Communication:** The system is built around an event-driven model using Apache Kafka, ensuring resilience to component failures and enabling horizontal scaling.
*   **AI-Native:** The architecture is purpose-built to support sophisticated AI workflows, featuring advanced state management with LangGraph and a high-accuracy RAG pipeline.
*   **User-Centric:** User experience is paramount. Real-time status updates are implemented to provide transparency and keep the user engaged throughout the process.

---

## 4. Technology Stack

| Category | Technology | Rationale for Selection |
| :--- | :--- | :--- |
| **Backend** | Python & FastAPI | The industry standard for AI/ML, offering high performance and first-class asynchronous support. |
| **Frontend** | Next.js (React) | A leading framework for building fast, modern, and SEO-friendly web applications. |
| **UI Library** | Shadcn/ui | A highly customizable and accessible component library for accelerating UI development. |
| **Agentic Logic** | LangGraph | Essential for orchestrating the complex, stateful, and potentially cyclical workflows of our agentic system. |
| **RAG Framework** | LlamaIndex | A comprehensive framework that simplifies and optimizes the entire RAG pipeline. |
| **Vector Database**| Qdrant | A high-performance, production-ready vector database optimized for semantic search. |
| **Reranking** | Cohere Rerank API | Delivers State-of-the-Art (SOTA) accuracy for document relevance with excellent multilingual support. |
| **Database** | PostgreSQL | A battle-tested relational database, perfect for storing structured data like tickets and user metadata. |
| **Messaging** | Apache Kafka | The definitive choice for a resilient, scalable, and high-throughput asynchronous communication backbone. |
| **Caching** | Redis | An extremely fast in-memory data store, ideal for caching frequently accessed data to reduce latency. |

---

## 5. Core Components (Microservices)

The ThemisAI backend is composed of the following specialized and **independent microservices**. Each service has a clearly defined responsibility and communicates with others via the Kafka message bus.

### I/O Agent (Frontend Gateway)

*   **Purpose:** To act as the primary interface between the user and the complex backend system.
*   **Key Responsibilities:**
    1.  Serves the responsive user interface built with Next.js and Shadcn/ui.
    2.  Receives and validates user requests through its secure FastAPI backend.
    3.  Generates a unique `Ticket ID` and initiates a workflow by publishing a task to the `ticket.new` Kafka topic.
    4.  Exposes a **Server-Sent Events (SSE)** endpoint to stream real-time status updates to the frontend.
    5.  Presents the final, synthesized answer to the user in a clear and structured format.

### Supervisor Agent (Central Orchestrator)

*   **Purpose:** To serve as the "brain" of the operation, managing the entire lifecycle of a user request.
*   **Key Responsibilities:**
    1.  Utilizes **LangGraph** to model the workflow as a stateful graph, allowing for complex logic, retries, and cycles.
    2.  Consumes new tasks from `ticket.new` and results from all `result.*` topics.
    3.  Decomposes user problems into discrete tasks and delegates them to the appropriate specialized agents.
    4.  Publishes user-friendly status updates to the `ticket.status.updates` topic at each significant state transition.
    5.  Synthesizes findings from all agents into a single, coherent, and comprehensive final response.

### Legal Researcher Agent

*   **Purpose:** To perform deep, accurate, and citation-backed legal research.
*   **Key Responsibilities:**
    1.  Consumes research tasks from the `request.research` topic.
    2.  Executes an advanced RAG pipeline using **LlamaIndex** (detailed in Section 7).
    3.  Retrieves document candidates from the **Qdrant** vector database.
    4.  Leverages the **Cohere Rerank API** to ensure the highest possible relevance of source materials.
    5.  Generates concise summaries and extracts key insights from the top-ranked documents.
    6.  Publishes its structured findings to the `result.research` topic.

### Legal Documents Drafter Agent

*   **Purpose:** To generate well-structured drafts of legal documents.
*   **Key Responsibilities:**
    1.  Consumes drafting tasks from the `request.drafting` topic.
    2.  Receives a rich context from the Supervisor, including user specifications and relevant legal research.
    3.  Utilizes a powerful LLM to generate the document draft according to the provided instructions.
    4.  Publishes the final document to the `result.drafting` topic.

### Legal Documents Reviewer Agent

*   **Purpose:** To analyze and provide feedback on existing legal documents.
*   **Key Responsibilities:**
    1.  Consumes review tasks from the `request.review` topic.
    2.  Analyzes the content and structure of user-uploaded documents.
    3.  Can initiate a research sub-task via the Supervisor to verify specific clauses against current law.
    4.  Publishes a detailed analysis, highlighting potential risks and recommendations, to the `result.review` topic.

---

## 6. Data Flow & Kafka Topics

| Topic Name | Example Message Payload | Producer | Consumers |
| :--- | :--- | :--- | :--- |
| `ticket.new` | `{"ticket_id": "...", "user_query": "..."}` | I/O Agent | Supervisor Agent |
| `request.research`| `{"ticket_id": "...", "research_task": "..."}` | Supervisor Agent | Legal Researcher |
| `result.research` | `{"ticket_id": "...", "summary": "...", "sources": [...]}` | Legal Researcher | Supervisor Agent |
| `ticket.status.updates`| `{"ticket_id": "...", "status_message": "..."}` | Supervisor Agent | I/O Agent (SSE) |
| `system.logs` | `{"timestamp": "...", "service": "...", "level": "..."}` | All Agents | Logging Service (ELK/Loki) |

---

## 7. The Advanced RAG Pipeline in Detail

Our research pipeline is designed for maximum accuracy by combining vector search and reranking.

```mermaid
sequenceDiagram
    participant RA as Legal Researcher Agent
    participant Q as Qdrant
    participant C as Cohere Rerank API
    participant LLM as Large Language Model

    RA->>Q: 1. Retrieve initial documents (e.g., top 25) based on query vector
    Q-->>RA: Returns 25 candidate documents
    RA->>C: 2. Send query and 25 documents to Rerank API
    C-->>RA: Returns re-ordered list of documents by relevance
    RA->>RA: 3. Select top 5 most relevant documents from Cohere's response
    RA->>LLM: 4. Generate answer using query and the top 5 re-ranked documents as context
    LLM-->>RA: Returns final, accurate answer
```

1.  **Initial Retrieval:** Fetches a large number of candidate documents from Qdrant to ensure no critical information is missed.
2.  **Reranking Layer:** Uses Cohere's advanced model to filter out "noise" and identify the most semantically relevant documents from the initial candidates.
3.  **Context Augmentation:** Only the most relevant documents post-reranking are used as context for the LLM.
4.  **Generation:** The LLM produces a much more accurate and focused answer because its context is clean and highly relevant.

---

## 8. Real-time User Experience (UX) Flow

We ensure the user never feels left in the dark during a complex background process.

1.  **User Submits Query:** The user types a question and hits "Enter."
2.  **UI Responds Instantly:** The Next.js frontend immediately opens an SSE connection to the I/O Agent's backend and displays an initial status like "Connecting...".
3.  **Supervisor Publishes Status:** As the Supervisor Agent begins the workflow in LangGraph, it publishes the first message to `ticket.status.updates`, e.g., `{"status_message": "✅ Request received. Beginning analysis..."}`.
4.  **Status Streamed to UI:** The I/O Agent's backend receives this message from Kafka and instantly pushes it through the SSE connection to the user's browser. The UI updates dynamically.
5.  **Continuous Updates:** This process continues for every significant step:
    *   `⏳ Researching relevant regulations...`
    *   `🎯 Relevant documents found. Analyzing...`
    *   `✍️ Composing final answer...`
6.  **Final Result:** Once the final answer is ready, the SSE connection can be closed, and the full result is displayed.

---

## 9. Getting Started (Developer Guide)

### 9.1 Local Development (using Docker Compose)

This setup uses Docker Compose to orchestrate the entire application stack locally. The `docker-compose.yml` file is configured to **pull official Docker images for all infrastructure components** (PostgreSQL, Kafka, Qdrant, Redis) and build images for our custom AI agent services. This ensures a consistent, one-command setup without needing to manually install any databases or message brokers.

#### Prerequisites
*   Git
*   Docker & Docker Compose
*   Python 3.10+ & Poetry
*   Node.js 18+
*   API Keys for external services (Cohere, OpenAI/Anthropic, etc.)

#### Configuration
1.  **Clone the Repository:**
    ```bash
    git clone <your-repository-url>
    cd themisai
    ```
2.  **Create Environment File:**
    Copy the example `.env.example` file to `.env`.
    ```bash
    cp .env.example .env
    ```
3.  **Edit Environment File:**
    Open the `.env` file and populate all required variables.

#### Running the System
1.  **Start All Services:**
    From the project's root directory, launch all services in detached mode.
    ```bash
    docker-compose up -d
    ```
2.  **Verify Services:**
    Check the status of all running containers.
    ```bash
    docker-compose ps
    ```
3.  **Access the Application:**
    Open your browser and navigate to `http://localhost:3000`.

4.  **Development Workflow:**
    This project adheres to the structured package layout defined in **Section 2**. When working on a specific Python service, navigate to its directory (e.g., `services/supervisor_agent`) to manage its dependencies with `poetry` and run its tests.

### 9.2 Production Deployment (using Kubernetes)

This method outlines deploying **ThemisAI** to a production-grade Kubernetes cluster using Helm.

#### Prerequisites
*   `kubectl` configured to access your Kubernetes cluster.
*   `helm` (v3+) installed.

#### Configuration
1.  **Create a Namespace:**
    Isolate the application within its own namespace.
    ```bash
    kubectl create namespace themisai
    ```
2.  **Manage Secrets:**
    Create a Kubernetes Secret to securely store all sensitive credentials.
    ```bash
    kubectl create secret generic themisai-secrets \
      --namespace themisai \
      --from-literal=POSTGRES_PASSWORD='YOUR_SECURE_PASSWORD' \
      --from-literal=COHERE_API_KEY='YOUR_COHERE_API_KEY'
    ```

#### Deployment Steps
1.  **Add Helm Repository:**
    ```bash
    helm repo add themisai-charts <url-to-your-helm-repo>
    helm repo update
    ```
2.  **Configure `values.yaml`:**
    Create a `my-values.yaml` file to customize the deployment.
    ```yaml
    # my-values.yaml
    replicaCount: 2 # Example: scale a service
    
    secrets:
      existingSecret: themisai-secrets
    
    ingress:
      enabled: true
      hosts:
        - host: themis.your-company.com
          paths: ["/"]
    ```
3.  **Install the Helm Chart:**
    Deploy the application using your custom values.
    ```bash
    helm install themisai themisai-charts/themisai \
      --namespace themisai \
      -f my-values.yaml
    ```
