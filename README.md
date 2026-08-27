# diff-review-bot

GitHub App that reviews a pull request diff and posts inline comments.

When a developer opens or updates a PR, the bot fetches the changed lines, runs several review passes in parallel (static, security, style, architecture), and writes findings back on the PR. After a merge, it stores frequent issues so later reviews on that repo can take them into account.

```mermaid
flowchart TB
  subgraph GitHub
    Dev[Developer opens or updates a PR]
    Event[PR webhook]
    PR[PR with inline comments]
  end

  Dev --> Event
  Event --> URL[Public webhook URL]
  URL --> ALB[Load balancer]

  subgraph Cluster["Runtime"]
    GW[Gateway — verify the request is from GitHub]
    WH[Webhook — save the PR, enqueue a job]
    Q[Queue]
    Worker[Worker]
    Orch[Orchestrator — fetch diff, run review passes]
    Rev[Reviewer — post comments]
    Learn[Learner — on merge, store frequent issues]
    GW --> WH --> Q --> Worker --> Orch --> Rev
    WH --> Learn
  end

  ALB --> GW
  Rev --> PR
  PR --> Dev

  subgraph Data
    PG[(Postgres — PRs, findings, patterns)]
    Redis[(Redis — job queue)]
  end

  WH --> PG
  Orch --> PG
  Learn --> PG
  Q --> Redis

  subgraph Ship["Ship"]
    GHA[GitHub Actions]
    Img[Container images]
    GHA --> Img --> Cluster
  end

  subgraph Watch["Watch"]
    Prom[Prometheus / Grafana]
    LF[LLM traces]
  end

  GW -.-> Prom
  WH -.-> Prom
  Orch -.-> Prom
  Orch -.-> LF
  Rev -.-> Prom
```
