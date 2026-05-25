# 🖥️ Episode 9 — Building a DevOps Command Center

> A complete, hands-on monitoring and alerting stack built locally using Flask, Prometheus, Alertmanager, Grafana, and Node Exporter — all orchestrated with Docker Compose.

![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-000000?style=for-the-badge&logo=flask&logoColor=white)

---

## 📖 Project Overview

This project is **Episode 9** of a progressive DevOps Homelab series focused on real-world monitoring and alerting. In this episode, I successfully implemented and ran a full **Operations Command Center** locally — a single unified workspace where an on-call engineer can observe traffic, latency, error budgets, and live alert status without jumping between multiple tools.

The idea behind a command center is simple but powerful: in a real production environment, engineers often waste critical minutes during an incident switching between dashboards, alert tools, and logs. Episode 9 solves this by wiring together a Flask-based web application, Prometheus for metrics collection, Alertmanager for intelligent alert routing, Grafana for visualization, and Node Exporter for host-level metrics — all running inside Docker containers that communicate seamlessly over a shared network.

What makes this episode different from earlier ones in the series is the focus on **operationalization**. The code itself is similar to Episode 8, but here the emphasis is on how you package, present, and run the monitoring experience so that it is genuinely useful during real incidents. I practiced assembling a Grafana home dashboard that embedded key panels for traffic rate, latency percentiles, error budgets, and an active alerts table — all in one screen, giving me the "wallboard" experience that DevOps and SRE teams use in production environments.

By running this project locally, I gained deep hands-on understanding of how metrics flow from an application all the way through to a Slack notification, how alert rules are written and tuned, how Alertmanager routes different severity levels to different audiences, and how Grafana dashboards are structured to support fast incident triage.

---

## 🧰 Tech Stack

### Flask (Python) — The Simulated Application

The application at the heart of this project is a lightweight **Flask web app** written in Python. Flask is a micro web framework that makes it easy to build simple HTTP services. In this project, the app does not represent a real business service — instead, it is intentionally instrumented to **simulate real-world failure patterns** so that the monitoring stack has something meaningful to observe.

The app exposes three HTTP endpoints. The home route `/` is where all the interesting behavior happens: on every request, the app uses Python's `random` module to probabilistically decide what kind of response to return. Roughly 70% of the time it returns a healthy `200 OK` response. About 20% of the time it simulates a minor issue and returns an HTTP `400` response, incrementing a warning-level error counter. The remaining 10% of the time it simulates a critical failure and returns an HTTP `500` response, incrementing a critical-level error counter. The `/slow` endpoint simulates latency spikes by sleeping for a random duration between 1 and 3 seconds before responding — this is useful for testing latency-based dashboards and alerts. The `/metrics` endpoint is the most important one from Prometheus's perspective: it returns all collected metrics in the Prometheus exposition format, which Prometheus scrapes every 15 seconds.

The app uses two Prometheus metric types. A `Counter` named `app_errors_total` counts errors and is labelled by type (`warning` or `critical`), allowing Prometheus to query each type independently. A `Histogram` named `app_request_latency_seconds` records the time each request takes, which enables percentile calculations like p50, p95, and p99 latency in Grafana.

### Prometheus — The Metrics Collection and Alerting Engine

**Prometheus** is an open-source systems monitoring and alerting toolkit originally developed at SoundCloud. It operates on a **pull model**: rather than applications pushing data to a central server, Prometheus actively scrapes configured targets at regular intervals and stores the data in its own time-series database. This makes it resilient, self-contained, and easy to operate.

In this project, Prometheus is configured to scrape three targets every 15 seconds. The first is the Flask app itself at `web:5000`, which provides application-level error and latency metrics. The second is Node Exporter at `node_exporter:9100`, which provides host-level system metrics like CPU usage, memory consumption, disk I/O, and network throughput. The third is Prometheus scraping itself at `prometheus:9090`, which is useful for monitoring the health of the monitoring system itself.

Beyond collection, Prometheus also evaluates **alert rules** defined in `alert_rules.yml` on the same 15-second interval. When a rule's expression evaluates to true for long enough, Prometheus marks the alert as firing and forwards it to Alertmanager. This separation of concerns — Prometheus handles detection, Alertmanager handles notification — is a core design principle that keeps the system modular and maintainable.

### Alertmanager — The Intelligent Notification Router

