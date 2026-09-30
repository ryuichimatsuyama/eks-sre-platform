# SRE Platform on Amazon EKS
Amazon EKS上に構築した、Production運用を想定したSRE / DevOpsプラットフォームです。

Google Online Boutiqueの[fork](https://github.com/ryuichimatsuyama/microservices-demo)をWorkloadとして利用し、Infrastructure Provisioning、CI/CD、GitOps、Progressive Delivery、Observability、SLO-based Alerting、Incident ResponseまでをEnd-to-Endで実装しています。

---

## Architecture

```mermaid
flowchart TB

    %% =========================================================
    %% External
    %% =========================================================

    USER["Users"]
    CF["Cloudflare<br/>WAF / Rate Limiting"]
    PD["PagerDuty<br/>Incident Management"]

    USER -->|HTTPS| CF


    %% =========================================================
    %% CI/CD & Artifact Management
    %% =========================================================

    subgraph CICD["CI/CD & Artifact Management"]
        direction TB

        APP_PR["Application PR<br/>Source Validation"]

        PR_CI["GitHub Actions<br/>
        Build / Test / Trivy<br/>
        PR Environment / E2E"]

        MAIN["GitHub<br/>main"]

        BUILD["GitHub Actions<br/>
        Build / Push"]

        GHCR["GHCR<br/>Container Images"]

        RENOVATE["Renovate<br/>
        Artifact Promotion"]

        RENOVATE_PR["Renovate PR<br/>
        Artifact Validation"]

        APP_PR -->|Trigger| PR_CI
        PR_CI -->|Required Checks| APP_PR
        APP_PR -->|Merge| MAIN

        MAIN -->|Trigger| BUILD
        BUILD -->|Push Image| GHCR
        GHCR --> RENOVATE
        RENOVATE -->|Image Tag Update| RENOVATE_PR
        RENOVATE_PR -->|Merge| MAIN
    end


    %% =========================================================
    %% Infrastructure & Platform Configuration as Code
    %% =========================================================

    subgraph IAC["Infrastructure & Platform Configuration as Code"]
        direction TB

        TF["Terraform"]

        TF_AWS["AWS<br/>VPC / EKS / IAM"]

        TF_PLATFORM["Platform Services<br/>
        GitHub / Cloudflare / PagerDuty"]

        TF --> TF_AWS
        TF --> TF_PLATFORM
    end


    %% =========================================================
    %% AWS
    %% =========================================================

    subgraph AWS["AWS — ap-northeast-1"]
        direction TB

        subgraph VPC["VPC — 10.0.0.0/16"]
            direction TB

            SUBNETS["Private Subnets<br/>
            3 Availability Zones"]

            subgraph EKS["Amazon EKS"]
                direction TB

                ARGO["Argo CD<br/>GitOps Controller"]

                APPSET["ApplicationSet<br/>
                PR Environments"]

                subgraph GITOPS["GitOps Managed Resources"]
                    direction TB


                    %% -----------------------------------------
                    %% Application
                    %% -----------------------------------------

                    subgraph APPLICATION["Application"]
                        direction TB

                        APP["Online Boutique<br/>
                        11 Microservices"]

                        PREVIEW["PR Environments"]
                    end


                    %% -----------------------------------------
                    %% Progressive Delivery & Service Mesh
                    %% -----------------------------------------

                    subgraph DELIVERY["Progressive Delivery & Service Mesh"]
                        direction TB

                        ROLLOUTS["Argo Rollouts<br/>
                        Canary 20% → 50% → 100%"]

                        ISTIO["Istio<br/>
                        STRICT mTLS<br/>
                        Canary Traffic Splitting"]
                    end


                    %% -----------------------------------------
                    %% Observability
                    %% -----------------------------------------

                    subgraph OBS["Observability & Reliability"]
                        direction TB

                        PROM["Prometheus / Sloth<br/>
                        SLI / SLO / Burn Rate"]

                        GRAFANA["Grafana<br/>
                        Metrics / Logs / Traces"]

                        LOKI["Loki / Promtail<br/>Logs"]

                        OTEL["OpenTelemetry<br/>Telemetry"]

                        JAEGER["Jaeger<br/>Tracing"]

                        ALERT["Alertmanager<br/>Alert Routing"]
                    end


                    %% -----------------------------------------
                    %% External Connectivity
                    %% -----------------------------------------

                    TUNNEL["cloudflared<br/>
                    Cloudflare Tunnel"]
                end
            end
        end
    end


    %% =========================================================
    %% Infrastructure Provisioning
    %% =========================================================

    TF_AWS -.->|Provision| SUBNETS
    TF_AWS -.->|Bootstrap| ARGO


    %% =========================================================
    %% GitOps Desired State
    %% =========================================================

    MAIN -->|Stable Desired State| ARGO

    APP_PR -->|PR Desired State| APPSET
    RENOVATE_PR -->|Release Desired State| APPSET

    APPSET -->|Generate PR Applications| PREVIEW

    ARGO -->|Reconcile| GITOPS


    %% =========================================================
    %% Progressive Delivery
    %% =========================================================

    ROLLOUTS -->|Canary| APP
    ISTIO -->|Traffic Split| APP

    PROM -->|SLO / Burn Rate| ROLLOUTS


    %% =========================================================
    %% Observability
    %% =========================================================

    APP -->|Metrics| PROM
    APP -->|Logs| LOKI
    APP -->|Telemetry| OTEL

    PROM -->|Metrics| GRAFANA
    PROM -->|Alerts| ALERT

    LOKI -->|Logs| GRAFANA

    OTEL -->|Traces| JAEGER
    JAEGER -->|Traces| GRAFANA

    ALERT -->|Page| PD


    %% =========================================================
    %% External Traffic
    %% =========================================================

    TUNNEL -->|Outbound Tunnel| CF
    CF -.->|Request| TUNNEL
    TUNNEL -->|Origin Traffic| APP
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
