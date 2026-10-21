# System Diagrams

Back to the [README](../README.md). All diagrams render natively on GitHub.

## 1. Data Flow Diagram (DFD)

Process 5.0 (reverse proxy) and data store D5 (TLS certificates) are new in v2.2.

```mermaid
flowchart TB
    SRV([Production Servers])
    ENG([Engineer])
    ADM([Administrator])
    SLK([Slack - ChatOps only])
    LE([Let's Encrypt CA])

    P1(("1.0<br/>Collect Metrics & Logs<br/>Grafana Alloy"))
    P2(("2.0<br/>Store & Evaluate Metrics<br/>Prometheus"))
    P3(("2.1<br/>Store Logs<br/>Loki"))
    P4(("3.0<br/>Visualize Data<br/>Grafana"))
    P5(("4.0<br/>Route Alerts<br/>Alertmanager"))
    P7(("5.0<br/>Terminate TLS & Proxy<br/>Nginx (host)"))
    P6(("6.0<br/>Create & Manage Incident<br/>OpsKnight (local container)"))
    P8(("7.0<br/>Deploy & Configure<br/>Ansible"))

    D1[(D1: Prometheus TSDB)]
    D2[(D2: Loki Log Store)]
    D3[(D3: OpsKnight PostgreSQL<br/>Incidents/Schedules/Users)]
    D4[(D4: Alert Rules Config)]
    D5[(D5: TLS Certificates<br/>/etc/letsencrypt)]

    SRV -- "CPU / Mem / Disk / Container / App metrics" --> P1
    SRV -- "System & app logs" --> P1

    P1 -- "remote_write metrics" --> P2
    P1 -- "log streams" --> P3

    P2 -- "write metrics" --> D1
    P3 -- "write logs" --> D2

    D1 -- "query metrics" --> P4
    D2 -- "query logs" --> P4
    P4 -- "dashboards (LAN / VPN)" --> ENG

    D4 -- "alert rules" --> P2
    P2 -- "firing/resolved alerts" --> P5
    P5 -- "HTTPS webhook (host-gateway)" --> P7
    P7 -- "proxy_pass 127.0.0.1:3001" --> P6

    LE -- "issue / renew certificate" --> D5
    D5 -- "certificate + key" --> P7

    P6 -- "create/update incident" --> D3
    P6 -- "notify via push/SMS/email" --> ENG
    P6 -- "post ChatOps war room" --> SLK
    SLK -- "interactive actions (HTTPS)" --> P7
    ENG -- "HTTPS UI: Ack / Assign / Note / Escalate / Resolve" --> P7
    P6 -- "update incident" --> D3
    D3 -- "resolved notification" --> ENG

    ADM -- "SSH / playbook run" --> P8
    P8 -- "install & configure Alloy" --> SRV
    ADM -- "define thresholds" --> D4
    ADM -- "manage inventory" --> P8
    ADM -- "configure DNS, Nginx & certificates" --> P7
    ADM -- "configure OpsKnight integration, on-call schedules & escalation policies" --> P6
```

## 2. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    SERVER ||--o{ METRIC : generates
    SERVER ||--o{ LOG_ENTRY : generates
    SERVER ||--o{ ALERT : triggers
    SERVER {
        string instance_id PK
        string hostname
        string ip_address
        string environment
        string role
        boolean alloy_installed
    }

    METRIC {
        string metric_id PK
        string server_id FK
        string metric_name
        float value
        datetime timestamp
    }

    LOG_ENTRY {
        string log_id PK
        string server_id FK
        string source
        string message
        datetime timestamp
    }

    ALERT_RULE ||--o{ ALERT : defines
    ALERT_RULE {
        string rule_id PK
        string alert_name
        string expr
        string severity
        string for_duration
        string summary_template
    }

    ALERT ||--|| OPSKNIGHT_INCIDENT : creates
    ALERT {
        string alert_id PK
        string rule_id FK
        string server_id FK
        string status
        float metric_value
        string fingerprint
        datetime started_at
        datetime resolved_at
    }

    OPSKNIGHT_INCIDENT {
        string incident_id PK, FK
        string dedup_key
        string status
        string severity
        string acknowledged_by FK
        datetime acknowledged_at
        string assigned_to FK
        string notes
        string escalation_policy_id
    }

    ENGINEER ||--o{ OPSKNIGHT_INCIDENT : acknowledges
    ENGINEER ||--o{ OPSKNIGHT_INCIDENT : assigned_to
    ENGINEER {
        string engineer_id PK
        string username
        string opsknight_user_id
        string team
    }

    ONCALL_SCHEDULE ||--o{ ENGINEER : includes
    ONCALL_SCHEDULE {
        string schedule_id PK
        string team
        string rotation_type
        string timezone
    }

    DASHBOARD {
        string dashboard_id PK
        string name
        string datasource
    }

    DASHBOARD ||--o{ SERVER : visualizes
```

## 3. Use Case Diagram

UC21 (reverse proxy, TLS and DNS) is new in v2.2.

```mermaid
flowchart LR
    Engineer(["Engineer"])
    Admin(["Administrator"])
    Prometheus(["Prometheus<br/>(system actor)"])
    OpsKnight(["OpsKnight<br/>(local system actor)"])

    subgraph SYS["Proxmox Monitoring System"]
        UC1(["View Dashboards"])
        UC2(["Explore Logs"])
        UC3(["Check Alert History"])
        UC4(["Receive Incident Notification"])
        UC5(["Acknowledge Incident"])
        UC6(["Assign Incident"])
        UC7(["Add Note to Incident"])
        UC8(["Snooze Incident"])
        UC9(["Escalate Incident"])
        UC10(["Mark Incident Resolved"])
        UC11(["Receive Resolved Notification"])
        UC12(["Configure Alert Rules"])
        UC13(["Configure Alertmanager Routing"])
        UC14(["Deploy Grafana Alloy via Ansible"])
        UC15(["Add Server to Inventory"])
        UC16(["Configure OpsKnight Integration,<br/>On-Call Schedule & Escalation Policy"])
        UC17(["Create Custom Dashboard"])
        UC18(["Evaluate Alert Rule"])
        UC19(["Fire / Resolve Alert"])
        UC20(["Back Up OpsKnight Postgres Volume"])
        UC21(["Configure Reverse Proxy,<br/>TLS Certificate & DNS"])
    end

    Engineer --> UC1
    Engineer --> UC2
    Engineer --> UC3
    Engineer --> UC4
    Engineer --> UC5
    Engineer --> UC6
    Engineer --> UC7
    Engineer --> UC8
    Engineer --> UC9
    Engineer --> UC10
    Engineer --> UC11
    Engineer --> UC17

    Admin --> UC12
    Admin --> UC13
    Admin --> UC14
    Admin --> UC15
    Admin --> UC16
    Admin --> UC1
    Admin --> UC17
    Admin --> UC20
    Admin --> UC21

    Prometheus --> UC18
    Prometheus --> UC19

    UC19 -.include.-> UC4
    UC10 -.include.-> UC11
    OpsKnight --> UC4
    OpsKnight --> UC11
    UC5 -.extend.-> UC7
    UC6 -.extend.-> UC7
```