**Alertmanager** is a companion tool to Prometheus that handles the lifecycle of alerts after they are detected. Its job is not to detect problems — Prometheus does that — but to decide **who should be notified, when, and how**. This is a subtle but critical distinction that prevents alert fatigue and ensures the right people get actionable notifications.

Alertmanager receives alerts from Prometheus over HTTP and applies a routing tree to decide where each alert should go. In this project, the routing logic is severity-based. Warning-level alerts are sent to a Slack channel monitored by the development team — the kind of alerts that need attention during business hours but do not require waking anyone up. Critical-level alerts are sent to a separate Slack channel monitored by the on-call engineer — these require immediate action regardless of the time of day.

Alertmanager also handles several important operational concerns automatically. It **groups** related alerts together so that if 50 similar errors fire at once, the team receives one organized message rather than 50 individual pings. It **deduplicates** alerts so that the same alert does not keep firing notifications every 15 seconds. It enforces a **repeat interval** of 30 minutes, meaning that if an issue is still ongoing, a reminder is sent every half hour. When a problem resolves, Alertmanager automatically sends a **resolved notification** to the same channel, closing the loop for the on-call engineer.

### Grafana — The Visualization and Command Center Layer

**Grafana** is an open-source analytics and interactive visualization platform. It connects to Prometheus as a data source and allows engineers to build rich, real-time dashboards using PromQL (Prometheus Query Language) queries. In this project, Grafana serves as the **front-end of the command center** — the visual interface that an engineer looks at during an incident or during their daily operations review.

The power of Grafana in a command center context is its ability to bring multiple signal types onto one screen. A well-designed Episode 9 dashboard would show request traffic rate, error rate broken down by severity, request latency at multiple percentiles (p50, p95, p99), an active alerts table pulling directly from Alertmanager, and host-level resource utilization from Node Exporter — all updating in real time. This eliminates the need to context-switch between tools during an incident.

Grafana also supports **kiosk mode**, where the dashboard fills the entire screen without the navigation UI — ideal for displaying on a large monitor or TV in an operations center. Playlists can be configured to rotate through multiple dashboards automatically, giving a team-wide view of system health at a glance.

### Node Exporter — The Host Metrics Agent

**Node Exporter** is a Prometheus exporter that exposes hardware and operating system metrics from the host system. In this project, it runs as a Docker container alongside the other services and provides metrics that the Flask app itself cannot report — things like CPU utilization per core, total and available memory, disk read/write rates, filesystem usage, and network bytes in and out. These metrics appear in Grafana dashboards and give the infrastructure context that complements the application-level metrics from Flask.

### Docker Compose — The Orchestration Layer

**Docker Compose** is the glue that holds everything together. The `docker-compose.yml` file defines all five services — the Flask app, Prometheus, Alertmanager, Grafana, and Node Exporter — and specifies how they are built, configured, networked, and started. With a single command (`docker compose up --build`), the entire stack comes to life. Each service runs in its own isolated container but communicates with others over Docker's internal networking using service names as hostnames (e.g., Prometheus reaches the app at `web:5000` and Alertmanager at `alertmanager:9093`). Grafana's dashboard data is persisted in a named Docker volume so that dashboards survive container restarts.

---

## 📁 Project Structure

```
episode9-command-center/
│
├── app/
│   ├── app.py               ← Flask web app with Prometheus metrics instrumentation
│   ├── Dockerfile           ← Container build recipe for the Flask app
│   └── requirements.txt     ← Python package dependencies (flask, prometheus-client)
│
├── prometheus/
│   ├── prometheus.yml       ← Scrape targets, alertmanager endpoint, scrape intervals
│   └── alert_rules.yml      ← Alert rule definitions (warning and critical thresholds)
│
├── alertmanager/
│   └── alertmanager.yml     ← Slack routing rules, grouping, repeat intervals
│
├── scripts/
│   └── fire_errors.sh       ← Bash script to simulate traffic and trigger alerts
│
├── docker-compose.yml       ← Orchestrates all 5 services in one command
├── INTERVIEW_QUESTIONS.md   ← SRE/DevOps interview questions for this episode
├── INTERVIEW_ANSWERS.md     ← Model answers to the interview questions
└── README.md                ← This documentation file
```

---

