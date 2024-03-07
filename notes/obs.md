```mermaid
sequenceDiagram
    Prometheus->>Thanos: All Metrics
    Thanos->>ODF: All Metrics
    Thanos->>VictoriaMetrics: All Metrics
    External Grafana-->>Thanos: Query A
    Thanos-->>Thanos: Query A
    Thanos-->>Thanos: A Metrics Age<6h
    Thanos-->>ODF: Query A
    ODF-->>Thanos: A 6h<Metrics Age<90d
    Thanos-->>VictoriaMetrics: Query A
    VictoriaMetrics-->>Thanos: A 90d<Metrics Age<1y
    Thanos-->>External Grafana: Answer A
```
