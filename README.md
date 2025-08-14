# Project ThemisAI: System Design & Architecture

**Version:** 2.7
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
    *   [I/O Agent (Conversational Gateway)](#io-agent-conversational-gateway)
    *   [Supervisor Agent (High-Level Orchestrator)](#supervisor-agent-high-level-orchestrator)
    *   [Legal Researcher Agent (Reasoning & Research)](#legal-researcher-agent-reasoning--research)
    *   [Legal Documents Drafter Agent (Structured Drafting)](#legal-documents-drafter-agent-structured-drafting)
    *   [Legal Documents Reviewer Agent (Analytical Review)](#legal-documents-reviewer-agent-analytical-review)
6.  [Data Flow & Kafka Topics](#6-data-flow--kafka-topics)
7.  [The Advanced RAG Pipeline in Detail](#7-the-advanced-rag-pipeline-in-detail)
8.  [Real-time User Experience (UX) Flow](#8-real-time-user-experience-ux-flow)
9.  [Getting Started (Developer Guide)](#9-getting-started-developer-guide)
    *   [9.1 Local Development (using Docker Compose)](#91-local-development-using-docker-compose)
    *   [9.2 Production Deployment (using Google Kubernetes Engine - GKE)](#92-production-deployment-using-google-kubernetes-engine---gke)

---

## 1. Project Overview

### Vision

Project **ThemisAI** is an initiative to build a state-of-the-art, AI-powered web application that serves as an intelligent legal assistant. Our vision is to democratize access to legal knowledge by providing a guided, conversational, and accurate platform for legal research, document drafting, and compliance review within its defined domains.

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
*   **`app/agent/graph.py`**: **Crucially, this file defines the internal reasoning graph for each agent using LangGraph.**
*   **`pyproject.toml`**: The modern standard for Python project configuration, defining dependencies, metadata, and scripts for tools like `Poetry` and `pytest`.

---

## 3. System Architecture

### High-Level Diagram

```mermaid
graph TD
    subgraph "User's Browser"
        A[Next.js Frontend]
    end

    subgraph "I/O Agent (Conversational Gateway)"
        B(API Endpoint)
        LLM_IO[Conversational LLM]
        Tool(Tool: create_legal_ticket)
    end

    subgraph "Kafka Cluster"
        K1(ticket.new)
    end

    subgraph "Backend Agentic System"
        D(Supervisor Agent)
        E(Researcher Agent)
        F(Drafter Agent)
        G(Reviewer Agent)
    end

    subgraph "Data & External Services"
        H[PostgreSQL]
        I[Qdrant Vector DB]
        J[Cohere Rerank API]
        L[Cache - Redis]
        M[MinIO Object Storage]
    end

    A <--> B
    B -- "User Input" --> LLM_IO
    LLM_IO -- "Decides Action" --> B
    
    subgraph "Decision Flow"
        direction LR
        LLM_IO -- "If Legal & In-Scope" --> Tool
        LLM_IO -- "If Non-Legal/Chat" --> B
    end

    Tool -- "Publishes Payload" --> K1
    B -- "Conversational Reply" --> A
    K1 --> D
    D -- "Delegates Task" --> E & F & G

    A -- "Uploads/Downloads File" <--> M
    G -- "Reads File" --> M
    F -- "Writes File" --> M
```

### Architectural Principles

*   **Distributed Reasoning with LangGraph:** Every backend AI agent employs its own internal LangGraph to perform complex, multi-step tasks with internal logic, state management, and self-correction loops.
*   **Conversational Intake & Tool-Based Handoff:** The system begins with a sophisticated conversational agent that uses an LLM to understand intent and only triggers the main backend workflow by calling a specific "tool" when a valid, in-scope legal request is identified.
*   **Independent Microservices:** Each AI Agent and infrastructure component (like MinIO) is a **fully independent, containerized service**.
*   **Asynchronous Communication:** The system is built around an event-driven model using Apache Kafka, ensuring resilience and scalability.

---

## 4. Technology Stack

| Category | Technology | Rationale for Selection |
| :--- | :--- | :--- |
| **Large Language Model**| **GPT-5** (or equivalent SOTA model) | Chosen for its superior reasoning, instruction-following, and tool-calling capabilities. |
| **Object Storage** | **MinIO** | A high-performance, S3-compatible object storage server, self-hosted on our Kubernetes cluster for full data control. |
| **Backend Runtime** | Python & FastAPI | The industry standard for AI/ML, offering high performance and first-class asynchronous support. |
| **Frontend Runtime**| Bun | A modern, all-in-one JavaScript runtime and toolkit chosen for its exceptional performance. |
| **Frontend Framework**| Next.js (React) | A leading framework for building fast, modern, and SEO-friendly web applications. |
| **UI Library** | Shadcn/ui | A highly customizable and accessible component library for accelerating UI development. |
| **Core Agentic Logic**| LangGraph | The core framework for implementing the internal state machines and reasoning graphs for all backend AI agents. |
| **RAG Framework** | LlamaIndex | A comprehensive framework that simplifies and optimizes the entire RAG pipeline. |
| **Vector Database**| Qdrant | A high-performance, production-ready vector database optimized for semantic search. |
| **Reranking** | Cohere Rerank API | Delivers State-of-the-Art (SOTA) accuracy for document relevance with excellent multilingual support. |
| **Database** | PostgreSQL | A battle-tested relational database for storing structured data. |
| **Messaging** | Apache Kafka | The definitive choice for a resilient, scalable, and high-throughput asynchronous communication backbone. |
| **Caching** | Redis | An extremely fast in-memory data store for reducing data access latency. |

---

## 5. Core Components (Microservices)

The introduction of MinIO refines the responsibilities of several agents, particularly regarding file handling.

### I/O Agent (Conversational Gateway)

*   **Purpose:** To act as the intelligent, conversational "front door" and manage file access credentials.
*   **Refined File Handling Logic:**
    *   **For Uploads (Review):** When a user wants to upload a document, the I/O Agent requests a **pre-signed upload URL** from MinIO and sends it to the frontend. The frontend uploads the file directly to MinIO. The I/O Agent then includes the resulting object path in the Kafka message for the `create_legal_ticket` tool.
    *   **For Downloads (Drafting):** When a document is ready, the Supervisor Agent notifies the I/O Agent. The I/O Agent generates a **pre-signed download URL** from MinIO and presents this secure, temporary link to the user.

### Supervisor Agent (High-Level Orchestrator)

*   **Purpose:** To manage the high-level lifecycle of a user request after it has been ticketed.
*   **LangGraph Implementation:** Its graph, powered by **GPT-5's** reasoning, is responsible for macro-level orchestration. It delegates complex micro-tasks to the specialized agents.

### Legal Researcher Agent (Reasoning & Research)

*   **Purpose:** To perform deep, accurate, and citation-backed legal research as a multi-step process.
*   **LangGraph Implementation:** It uses its own internal LangGraph to manage a sophisticated research process. **GPT-5** enables complex query decomposition and self-critique loops for higher accuracy.

### Legal Documents Drafter Agent (Structured Drafting)

*   **Purpose:** To generate well-structured drafts of legal documents.
*   **Refined File Handling Logic:** After its internal LangGraph completes the drafting process, the agent's final step is to **write the generated document (e.g., as a `.docx` or `.pdf`) directly to a specified bucket in MinIO**. It then reports the object path and success status back to the Supervisor.

### Legal Documents Reviewer Agent (Analytical Review)

*   **Purpose:** To analyze and provide feedback on existing legal documents.
*   **Refined File Handling Logic:** The agent receives the object path of the user's uploaded document from the Kafka message. Its first step is to **use this path to download the document directly from MinIO** before beginning its LangGraph-driven review process.

---

## 6. Data Flow & Kafka Topics

| Topic Name | Example Message Payload | Producer | Consumers |
| :--- | :--- | :--- | :--- |
| `ticket.new` | `{"ticket_id": "...", "user_query": "...", "file_path": "..."}` | I/O Agent (via Tool Call) | Supervisor Agent |
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

The user's journey is designed for clarity and transparency.

1.  **Conversational Intake:** The user starts a conversation. The UI is a standard chat interface.
2.  **Initial Interaction:** The I/O Agent engages in conversation. If the user asks "What can you do?", it explains its capabilities. If the user says "Hello", it responds politely. This phase is fully handled by the I/O Agent.
3.  **Handoff Point:** The user asks a valid legal question like, "Can you draft an NDA for a software project?"
4.  **Tool Call & Confirmation:** The I/O Agent recognizes this as an in-scope task. The UI might show a brief "Thinking..." indicator. Then, the agent responds: "Certainly. I can help with that. I'm creating a ticket to begin the drafting process. You'll see live progress below."
5.  **Workflow Updates:** The UI now switches to receiving SSE updates, showing the backend progress:
    *   `✅ Ticket #56789 created.`
    *   `⏳ Researching standard NDA clauses...`
    *   `✍️ Drafting your document...`
6.  **Final Result:** Once the workflow is complete, the final, detailed answer and the drafted document are displayed.

---

## 9. Getting Started (Developer Guide)

### 9.1 Local Development (using Docker Compose)

This setup uses Docker Compose to orchestrate the entire application stack locally. The `docker-compose.yml` file is configured to **pull official Docker images for all infrastructure components, including MinIO**, and build images for our custom AI agent services.

#### Prerequisites
*   Git
*   Docker & Docker Compose
*   Bun (for managing the frontend locally)
*   Python 3.10+ & Poetry
*   API Keys for external services (Cohere, GPT-5 via OpenAI, etc.)

#### Configuration & Running
Of course. Here is the complete and final version of the Project ThemisAI System Design & Architecture document. As requested, the entire content is rewritten from top to bottom to provide a clean, cohesive, and definitive blueprint for your project, incorporating all the specified details.

---

# Project ThemisAI: System Design & Architecture

**Version:** 2.7
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
    *   [I/O Agent (Conversational Gateway)](#io-agent-conversational-gateway)
    *   [Supervisor Agent (High-Level Orchestrator)](#supervisor-agent-high-level-orchestrator)
    *   [Legal Researcher Agent (Reasoning & Research)](#legal-researcher-agent-reasoning--research)
    *   [Legal Documents Drafter Agent (Structured Drafting)](#legal-documents-drafter-agent-structured-drafting)
    *   [Legal Documents Reviewer Agent (Analytical Review)](#legal-documents-reviewer-agent-analytical-review)
6.  [Data Flow & Kafka Topics](#6-data-flow--kafka-topics)
7.  [The Advanced RAG Pipeline in Detail](#7-the-advanced-rag-pipeline-in-detail)
8.  [Real-time User Experience (UX) Flow](#8-real-time-user-experience-ux-flow)
9.  [Getting Started (Developer Guide)](#9-getting-started-developer-guide)
    *   [9.1 Local Development (using Docker Compose)](#91-local-development-using-docker-compose)
    *   [9.2 Production Deployment (using Google Kubernetes Engine - GKE)](#92-production-deployment-using-google-kubernetes-engine---gke)

---

## 1. Project Overview

### Vision

Project **ThemisAI** is an initiative to build a state-of-the-art, AI-powered web application that serves as an intelligent legal assistant. Our vision is to democratize access to legal knowledge by providing a guided, conversational, and accurate platform for legal research, document drafting, and compliance review within its defined domains.

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
*   **`app/agent/graph.py`**: **Crucially, this file defines the internal reasoning graph for each agent using LangGraph.**
*   **`pyproject.toml`**: The modern standard for Python project configuration, defining dependencies, metadata, and scripts for tools like `Poetry` and `pytest`.

---

## 3. System Architecture

### High-Level Diagram

```mermaid
graph TD
    subgraph "User's Browser"
        A[Next.js Frontend]
    end

    subgraph "I/O Agent (Conversational Gateway)"
        B(API Endpoint)
        LLM_IO[Conversational LLM]
        Tool(Tool: create_legal_ticket)
    end

    subgraph "Kafka Cluster"
        K1(ticket.new)
    end

    subgraph "Backend Agentic System"
        D(Supervisor Agent)
        E(Researcher Agent)
        F(Drafter Agent)
        G(Reviewer Agent)
    end

    subgraph "Data & External Services"
        H[PostgreSQL]
        I[Qdrant Vector DB]
        J[Cohere Rerank API]
        L[Cache - Redis]
        M[MinIO Object Storage]
    end

    A <--> B
    B -- "User Input" --> LLM_IO
    LLM_IO -- "Decides Action" --> B
    
    subgraph "Decision Flow"
        direction LR
        LLM_IO -- "If Legal & In-Scope" --> Tool
        LLM_IO -- "If Non-Legal/Chat" --> B
    end

    Tool -- "Publishes Payload" --> K1
    B -- "Conversational Reply" --> A
    K1 --> D
    D -- "Delegates Task" --> E & F & G

    A -- "Uploads/Downloads File" <--> M
    G -- "Reads File" --> M
    F -- "Writes File" --> M
```

### Architectural Principles

*   **Distributed Reasoning with LangGraph:** Every backend AI agent employs its own internal LangGraph to perform complex, multi-step tasks with internal logic, state management, and self-correction loops.
*   **Conversational Intake & Tool-Based Handoff:** The system begins with a sophisticated conversational agent that uses an LLM to understand intent and only triggers the main backend workflow by calling a specific "tool" when a valid, in-scope legal request is identified.
*   **Independent Microservices:** Each AI Agent and infrastructure component (like MinIO) is a **fully independent, containerized service**.
*   **Asynchronous Communication:** The system is built around an event-driven model using Apache Kafka, ensuring resilience and scalability.

---

## 4. Technology Stack

| Category | Technology | Rationale for Selection |
| :--- | :--- | :--- |
| **Large Language Model**| **GPT-5** (or equivalent SOTA model) | Chosen for its superior reasoning, instruction-following, and tool-calling capabilities. |
| **Object Storage** | **MinIO** | A high-performance, S3-compatible object storage server, self-hosted on our Kubernetes cluster for full data control. |
| **Backend Runtime** | Python & FastAPI | The industry standard for AI/ML, offering high performance and first-class asynchronous support. |
| **Frontend Runtime**| Bun | A modern, all-in-one JavaScript runtime and toolkit chosen for its exceptional performance. |
| **Frontend Framework**| Next.js (React) | A leading framework for building fast, modern, and SEO-friendly web applications. |
| **UI Library** | Shadcn/ui | A highly customizable and accessible component library for accelerating UI development. |
| **Core Agentic Logic**| LangGraph | The core framework for implementing the internal state machines and reasoning graphs for all backend AI agents. |
| **RAG Framework** | LlamaIndex | A comprehensive framework that simplifies and optimizes the entire RAG pipeline. |
| **Vector Database**| Qdrant | A high-performance, production-ready vector database optimized for semantic search. |
| **Reranking** | Cohere Rerank API | Delivers State-of-the-Art (SOTA) accuracy for document relevance with excellent multilingual support. |
| **Database** | PostgreSQL | A battle-tested relational database for storing structured data. |
| **Messaging** | Apache Kafka | The definitive choice for a resilient, scalable, and high-throughput asynchronous communication backbone. |
| **Caching** | Redis | An extremely fast in-memory data store for reducing data access latency. |

---

## 5. Core Components (Microservices)

The introduction of MinIO refines the responsibilities of several agents, particularly regarding file handling.

### I/O Agent (Conversational Gateway)

*   **Purpose:** To act as the intelligent, conversational "front door" and manage file access credentials.
*   **Refined File Handling Logic:**
    *   **For Uploads (Review):** When a user wants to upload a document, the I/O Agent requests a **pre-signed upload URL** from MinIO and sends it to the frontend. The frontend uploads the file directly to MinIO. The I/O Agent then includes the resulting object path in the Kafka message for the `create_legal_ticket` tool.
    *   **For Downloads (Drafting):** When a document is ready, the Supervisor Agent notifies the I/O Agent. The I/O Agent generates a **pre-signed download URL** from MinIO and presents this secure, temporary link to the user.

### Supervisor Agent (High-Level Orchestrator)

*   **Purpose:** To manage the high-level lifecycle of a user request after it has been ticketed.
*   **LangGraph Implementation:** Its graph, powered by **GPT-5's** reasoning, is responsible for macro-level orchestration. It delegates complex micro-tasks to the specialized agents.

### Legal Researcher Agent (Reasoning & Research)

*   **Purpose:** To perform deep, accurate, and citation-backed legal research as a multi-step process.
*   **LangGraph Implementation:** It uses its own internal LangGraph to manage a sophisticated research process. **GPT-5** enables complex query decomposition and self-critique loops for higher accuracy.

### Legal Documents Drafter Agent (Structured Drafting)

*   **Purpose:** To generate well-structured drafts of legal documents.
*   **Refined File Handling Logic:** After its internal LangGraph completes the drafting process, the agent's final step is to **write the generated document (e.g., as a `.docx` or `.pdf`) directly to a specified bucket in MinIO**. It then reports the object path and success status back to the Supervisor.

### Legal Documents Reviewer Agent (Analytical Review)

*   **Purpose:** To analyze and provide feedback on existing legal documents.
*   **Refined File Handling Logic:** The agent receives the object path of the user's uploaded document from the Kafka message. Its first step is to **use this path to download the document directly from MinIO** before beginning its LangGraph-driven review process.

---

## 6. Data Flow & Kafka Topics

| Topic Name | Example Message Payload | Producer | Consumers |
| :--- | :--- | :--- | :--- |
| `ticket.new` | `{"ticket_id": "...", "user_query": "...", "file_path": "..."}` | I/O Agent (via Tool Call) | Supervisor Agent |
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

The user's journey is designed for clarity and transparency.

1.  **Conversational Intake:** The user starts a conversation. The UI is a standard chat interface.
2.  **Initial Interaction:** The I/O Agent engages in conversation. If the user asks "What can you do?", it explains its capabilities. If the user says "Hello", it responds politely. This phase is fully handled by the I/O Agent.
3.  **Handoff Point:** The user asks a valid legal question like, "Can you draft an NDA for a software project?"
4.  **Tool Call & Confirmation:** The I/O Agent recognizes this as an in-scope task. The UI might show a brief "Thinking..." indicator. Then, the agent responds: "Certainly. I can help with that. I'm creating a ticket to begin the drafting process. You'll see live progress below."
5.  **Workflow Updates:** The UI now switches to receiving SSE updates, showing the backend progress:
    *   `✅ Ticket #56789 created.`
    *   `⏳ Researching standard NDA clauses...`
    *   `✍️ Drafting your document...`
6.  **Final Result:** Once the workflow is complete, the final, detailed answer and the drafted document are displayed.

---

## 9. Getting Started (Developer Guide)

### 9.1 Local Development (using Docker Compose)

This setup uses Docker Compose to orchestrate the entire application stack locally. The `docker-compose.yml` file is configured to **pull official Docker images for all infrastructure components, including MinIO**, and build images for our custom AI agent services.

#### Prerequisites
*   Git
*   Docker & Docker Compose
*   Bun (for managing the frontend locally)
*   Python 3.10+ & Poetry
*   API Keys for external services (Cohere, GPT-5 via OpenAI, etc.)

#### Configuration & Running
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
    docker-compose up -d --build
    ```
2.  **Verify Services:**
    Check the status of all running containers.
    ```bash
    docker-compose ps
    ```
3.  **Access the Application:**
    Open your browser and navigate to `http://localhost:3000`.

#### Frontend Development with Bun

The frontend service is powered by Bun. To work on it locally:
1.  Navigate to the frontend directory: `cd frontend`
2.  Install dependencies using Bun's fast package manager: `bun install`
3.  Run the development server: `bun run dev`

#### Frontend Dockerfile Example (`frontend/Dockerfile`)

The Dockerfile for the frontend service must be updated to use Bun's official image and commands.

```Dockerfile
# Stage 1: Install dependencies
FROM oven/bun:1.0 as deps
WORKDIR /app

# Copy package.json and bun.lockb to leverage Docker cache
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile

# Stage 2: Build the application
FROM deps as builder
WORKDIR /app
COPY . .
RUN bun run build

# Stage 3: Production image
FROM oven/bun:1.0 as runner
WORKDIR /app

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json

EXPOSE 3000
CMD ["bun", "start"]
```

### 9.2 Production Deployment (using Google Kubernetes Engine - GKE)

This method outlines deploying **ThemisAI** and its self-hosted **MinIO** dependency to a production-grade environment on **Google Cloud Platform (GCP)** using **GKE**.

#### Prerequisites
*   A Google Cloud Platform (GCP) project with billing enabled.
*   The `gcloud` command-line tool installed and authenticated (`gcloud auth login`).
*   `kubectl` installed.
*   `helm` (v3+) installed.
*   A GKE cluster created in your project.

#### Step 1: Deploy MinIO on GKE

We will use the official MinIO Helm chart for a robust, production-ready deployment.

1.  **Add MinIO Helm Repository:**
    ```bash
    helm repo add minio https://operator.min.io/
    helm repo update
    ```
2.  **Create MinIO Secrets:**
    Create a Kubernetes secret to hold the root user and password for MinIO.
    ```bash
    kubectl create secret generic minio-credentials \
      --namespace themisai \
      --from-literal=root-user='YOUR_MINIO_ADMIN_USER' \
      --from-literal=root-password='YOUR_VERY_SECURE_MINIO_PASSWORD'
    ```
3.  **Configure `minio-values.yaml`:**
    Create a configuration file for the MinIO deployment. This example sets up a standalone instance with persistence using GKE's standard persistent disks.
    ```yaml
    # minio-values.yaml
    
    # Use the secret we just created
    existingSecret: "minio-credentials"
    
    # Persistence configuration
    persistence:
      enabled: true
      size: 50Gi # Adjust size as needed
      storageClass: "standard-rwo" # Standard GKE persistent disk
      
    # Expose MinIO via a GKE Ingress
    ingress:
      api:
        enabled: true
        hosts:
          - minio-api.your-company.com
      console:
        enabled: true
        hosts:
          - minio-console.your-company.com
    ```
4.  **Install the MinIO Helm Chart:**
    ```bash
    helm install minio minio/minio \
      --namespace themisai \
      -f minio-values.yaml
    ```

#### Step 2: Deploy ThemisAI Application

Now, deploy the ThemisAI agents, configuring them to connect to the self-hosted MinIO service.

1.  **Manage ThemisAI Secrets:**
    Store the MinIO credentials and other secrets in **Google Secret Manager** and use **Workload Identity** for secure access, as described in the previous version. The secrets should now include MinIO access details.
    ```bash
    # Add MinIO credentials to Secret Manager
    gcloud secrets create minio-access-key --replication-policy="automatic"
    echo -n "YOUR_MINIO_ADMIN_USER" | gcloud secrets versions add minio-access-key --data-file=-
    
    gcloud secrets create minio-secret-key --replication-policy="automatic"
    echo -n "YOUR_VERY_SECURE_MINIO_PASSWORD" | gcloud secrets versions add minio-secret-key --data-file=-
    ```
2.  **Configure `themisai-values.yaml` for GKE:**
    Update your application's values file to include the internal address of the MinIO service.
    ```yaml
    # themisai-gke-values.yaml
    
    # Workload Identity setup (same as before)
    serviceAccount:
      create: true
      name: "themisai-ksa"
      annotations:
        iam.gke.io/gcp-service-account: "themisai-sa@YOUR_GCP_PROJECT_ID.iam.gserviceaccount.com"

    # Ingress for the I/O Agent
    ingress:
      enabled: true
      className: "gce"
      hosts:
        - host: themis.your-company.com
          paths: ["/"]

    # Environment variables for the agents to connect to MinIO
    # These will be populated from Secret Manager by your application's startup logic
    env:
      MINIO_ENDPOINT: "minio-api.your-company.com" # Or the internal service name: "minio.themisai.svc.cluster.local"
      MINIO_USE_SSL: "true"
    ```
3.  **Install the ThemisAI Helm Chart:**
    ```bash
    helm install themisai themisai-charts/themisai \
      --namespace themisai \
      -f themisai-gke-values.yaml
    ```

### 9.2 Production Deployment (using Google Kubernetes Engine - GKE)

This method outlines deploying **ThemisAI** and its self-hosted **MinIO** dependency to a production-grade environment on **Google Cloud Platform (GCP)** using **GKE**.

#### Prerequisites
*   A Google Cloud Platform (GCP) project with billing enabled.
*   The `gcloud` command-line tool installed and authenticated (`gcloud auth login`).
*   `kubectl` installed.
*   `helm` (v3+) installed.
*   A GKE cluster created in your project.

#### Step 1: Deploy MinIO on GKE

We will use the official MinIO Helm chart for a robust, production-ready deployment.

1.  **Add MinIO Helm Repository:**
    ```bash
    helm repo add minio https://operator.min.io/
    helm repo update
    ```
2.  **Create MinIO Secrets:**
    Create a Kubernetes secret to hold the root user and password for MinIO.
    ```bash
    kubectl create secret generic minio-credentials \
      --namespace themisai \
      --from-literal=root-user='YOUR_MINIO_ADMIN_USER' \
      --from-literal=root-password='YOUR_VERY_SECURE_MINIO_PASSWORD'
    ```
3.  **Configure `minio-values.yaml`:**
    Create a configuration file for the MinIO deployment. This example sets up a standalone instance with persistence using GKE's standard persistent disks.
    ```yaml
    # minio-values.yaml
    
    # Use the secret we just created
    existingSecret: "minio-credentials"
    
    # Persistence configuration
    persistence:
      enabled: true
      size: 50Gi # Adjust size as needed
      storageClass: "standard-rwo" # Standard GKE persistent disk
      
    # Expose MinIO via a GKE Ingress
    ingress:
      api:
        enabled: true
        hosts:
          - minio-api.your-company.com
      console:
        enabled: true
        hosts:
          - minio-console.your-company.com
    ```
4.  **Install the MinIO Helm Chart:**
    ```bash
    helm install minio minio/minio \
      --namespace themisai \
      -f minio-values.yaml
    ```

#### Step 2: Deploy ThemisAI Application

Now, deploy the ThemisAI agents, configuring them to connect to the self-hosted MinIO service.

1.  **Manage ThemisAI Secrets:**
    Store the MinIO credentials and other secrets in **Google Secret Manager** and use **Workload Identity** for secure access, as described in the previous version. The secrets should now include MinIO access details.
    ```bash
    # Add MinIO credentials to Secret Manager
    gcloud secrets create minio-access-key --replication-policy="automatic"
    echo -n "YOUR_MINIO_ADMIN_USER" | gcloud secrets versions add minio-access-key --data-file=-
    
    gcloud secrets create minio-secret-key --replication-policy="automatic"
    echo -n "YOUR_VERY_SECURE_MINIO_PASSWORD" | gcloud secrets versions add minio-secret-key --data-file=-
    ```
2.  **Configure `themisai-values.yaml` for GKE:**
    Update your application's values file to include the internal address of the MinIO service.
    ```yaml
    # themisai-gke-values.yaml
    
    # Workload Identity setup (same as before)
    serviceAccount:
      create: true
      name: "themisai-ksa"
      annotations:
        iam.gke.io/gcp-service-account: "themisai-sa@YOUR_GCP_PROJECT_ID.iam.gserviceaccount.com"

    # Ingress for the I/O Agent
    ingress:
      enabled: true
      className: "gce"
      hosts:
        - host: themis.your-company.com
          paths: ["/"]

    # Environment variables for the agents to connect to MinIO
    # These will be populated from Secret Manager by your application's startup logic
    env:
      MINIO_ENDPOINT: "minio-api.your-company.com" # Or the internal service name: "minio.themisai.svc.cluster.local"
      MINIO_USE_SSL: "true"
    ```
3.  **Install the ThemisAI Helm Chart:**
    ```bash
    helm install themisai themisai-charts/themisai \
      --namespace themisai \
      -f themisai-gke-values.yaml
    ```