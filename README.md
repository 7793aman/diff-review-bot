# diff-review-bot

GitHub App that reviews a pull request diff and posts inline comments.

When a developer opens or updates a PR, the bot fetches the changed lines, runs several review passes in parallel, and writes findings back on the PR. After a merge, it stores frequent issues so later reviews on that repo can take them into account.

**On a pull request**

```mermaid
%%{init: {"flowchart": {"curve": "basis", "nodeSpacing": 28, "rankSpacing": 48}, "theme": "base", "themeVariables": {"fontFamily": "system-ui, sans-serif", "lineColor": "#8b949e"}}}%%
flowchart LR
    classDef gh fill:#21262d,stroke:#6e7681,color:#e6edf3
    classDef svc fill:#0d419d,stroke:#58a6ff,color:#e6edf3
    classDef store fill:#0f3d24,stroke:#3fb950,color:#e6edf3

    PR((Pull request)):::gh
    GW[Gateway]:::svc
    WH[Webhook]:::svc
    Q[(Redis)]:::store
    OR[Orchestrator]:::svc
    RV[Reviewer]:::svc
    OUT((Comments)):::gh

    PR -->|webhook| GW --> WH --> Q --> OR --> RV -->|inline + summary| OUT
```

**Inside the review**

```mermaid
%%{init: {"flowchart": {"curve": "basis", "nodeSpacing": 20, "rankSpacing": 36}, "theme": "base", "themeVariables": {"fontFamily": "system-ui, sans-serif", "lineColor": "#8b949e"}}}%%
flowchart TB
    classDef svc fill:#0d419d,stroke:#58a6ff,color:#e6edf3
    classDef agent fill:#3b2263,stroke:#d2a8ff,color:#e6edf3
    classDef store fill:#0f3d24,stroke:#3fb950,color:#e6edf3

    OR[Orchestrator]:::svc

    OR --> S[Static]:::agent
    OR --> C[Security]:::agent
    OR --> Y[Style]:::agent
    OR --> A[Architecture]:::agent

    S --> M[Merge findings]
    C --> M
    Y --> M
    A --> M
    M --> RV[Reviewer]:::svc

    WH[Webhook]:::svc --> PG[(Postgres)]:::store
    M --> PG
    WH -->|on merge| LN[Learner]:::svc --> PG
```