## 🏗️ Architecture — Big Picture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DOCKER COMPOSE NETWORK                             │
│                                                                             │
│  ┌─────────────────────┐          ┌──────────────────────────────────────┐  │
│  │    Flask App         │          │           Node Exporter              │  │
│  │    web:5000          │          │         node_exporter:9100           │  │
│  │                      │          │                                      │  │
│  │  GET /       → 200   │          │  Exposes CPU, RAM, Disk, Network     │  │
│  │  GET /       → 400   │          │  metrics of the Docker host system   │  │
│  │  GET /       → 500   │          └──────────────┬───────────────────────┘  │
│  │  GET /slow   → wait  │                         │                          │
│  │  GET /metrics ────────┼─────────────────────── │ ──────────┐             │
│  └─────────────────────┘                         │            │             │
│                                                  │            │             │
│                              scrape every 15s    │            │             │
│                                       ┌──────────▼────────────▼──────────┐  │
│                                       │          Prometheus               │  │
│                                       │        prometheus:9090            │  │
│                                       │                                   │  │
│                                       │  ① Scrapes /metrics endpoints    │  │
│                                       │  ② Stores as time-series data    │  │
│                                       │  ③ Evaluates alert_rules.yml     │  │
│                                       │  ④ Fires alerts → Alertmanager   │  │
│                                       └────────┬──────────────┬───────────┘  │
│                                                │              │              │
│                                         alerts │              │ queries      │
│                                                │              │              │
│                              ┌─────────────────▼──┐  ┌────────▼───────────┐  │
│                              │    Alertmanager     │  │      Grafana        │  │
│                              │  alertmanager:9093  │  │    grafana:3000     │  │
│                              │                     │  │                    │  │
│                              │  warning  → #dev    │  │  Command Center    │  │
│                              │  critical → #oncall │  │  Dashboards        │  │
│                              │                     │  │  Kiosk / Wallboard │  │
│                              └──────────┬──────────┘  └────────────────────┘  │
│                                         │                                    │
└─────────────────────────────────────────┼────────────────────────────────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │      Slack             │
                              │                        │
                              │  #slack-dev-warnings   │
                              │  (warning alerts)      │
                              │                        │
                              │  #devops-alerts-       │
                              │  monitoring            │
                              │  (critical alerts)     │
                              └───────────────────────┘
```

---

## 🔄 How Everything Flows — Detailed Walkthrough

### Flow 1: Metrics Collection Flow

The metrics flow begins the moment a user or script sends an HTTP request to the Flask application. When the request hits the `/` endpoint, the application processes it and decides what kind of response to return based on random probability. Regardless of the outcome — healthy, warning, or critical — the Prometheus client library inside the app silently updates the appropriate metric counters and histograms in memory.

Every 15 seconds, Prometheus wakes up and visits the `/metrics` endpoint of each configured scrape target. For the Flask app, this means reading the current values of `app_errors_total` and `app_request_latency_seconds`. For Node Exporter, it reads dozens of system-level metrics. Prometheus records each reading as a data point in its internal time-series database, associating it with a timestamp and the labels that identify the source.

Grafana, which has Prometheus configured as a data source, continuously issues PromQL queries against this database to power its dashboard panels. When an engineer looks at the Grafana command center, they are seeing live data that was collected moments ago from the running containers. The latency between a real event (like a crash in the Flask app) and its appearance on a Grafana dashboard is typically under 30 seconds, driven by Prometheus's 15-second scrape interval.

```
HTTP Request arrives at Flask App
         │
         ▼
  app.py decides response type (random probability):
         ├── 70% → HTTP 200 "App is healthy!" → latency recorded
         ├── 20% → HTTP 400 "Minor issue"     → warning counter +1
         └── 10% → HTTP 500 "Critical error!" → critical counter +1
         │
         ▼
  /metrics endpoint holds updated counters in memory
         │
         │ ◄── Prometheus scrapes every 15 seconds
         ▼
  Prometheus stores data as time-series in its TSDB
         │
         ▼
  Grafana queries Prometheus via PromQL
         │
         ▼
  Dashboard panels update with live data
