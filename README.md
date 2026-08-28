# diff-review-bot

GitHub App that reviews a pull request diff and posts inline comments.

When a developer opens or updates a PR, the bot fetches the changed lines, runs several review passes in parallel (static, security, style, architecture), and writes findings back on the PR. After a merge, it stores frequent issues so later reviews on that repo can take them into account.

```mermaid
%%{init: {"flowchart": {"curve": "basis", "nodeSpacing": 18, "rankSpacing": 28}, "theme": "base", "themeVariables": {"fontFamily": "system-ui, sans-serif", "lineColor": "#8b949e", "clusterBkg": "#161b22", "clusterBorder": "#30363d", "titleColor": "#e6edf3"}}}%%
flowchart TB
    classDef gh fill:#21262d,stroke:#6e7681,color:#e6edf3
    classDef svc fill:#0d419d,stroke:#58a6ff,color:#e6edf3
    classDef store fill:#0f3d24,stroke:#3fb950,color:#e6edf3
    classDef agent fill:#3b2263,stroke:#d2a8ff,color:#e6edf3
    classDef infra fill:#3d2f00,stroke:#d4a72c,color:#e6edf3

    Dev[Developer opens a PR]:::gh --> Event[GitHub webhook]:::gh
    Event --> URL[Public webhook URL]:::infra
    URL --> ALB[Load balancer]:::infra

    subgraph runtime [Runtime]
        GW[Gateway — verify HMAC]:::svc
        WH[Webhook — parse, dedupe, store]:::svc
        Worker[Celery worker]:::svc
        OR[Orchestrator — fetch diff]:::svc
        RV[Reviewer — post comments]:::svc
        LN[Learner — store patterns]:::svc

        GW --> WH
        WH --> Worker --> OR --> RV
    end

    ALB --> GW

    subgraph review [Review passes]
        S[Static]:::agent
        C[Security]:::agent
        Y[Style]:::agent
        A[Architecture]:::agent
        M[Merge and dedupe]
        S --> M
        C --> M
        Y --> M
        A --> M
    end

    OR --> S
    OR --> C
    OR --> Y
    OR --> A
    M --> RV

    subgraph data [Data]
        PG[(Postgres — PRs, findings, patterns)]:::store
        Redis[(Redis — job queue)]:::store
    end

    WH --> PG
    WH --> Redis
    Redis --> Worker
    M --> PG
    LN --> PG

    RV --> PR[PR updated with comments]:::gh
    PR --> Again[Developer pushes again]:::gh
    Again --> Event

    PR --> Merged{PR merged?}
    Merged -->|yes| LN
    LN --> Smarter[Later reviews see frequent issues]:::agent

    subgraph ship [Ship]
        GHA[GitHub Actions]:::infra --> Docker[Docker build]:::infra --> ECR[Container registry]:::infra --> runtime
    end

    subgraph watch [Watch]
        Prom[Prometheus]:::infra --> Graf[Grafana]:::infra
        LF[LLM traces]:::infra
    end

    GW -.-> Prom
    WH -.-> Prom
    OR -.-> Prom
    RV -.-> Prom
    OR -.-> LF
```
