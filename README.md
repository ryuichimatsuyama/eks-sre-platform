# SRE Platform on Amazon EKS
Amazon EKS上に構築した、Production運用を想定したSRE / DevOpsプラットフォームです。

Google Online Boutiqueの[fork](https://github.com/ryuichimatsuyama/microservices-demo)をWorkloadとして利用し、Infrastructure Provisioning、CI/CD、GitOps、Progressive Delivery、Observability、SLO-based Alerting、Incident ResponseまでをEnd-to-Endで実装しています。

---

## Architecture

### CI/CD & GitOps
```mermaid
flowchart TB

    APP_PR["Application PR<br/>Source Validation"]
    CI["GitHub Actions<br/>Build / Test / Trivy / E2E"]
    MAIN["GitHub<br/>main"]
    BUILD["GitHub Actions<br/>Build / Push"]
    GHCR["GHCR<br/>Container Images"]
    RENOVATE["Renovate<br/>Artifact Promotion"]
    RELEASE_PR["Renovate PR<br/>Artifact Validation"]

    APPSET["ApplicationSet<br/>PR Environments"]
    ARGO["Argo CD<br/>GitOps Controller"]

    PREVIEW["PR Environment"]
    STABLE["Stable Environment"]

    APP_PR -->|Trigger| CI
    CI -->|Required Checks| APP_PR
    APP_PR -->|Merge| MAIN

    MAIN --> BUILD
    BUILD -->|Push Image| GHCR
    GHCR --> RENOVATE
    RENOVATE -->|Image Tag Update| RELEASE_PR
    RELEASE_PR -->|Merge| MAIN

    APP_PR -->|PR Desired State| APPSET
    RELEASE_PR -->|Release Desired State| APPSET
    APPSET -->|Generate| PREVIEW

    MAIN -->|Stable Desired State| ARGO
    ARGO -->|Reconcile| STABLE
```

### Infrastructure & Runtime
```mermaid
flowchart TB

    USER["Users"]
    CF["Cloudflare<br/>WAF / Rate Limiting"]

    subgraph IAC["Infrastructure as Code"]
        TF["Terraform"]
        AWS_IAC["AWS<br/>VPC / EKS / IAM"]
        PLATFORM["Platform Services<br/>GitHub / Cloudflare / PagerDuty"]

        TF --> AWS_IAC
        TF --> PLATFORM
    end

    subgraph AWS["AWS — ap-northeast-1"]
        subgraph VPC["VPC — 10.0.0.0/16"]
            SUBNETS["Private Subnets<br/>3 Availability Zones"]

            subgraph EKS["Amazon EKS"]
                ARGO["Argo CD"]
                TUNNEL["cloudflared"]
                ISTIO["Istio<br/>STRICT mTLS"]
                ROLLOUTS["Argo Rollouts"]
                APP["Online Boutique<br/>11 Microservices"]

                ARGO --> APP
                ROLLOUTS -->|Canary| APP
                ISTIO -->|Traffic Split| APP
            end
        end
    end

    TF --> AWS_IAC
    AWS_IAC -.->|Provision| SUBNETS
    AWS_IAC -.->|Bootstrap| ARGO

    USER -->|HTTPS| CF
    TUNNEL -->|Outbound Tunnel| CF
    CF -.->|Request| TUNNEL
    TUNNEL -->|Origin Traffic| APP
```

### Observability & Reliability
```mermaid
flowchart TB

    APP["Online Boutique<br/>11 Microservices"]

    PROM["Prometheus / Sloth<br/>SLI / SLO / Burn Rate"]
    LOKI["Loki / Promtail<br/>Logs"]
    OTEL["OpenTelemetry<br/>Telemetry"]
    JAEGER["Jaeger<br/>Distributed Tracing"]
    GRAFANA["Grafana<br/>Metrics / Logs / Traces"]

    ALERT["Alertmanager<br/>Alert Routing"]
    PD["PagerDuty<br/>Incident Management"]

    ROLLOUTS["Argo Rollouts<br/>Canary Analysis"]

    APP -->|Metrics| PROM
    APP -->|Logs| LOKI
    APP -->|Telemetry| OTEL

    PROM -->|Metrics| GRAFANA
    LOKI -->|Logs| GRAFANA

    OTEL -->|Traces| JAEGER
    JAEGER -->|Traces| GRAFANA

    PROM -->|SLO / Burn Rate| ROLLOUTS

    PROM -->|SLO Alerts| ALERT
    ALERT -->|Page| PD
```

---

## Components

| Component | Role | Implementation |
|---|---|---|
| **Amazon EKS** | Kubernetes Platform | Online BoutiqueとSRE Platform Componentsを実行。VPC内の3 Availability Zonesに配置したPrivate Subnetを利用 |
| **Terraform** | Infrastructure & Configuration as Code | AWS / GitHub / Cloudflare / PagerDutyをCodeとして管理し、Argo CDを初期Bootstrap |
| **GitHub** | Source of Truth | Application Code / Kubernetes Desired State / Pull Requestを管理 |
| **GitHub Actions** | CI / Change Validation | Build・Test・Trivy Scan・Manifest Validationを実行し、PR Environment上でHealth Check / E2Eを実施 |
| **Docker Buildx Bake** | Container Build | Online Boutiqueの11 Microservicesを並列Build |
| **GHCR** | Container Registry | PR ImageおよびRelease Imageを保存 |
| **Renovate** | Artifact Promotion | 新しいRelease Imageを検出し、各Microserviceを同一Release Tagへ更新するPromotion PRを作成 |
| **Argo CD** | GitOps | GitをSource of TruthとしてKubernetes Desired Stateを継続的にReconcile。Configuration Driftを検出・修正 |
| **ApplicationSet** | PR Environments | Application PR / Renovate PRごとにPR環境を生成し、各PRのrevisionをデプロイ |
| **Argo Rollouts** | Progressive Delivery | Canaryを`20% → 50% → 100%`で段階的にRelease。アラート発生時はRolloutをAbort。|
| **Istio** | Service Mesh | カナリアリリースのトラフィック分割 / メトリクス収集 |
| **Prometheus** | Metrics | Metricsを収集し、SLI / SLO / Error Budget / Burn Rateを評価 |
| **Sloth** | SLO | Availability SLO `99.9%`を定義し、Alert Rulesを生成。|
| **Grafana** | Visualization | Metrics・SLI / SLO・Error Budget・Burn RateをDashboardで可視化。|
| **Loki / Promtail** | Logging | Logsを収集。|
| **OpenTelemetry** | Telemetry | Telemetryを収集。 |
| **Jaeger** | Distributed Tracing | Microservices間のRequest Flowを追跡し、Latency / Errorの発生箇所を調査。|
| **Alertmanager** | Alert Routing | Slothが生成したSLOアラートのうち、`sloth_severity="page"` をPagerDutyへルーティング |
| **PagerDuty** | Incident Management | `sloth_severity="page"` のSLOアラートをインシデント化し、オンコール担当者へ通知|
| **Trivy** | Security | CIでContainer Imageの脆弱性Scanを実行 |
| **Cloudflare Tunnel** | External Access | EKS内の`cloudflared`からCloudflareへOutbound Tunnelを確立し、Applicationを公開 |
| **Cloudflare WAF / Rate Limiting** | Edge Security | External RequestをEdgeで検査し、不正Requestや過剰Requestを制御 |