```

---

### Flow 2: Alert Detection and Firing Flow

While Prometheus is collecting metrics, it simultaneously evaluates the alert rules defined in `alert_rules.yml` on the same 15-second cycle. Each rule contains a PromQL expression that Prometheus evaluates against its stored data. If the expression resolves to a positive (non-zero) result, the alert enters a **pending** state. If it stays in that state for the duration defined by the `for` field, it transitions to a **firing** state and Prometheus forwards the alert to Alertmanager.

The first rule, `AppWarningRateHigh`, uses the `increase()` function to count how many warning-type errors occurred in the last 60 seconds. If that count exceeds 3, the alert fires immediately because its `for` duration is set to `0s`. This means the very first scrape interval where the condition is met will trigger the alert — no waiting period. This makes sense for warning alerts where you want fast visibility but not necessarily emergency-level urgency.

The second rule, `AppCriticalErrorRateHigh`, counts critical errors in the last 60 seconds and fires if that count is greater than zero. However, unlike the warning rule, this one has a `for: 30s` clause, meaning the condition must remain true for at least 30 seconds before the alert fires. This prevents a single transient critical error from immediately paging an on-call engineer — the system waits to confirm the problem is sustained before escalating.

```
Prometheus evaluates alert_rules.yml every 15 seconds
         │
         ├── Check: increase(app_errors_total{type="warning"}[1m]) > 3 ?
         │         YES → Alert enters PENDING → for: 0s → immediately FIRING
         │         NO  → Alert stays inactive
         │
         └── Check: increase(app_errors_total{type="critical"}[1m]) > 0 ?
                   YES → Alert enters PENDING
                         sustained for 30s? → YES → Alert becomes FIRING
                         sustained for 30s? → NO  → stays PENDING
         │
         ▼
  FIRING alerts sent to Alertmanager at alertmanager:9093
```

---

### Flow 3: Alert Routing and Slack Notification Flow

Once Alertmanager receives a firing alert from Prometheus, it begins applying its routing logic. The first thing it does is check whether the incoming alert matches any existing active alert groups. If multiple alerts fire with the same `alertname` and `severity` labels within a short window, Alertmanager groups them together and sends a single consolidated notification rather than flooding the Slack channel with individual messages.

Alertmanager waits for `group_wait: 10s` before sending the first notification. This brief delay allows any closely related alerts to arrive and be bundled into the same message. After the initial message is sent, any new alerts belonging to the same group are batched and sent at most once every `group_interval: 1m`. If the alert is still firing and no new information has arrived, Alertmanager sends a reminder notification every `repeat_interval: 30m`.

The routing tree then evaluates which receiver should handle the alert. If the alert carries a `severity=warning` label, it matches the first sub-route and is directed to the `slack-dev` receiver, which posts to `#slack-dev-warnings` — a channel the development team monitors during working hours. If the alert carries `severity=critical`, it matches the second sub-route and is directed to the `slack-oncall` receiver, which posts to `#devops-alerts-monitoring` — the channel actively monitored by the on-call engineer. The `continue: false` setting on each sub-route ensures that once a match is found, evaluation stops, preventing the same alert from being sent to multiple channels.

When the problem is resolved — meaning Prometheus's alert rule expression returns false for a sustained period — Prometheus sends a resolve notification to Alertmanager, which then posts a "resolved" message to the same Slack channel. This closes the feedback loop for the on-call engineer and confirms that the system has returned to a healthy state. The `send_resolved: true` setting in each Slack receiver configuration enables this behavior.

```
Alertmanager receives FIRING alert from Prometheus
         │
         ▼
  Group by [alertname, severity]
  Wait 10 seconds (group_wait) for related alerts to batch
         │
         ▼
  Evaluate routing tree:
         │
         ├── severity = "warning"?
         │         └──► Receiver: slack-dev
         │               Channel: #slack-dev-warnings
         │               Audience: Development team (non-urgent)
         │
         ├── severity = "critical"?
         │         └──► Receiver: slack-oncall
         │               Channel: #devops-alerts-monitoring
         │               Audience: On-call engineer (urgent 🚨)
         │
         └── No match?
                   └──► Default receiver: slack-notifications
                         Channel: #devops-alerts-monitoring
         │
         ▼
  Slack message sent via Incoming Webhook
         │
         ▼
  If still firing after 30 minutes → send repeat reminder
  If problem resolves → send ✅ Resolved notification
```

---

## 🔌 Port Reference

Each service in the Docker Compose stack exposes a specific port on localhost, making it easy to access all tools during development and learning.

| Service | Port | Access URL | Purpose |
|---|---|---|---|
| Flask App | `5000` | http://localhost:5000 | Web app endpoints |
| Prometheus | `9090` | http://localhost:9090 | Metrics, rules, targets UI |
| Alertmanager | `9093` | http://localhost:9093 | Alert status and silences UI |
| Grafana | `3000` | http://localhost:3000 | Dashboards and visualization |
| Node Exporter | `9100` | http://localhost:9100/metrics | Raw host metrics |

---

## 🚀 Running the Project Locally

