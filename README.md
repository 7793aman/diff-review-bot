# diff-review-bot

GitHub App that runs **LLM review agents** on a pull request diff and posts inline comments.

Four agents look at the same patch in parallel (static, security, style, architecture), share one LangGraph state, then a reviewer writes comments on the PR. After merge, frequent issues are stored so later reviews on that repo are not starting from zero.

## About

This project is a working example of **multi-agent LLM orchestration** wired into a real developer workflow — not a chatbot in a notebook.

- **Agents** — specialized reviewers, each with its own prompt and job
- **LangGraph** — shared state, parallel fan-out, merge of findings
- **LLM** — model calls traced (prompts, tokens, cost)
- **Product loop** — GitHub webhook in, comments out, patterns remembered on merge

The rest of the stack (queue, GitHub App, k8s) exists so those agents can run on a live PR.

## Features

- GitHub App webhook on `opened` / `synchronize` / merge
- HMAC verification at the edge
- Background jobs so GitHub is not blocked on the model
- Four review agents in parallel, then deduped findings
- Inline comments plus a summary review
- Repo-level patterns fed back into later style reviews

## Architecture

```mermaid
%%{init: {"flowchart": {"curve": "basis"}, "theme": "base", "themeVariables": {"fontFamily": "system-ui, sans-serif", "primaryTextColor": "#e6edf3", "lineColor": "#6e7681", "clusterBkg": "#0d1117", "clusterBorder": "#30363d", "titleColor": "#c9d1d9", "mainBkg": "#161b22"}}}%%
flowchart TB
    classDef gh fill:#161b22,stroke:#8b949e,color:#f0f6fc
    classDef svc fill:#1c3d5a,stroke:#4493f8,color:#f0f6fc
    classDef store fill:#163227,stroke:#56d364,color:#f0f6fc
    classDef agent fill:#2d1b4e,stroke:#a371f7,color:#f0f6fc
    classDef infra fill:#3d2914,stroke:#e3b341,color:#f0f6fc
    classDef step fill:#2d333b,stroke:#484f58,color:#c9d1d9

    A[👨‍💻 Developer Creates Pull Request on GitHub]:::gh --> B[🐙 GitHub PR Event]:::gh
    B --> C[🌐 Public Webhook URL]:::infra
    C --> D[⚖️ AWS Application Load Balancer]:::infra

    D --> E[🚪 Gateway Service : FastAPI]:::svc
    E --> E1[Verify HMAC Signature]:::step
    E --> E2[Reject Fake Requests]:::step
    E --> F[📩 Forward Verified Event]:::step

    F --> G[🪝 Webhook Service : FastAPI]:::svc
    G --> G1[Parse PR Number / Repo / SHA]:::step
    G --> G2[Deduplicate Existing SHA]:::step
    G --> G3[Store PR Metadata]:::step

    G3 --> DB[(🗄️ PostgreSQL RDS)]:::store

    G --> H[📬 Push Job to Redis Queue]:::step
    H --> R[(⚡ Redis / ElastiCache)]:::store

    R --> I[⚙️ Celery Worker]:::svc

    I --> J[🧠 Orchestrator Service]:::svc
    J --> J1[Fetch Code Diff from GitHub API]:::step
    J --> J2[Load Past Repo Patterns]:::step
    J --> J3[Run LangGraph Multi-Agent Review]:::step

    J3 --> K1[🔍 Static Analysis Agent]:::agent
    J3 --> K2[🛡️ Security Agent]:::agent
    J3 --> K3[🎨 Style Agent]:::agent
    J3 --> K4[🏗️ Architecture Agent]:::agent

    K1 --> L[Merge Findings]:::step
    K2 --> L
    K3 --> L
    K4 --> L

    L --> M[Remove Duplicate Comments]:::step
    M --> N[Save Findings to DB]:::step

    N --> DB

    M --> O[💬 Reviewer Service]:::svc
    O --> O1[Generate GitHub App JWT]:::step
    O --> O2[Create Installation Token]:::step
    O --> O3[Post Inline Comments]:::step
    O --> O4[Post Summary Review]:::step

    O --> P[🐙 GitHub Pull Request Updated]:::gh

    P --> Q[Developer Fixes Code & Pushes Again]:::gh
    Q --> B

    P --> Z{PR Merged?}:::step
    Z -->|Yes| AA[📘 Learner Worker]:::svc
    AA --> AB[📈 Learner Service]:::svc
    AB --> AC[Extract Frequent Issues]:::step
    AC --> AD[Store Repo Patterns]:::step
    AD --> DB
    AD --> AE[Future Reviews Become Smarter]:::agent

    subgraph Monitoring & Observability
        MO1[📊 Prometheus]:::infra
        MO2[📉 Grafana]:::infra
        MO3[🧾 LangFuse]:::infra
    end

    E --> MO1
    G --> MO1
    J --> MO1
    O --> MO1
    J --> MO3
    MO1 --> MO2

    subgraph DevOps CI/CD
        CI1[GitHub Actions]:::infra
        CI2[Docker Build]:::infra
        CI3[AWS ECR]:::infra
        CI4[EKS Kubernetes Cluster]:::infra
    end

    CI1 --> CI2 --> CI3 --> CI4
    CI4 --> E
    CI4 --> G
    CI4 --> J
    CI4 --> O
    CI4 --> AB
    CI4 --> I
```

## Agents

All four run on the same diff. Findings land in shared graph state and are merged before anything is posted.

| Agent | Looks for |
| --- | --- |
| Static | Complexity, unused names, noisy diffs |
| Security | Secrets, injection, dangerous APIs |
| Style | Readability; also sees past issues for this repo |
| Architecture | Error handling, boundaries, odd dependencies |

Orchestrator owns the graph. Reviewer is not an agent — it only talks to GitHub.

## Stack

| Layer | What |
| --- | --- |
| Agents / LLM | LangGraph, OpenAI, LangFuse |
| API | Python, FastAPI |
| Queue | Celery, Redis |
| Data | PostgreSQL |
| GitHub | GitHub App, webhooks |
| Ship | Docker, GitHub Actions, AWS ECR, EKS |
| Infra | Terraform |
| Watch | Prometheus, Grafana |

## How a review runs

1. GitHub sends a webhook to the public URL.
2. **Gateway** checks the HMAC signature and forwards the payload.
3. **Webhook** stores the PR and pushes a job onto Redis.
4. A **worker** calls **Orchestrator**, which pulls the diff and runs the four agents.
5. **Reviewer** posts inline comments and a summary with the GitHub App token.
6. If the PR is merged, **Learner** stores frequent issue patterns for that repo.

## Status

Early. Architecture and stack above are the target. Install and run steps will land here as the services exist.
