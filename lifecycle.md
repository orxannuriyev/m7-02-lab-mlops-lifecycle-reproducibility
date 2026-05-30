```mermaid
flowchart TD
    %% Nodes
    Data(Data Prep & Feature Engineering)
    Exp(Experimentation & Prototyping)
    Train(Automated Training Pipeline)
    Eval(Evaluation & Validation)

    subgraph Model Registry
        Staging([Staging])
        Production([Production])
        Archived([Archived])
    end

    Deploy(Online Inference API - 200 RPS)
    Monitor(Monitoring & Observability)

    %% Flow & Artifacts
    Data -- "Dataset Hash & Date Range" --> Exp
    Exp -- "Git SHA & Hyperparameters" --> Train
    Train -- "Run ID + Model URI (S3 Path)" --> Eval
    Eval -- "[Auto] Validation Metrics & Test Scores" --> Staging

    Staging -- "[Manual] Tech Lead Sign-off" --> Production
    Production -- "[Auto] Deprecation Schedule" --> Archived

    Production -- "[Auto] Deployed Version Tag" --> Deploy
    Deploy -- "Prediction Logs & Latency Stats" --> Monitor

    Monitor -- "[Auto] Drift Signal (e.g., MAE > threshold)" --> Train
    Monitor -- "Ground Truth Feedback (Actual Arrivals)" --> Data

    %% Styling
    classDef manual fill:#fdf1ec,stroke:#d9705a,stroke-width:2px;
    classDef auto fill:#eef7ec,stroke:#7eb075,stroke-width:2px;
    classDef registry fill:#eef2ff,stroke:#6366f1,stroke-width:2px;

    class Staging,Production,Archived registry;