### Prerequisites

Before starting, ensure the following are installed and running on your machine:

- **Docker Desktop** — provides both Docker Engine and Docker Compose
- A **Slack workspace** with Incoming Webhooks configured (required for alert notifications to reach Slack)

### Step 1 — Configure Slack Webhooks

Open `alertmanager/alertmanager.yml` and replace the placeholder `api_url` values with your real Slack Incoming Webhook URLs. You need webhooks for both the dev channel and the on-call channel. You can create webhooks at https://api.slack.com/messaging/webhooks.

### Step 2 — Start the Full Stack

Navigate to the `episode9-command-center` directory and run:

```bash
docker compose up --build
```

This command builds the Flask app Docker image, pulls all other service images, creates a shared Docker network, and starts all five services. The `--build` flag ensures the Flask app image is rebuilt if you have made any code changes. Wait until you see all services report as healthy in the terminal output.

### Step 3 — Verify All Services Are Running

Open a browser and visit each URL to confirm every service is live:

- **Flask App** at http://localhost:5000 — should show `"App is healthy!"` (most of the time)
- **Prometheus Targets** at http://localhost:9090/targets — all three targets (`app`, `node`, `prometheus`) should show as `UP`
- **Alertmanager** at http://localhost:9093 — should display the Alertmanager UI with no active alerts initially
- **Grafana** at http://localhost:3000 — login with `admin / admin`

### Step 4 — Seed Traffic to Populate Dashboards

Run the included traffic generation script to hit the app repeatedly and generate meaningful metrics:

```bash
# Send 100 requests (default)
./scripts/fire_errors.sh

# Send 500 requests for more data
./scripts/fire_errors.sh 500
```

This script hits `http://localhost:5000` every 0.3 seconds and prints the HTTP response code for each request. As it runs, Prometheus collects the error and latency data, your Grafana dashboards populate with real data, and if enough errors accumulate, Alertmanager sends Slack notifications.

### Step 5 — Build the Grafana Command Center

Log in to Grafana, add Prometheus as a data source (`http://prometheus:9090`), and create a new dashboard with the following panels:

| Panel Title | PromQL Query |
|---|---|
| Request Rate | `rate(app_request_latency_seconds_count[1m])` |
| Warning Error Rate | `rate(app_errors_total{type="warning"}[1m])` |
| Critical Error Rate | `rate(app_errors_total{type="critical"}[1m])` |
| p95 Request Latency | `histogram_quantile(0.95, rate(app_request_latency_seconds_bucket[5m]))` |
| Host CPU Usage % | `100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)` |
| Host Memory Usage % | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` |

Enable kiosk mode by adding `?kiosk` to the dashboard URL for a fullscreen wallboard view.

---

## 🔔 Alert Rules — In Detail

### AppWarningRateHigh

This alert monitors the rate of warning-level errors in the application. It uses the `increase()` function over a 1-minute window to count how many warning errors have accumulated. If more than 3 warning errors occur in any 60-second window, the alert fires immediately. The `for: 0s` setting means there is no grace period — the very first evaluation where the threshold is breached triggers the notification. Warning alerts are tagged with `severity: warning` and routed to the development team's Slack channel.

### AppCriticalErrorRateHigh

This alert monitors critical errors with a slightly different philosophy. Any critical error — even just one — is considered serious enough to alert on. However, the `for: 30s` setting introduces a deliberate 30-second confirmation window. This means the condition must remain true across at least two consecutive 15-second evaluation cycles before the alert fires. This design choice prevents a single transient error (perhaps caused by a momentary network blip or a one-off race condition) from immediately paging the on-call engineer. If critical errors are sustained, the alert fires and is routed urgently via the `slack-oncall` receiver.

---

## 🏋️ Practice Exercises

**Exercise 1 — Morning Ops Review Dashboard:** Build a Grafana dashboard snapshot that captures the key health metrics for the past 24 hours. Share it with a teammate and document what "normal" baseline values look like for this system. This teaches you to distinguish signal from noise.

**Exercise 2 — Kiosk Wallboard:** Configure Grafana's playlist feature to rotate through three dashboards — traffic overview, latency breakdown, and active alerts table — on a 30-second interval in kiosk mode. This simulates what a real operations team would display on a shared screen.

**Exercise 3 — Dashboard Provisioning:** Use Grafana's provisioning system to define your dashboard as a JSON file and mount it automatically on container startup. This means every time you run `docker compose up`, your dashboard is pre-installed without any manual setup.

**Exercise 4 — Blackbox Probing:** Add the `blackbox_exporter` service to the Docker Compose stack and configure Prometheus to probe `http://web:5000` externally. Add an uptime panel to your Grafana dashboard. This gives you the outside-in perspective that internal metrics cannot provide.

**Exercise 5 — Alert Tuning:** Deliberately adjust the alert rule thresholds and `for:` durations. Observe how this affects the number of alerts received and the time-to-alert. Document the tradeoff between sensitivity (catching real problems fast) and specificity (avoiding false positives).

---

## 💼 SRE Interview Q&A

**Q1: What information belongs on a "single pane of glass" for SREs, and how do you prevent it from becoming cluttered?**

A command center dashboard should surface the four golden signals — latency, traffic, errors, and saturation — alongside active alert status and incident timelines. The key to preventing clutter is discipline: only display metrics that require action or provide essential context during an incident. Less-critical panels should exist in linked detail dashboards that engineers can drill into rather than on the main wallboard. A useful mental model is to ask: if someone glanced at this screen for 5 seconds during an incident, would they know whether the system is healthy or not?

**Q2: How can you measure whether dashboards or runbooks actually improve MTTR?**

Instrument your incident response process. Record the time from alert firing to first acknowledgment, from acknowledgment to diagnosis, and from diagnosis to resolution. Track these numbers over time and compare periods before and after introducing the command center. Survey on-call engineers after incidents to ask whether the dashboard helped them diagnose faster. If MTTR is improving and post-incident surveys show reduced tool-switching, the investment in dashboards is paying off.

**Q3: Discuss strategies for sharing dashboards across teams while maintaining consistent standards.**

Establish a dashboard style guide that defines standard color coding (e.g., green for healthy, yellow for warning, red for critical), consistent time range defaults, and naming conventions. Use Grafana's library panels feature to create reusable panel templates that teams pull from rather than build from scratch. Implement a review process where dashboard changes are submitted as pull requests alongside the infrastructure-as-code files. Use Grafana provisioning to ensure all environments start from the same validated baseline.

**Q4: When would you invest in synthetic monitoring alongside internal metrics?**

Synthetic monitoring — external probes that simulate user behavior — is essential when your internal metrics cannot capture the full user experience. DNS failures, CDN misconfigurations, SSL certificate expiry, and regional routing issues might not register as errors inside your data center but will cause real users to experience outages. Adding synthetic probes to your command center gives you the outside-in perspective that validates whether users can actually reach your service, independent of what your internal health checks report.

**Q5: How do you handle secrets (API keys, webhooks) when publishing dashboards or kiosks to communal screens?**

Never embed sensitive credentials in dashboard JSON, configuration files committed to public repositories, or kiosk URLs that might be screen-recorded. Store webhook URLs and API tokens in a secrets manager such as HashiCorp Vault or AWS SSM Parameter Store, and reference them as environment variables in your container configuration. Use read-only service accounts for Grafana data source connections. When sharing dashboard snapshots publicly, ensure they contain only static data and no live data source credentials.

---

## 📚 Series Overview

This project is part of a 10-episode progressive DevOps monitoring series:

| Episode | Title | Key Concepts |
|---|---|---|
| 1 | No Monitoring | Baseline without observability |
| 2 | Basic Monitoring | Prometheus setup, first scrape |
| 3 | Grafana | First dashboards, PromQL basics |
| 4 | Alerting | Alert rules, Alertmanager intro |
| 5 | App Failure Alerts | Instrumenting a real app |
| 6 | Visualization | Advanced Grafana panels |
| 7 | Production Outage | Simulating and diagnosing incidents |
| 8 | Smart Alerting | Warning vs critical, routing logic |
| **9** | **Command Center** | **Unified ops workspace (this episode)** |
| 10 | Enterprise Monitoring | Scaling, federation, advanced patterns |

---

## 🛑 Stopping the Stack

```bash
# Stop all containers (preserves Grafana dashboard data)
docker compose down

# Stop all containers AND remove stored data volumes
docker compose down -v
```

---

## 🤝 1:1 Hands-On Support

Want guided incident practice, CI/CD practice packs, or a personal 1:1 coaching session?

👉 **Book here:** https://topmate.io/learnwithdevopsengineer

---

<div align="center">

⭐ **If this project helped you learn, please give it a star!** ⭐

Made with ❤️ for DevOps engineers who learn by doing.

</div>
