# Grafana Cloud Hands-On Learning Plan

## How to use this plan

This is a 5-week, hands-on Grafana Cloud track (now folded in with two Udemy courses) built around one running project rather than isolated tutorials. Each week adds a new signal (metrics → logs → traces/alerting → load testing) to the same test environment, so by the end you have one cohesive observability setup you built by hand — plus a portfolio piece.

**Time commitment:** \~6-9 hours/week in course-heavy weeks, \~5-7 hours/week otherwise.

**What you need:**

- A computer with Docker installed
- A free Grafana Cloud account (grafana.com/auth/sign-up/create-user)
- Something to monitor — a spare machine, a Raspberry Pi, a home server, or just your own laptop/a cloud VM running a few Docker containers

Each week below has setup steps, core practice steps, and a milestone to confirm you've actually learned it (not just followed along).

## Course Integration & Study Plan

I pulled the actual syllabi for both courses before deciding how (or whether) to fold them in — here's the honest breakdown.

### Course 1: Observability with Grafana, Prometheus, Loki, Alloy and Tempo — take this one

[Course link](https://www.udemy.com/course/grafana-prometheus-loki-alloy-tempo/) — 7h38m, 113 lectures across 16 sections, 4.5★ (6,256 ratings), best-selling Grafana course on Udemy for 7 straight years, last updated Jan 2026.

This is a genuine match: Prometheus, Grafana dashboarding, Loki, Alloy (OTel collection), Tempo, alerting — plus **Grafana Mimir** (large-scale, multi-tenant metrics storage), which the original 4-week plan didn't cover but is directly relevant to the \~500GB/day, multi-client federal environment you manage. It's built around one running example end to end, mirroring the single-project approach of this plan.

**Verdict: take the whole course**, woven into the weeks below — watch each week's relevant section right before that week's hands-on steps, so the video gives you the concept and the hands-on work makes it stick.

### Course 2: Grafana Full Course 2026 — mostly skip

[Course link](https://www.udemy.com/course/grafana-full-course-2026) — 2h41m, 17 lectures in 1 section, only 5 students enrolled (too new to have a track record), no visible rating yet.

Most of it — dashboards, PromQL basics, alerting, Loki, Tempo, Grafana Cloud setup — directly overlaps Course 1, just covered far more thinly (Course 1 spends 7h38m on ground this covers in under 3). Taking both in full means re-learning the same material twice.

It does have two things Course 1 doesn't:

- **Kubernetes-specific monitoring** (infrastructure, logging, tracing, application monitoring — all on Kubernetes), genuinely new since the hands-on plan and Course 1 both target a plain host/VM.
- A **"Grafana Interview Questions for 2026"** lecture (26 min), directly useful given your active job search.

**Verdict: skip the overlapping \~2 hours, watch just these two lectures** — folded into Week 5 below.

### Why 5 weeks instead of 4

\~7.5 hours of Course 1 video plus \~40 minutes of selected Course 2 video, added to the original \~20-28 hours of hands-on work, comes to roughly 30-36 hours total. Compressing that into 4 weeks means 7.5-9 hrs/week — risking this becoming video-watching instead of hands-on practice. Five weeks keeps each week sustainable and adds a dedicated week for Mimir + Kubernetes + interview prep, content that rounds out the track rather than padding it.

### Where each course section lands

| Week | Hands-on focus | Course 1 sections to watch first | Video time |
| --- | --- | --- | --- |
| 1 | Foundations + Metrics | Foundations of Observability; Installing Prometheus & Collecting Metrics; Installing & Configuring Grafana; Using Grafana | \~3h 5m |
| 2 | Logs (Loki) | Grafana Loki | \~38m |
| 3 | Traces + Alerting | Grafana Alloy for OTel; Grafana Tempo; Alerts, Notifications and Annotations | \~1h 28m |
| 4 | Load Testing + Polish | No direct overlap — k6 isn't covered in either course | — |
| 5 | Scale, Kubernetes & Interview Prep | Grafana Mimir: Observability at Scale (Course 1); Kubernetes Monitoring + Interview Questions (Course 2, optional) | \~1h 9m + \~40m |

Course 1 has 16 sections total; the mapping above covers 10 of them (\~6h20m). The remaining \~1h20m (further Mimir configuration, administration/security topics) isn't essential to this track's goals — browse it once enrolled and pull in anything relevant to your FedRAMP environment specifically.

### Where the courses fall short — and why the hands-on steps still matter

Watching a course lecture on a topic doesn't make the plan's hands-on step for that topic skippable. Two different things are happening: the course teaches the concept on its own mock dataset ("ShoeHub"), while the plan's steps build the same skill on your own real host — that's where it actually sticks.

A few specific gaps neither course fills at all, so these stay fully hands-on regardless of video watched:

- **LogQL depth** (Week 2) — Course 1's Loki section covers install + basic label extraction, not a dedicated LogQL filtering/parsing lecture.
- **Trace-to-logs/metrics correlation via exemplars** (Week 3) — not covered by either course; Tempo's service graphs are a related but different concept.
- **k6 load testing** (Week 4) — zero coverage in either course.
- **The correlation milestones themselves** (metrics+logs on one dashboard, a trace linked to its logs) — inherently hands-on exercises; a course demo of this doesn't substitute for doing it yourself.

## Week 1 — Foundations + Metrics

**Goal:** get a real host reporting metrics into Grafana Cloud, and build a dashboard for it from scratch so you understand every piece rather than importing someone else's.

### Step 1: Create your Grafana Cloud account

1. Go to grafana.com and sign up for the free tier.
2. Once in, note your **Grafana Cloud stack URL** (looks like `https://<yourname>.grafana.net`) and go to **My Account → Access Policies** to create an access policy token with `metrics:write` and `logs:write` scopes (you'll need `logs:write` in Week 2, so add it now to save a step). Save this token somewhere safe — you won't see it again.

*Why:* the access policy token is how Alloy authenticates when it pushes data to your Cloud stack. Treat it like a password.

### Step 2: Pick your target host

Choose one of:

- A spare Linux machine or Raspberry Pi on your network
- A small cloud VM (AWS EC2 free tier, DigitalOcean droplet)
- Your own laptop, if you just want to get started immediately

*Why it matters:* using a real host (not a toy container) means the metrics you see are meaningful — CPU spikes when you actually do something, disk fills up for a real reason. That's what makes correlation later in Week 2 satisfying instead of contrived.

### Step 3: Install Grafana Alloy

1. On your target host, install Alloy following Grafana's install docs for your OS (apt/yum package or Docker container — Docker is simplest if you're testing on your laptop).
2. Docker quick-start:

```bash
docker run -d --name alloy \
  -v $(pwd)/config.alloy:/etc/alloy/config.alloy \
  -p 12345:12345 \
  grafana/alloy:latest \
  run --server.http.listen-addr=0.0.0.0:12345 /etc/alloy/config.alloy
```

3. Verify Alloy is running: visit `http://localhost:12345` — you should see the Alloy UI.

*Why Alloy specifically:* Alloy is Grafana's current unified telemetry collector (it replaced the older Grafana Agent and Promtail). Learning it now means you're not learning a deprecated tool.

### Step 4: Configure Alloy to scrape and remote\_write metrics

1. In your `config.alloy` file, add a `prometheus.exporter.unix` component to collect host metrics (CPU, memory, disk) via node\_exporter's built-in collector logic.
2. Add a `prometheus.scrape` component pointing at that exporter.
3. Add a `prometheus.remote_write` component pointing at your Grafana Cloud Prometheus endpoint, using the access policy token from Step 1 as a Basic Auth password (username is your Cloud instance ID, found in the same Access Policies page).
4. Restart Alloy and check the Alloy UI's component graph — data should be flowing with no red/error states.

*Why this order:* scrape → remote\_write is the core Prometheus data-flow pattern you'll reuse for every future metric source, not just this host.

### Step 5: Confirm data is arriving

1. In Grafana Cloud, go to **Explore**, select your Prometheus data source.
2. Run a query like `up` — you should see your host as a target with value `1`.
3. Try `node_cpu_seconds_total` to confirm real host metrics are flowing.

### Step 6: Build your first dashboard from scratch

1. Create a **New Dashboard → Add new panel**.
2. Build three panels manually (do not import a community dashboard yet):
   - **CPU usage** — use a PromQL rate() query over `node_cpu_seconds_total`, filtered to mode `idle`, inverted to show usage %.
   - **Memory usage** — `node_memory_MemAvailable_bytes` vs `node_memory_MemTotal_bytes`.
   - **Disk usage** — `node_filesystem_avail_bytes` vs `node_filesystem_size_bytes`.
3. Add a **dashboard variable** (`$instance`) so the dashboard could support multiple hosts later, even though you only have one now.
4. Set panel types deliberately: try a **Time series** panel for CPU, a **Gauge** for current memory %, and a **Stat** panel for disk free space — so you get exposure to more than one panel type.

*Why build from scratch:* importing a dashboard teaches you to read someone else's PromQL. Building one teaches you to write your own — which is the actual skill.

### ✅ Milestone

One working dashboard, built by hand, showing live CPU/memory/disk for a real host, using a dashboard variable and at least three different panel types.

## Week 2 — Logs (Loki)

**Goal:** ship logs from the same host into Grafana Cloud Loki, and correlate a log spike with the metric spike that caused it — the moment observability tools start feeling powerful instead of academic.

### Step 1: Add a Loki write endpoint to Alloy

1. In Grafana Cloud, find your Loki endpoint URL and generate (or reuse) an access policy token with `logs:write` scope.
2. In your `config.alloy` file, add a `loki.write` component pointing at that endpoint, authenticated with Basic Auth (instance ID + token), same pattern as the Prometheus remote\_write from Week 1.

*Why reuse the pattern:* Alloy's config language (River) uses the same source → process → destination shape for every telemetry type. Recognizing that pattern is more valuable than memorizing Loki-specific syntax.

### Step 2: Configure log collection

1. Add a `loki.source.file` component pointing at a real log file on your host — `/var/log/syslog` (Linux) or a Docker container's logs via `loki.source.docker` if you're monitoring a containerized app.
2. Pipe that source into your `loki.write` component.
3. Restart Alloy and confirm no errors in the Alloy UI component graph.

### Step 3: Confirm logs are arriving

1. In Grafana Cloud, go to **Explore**, switch the data source to Loki.
2. Run `{job="varlogs"}` (or whatever label your source produces) — you should see live log lines streaming in.

### Step 4: Practice LogQL

Work through these query patterns directly in Explore, on your own real logs:

1. **Filter by keyword:** `{job="varlogs"} |= "error"`
2. **Filter by regex:** `{job="varlogs"} |~ "(?i)fail(ed|ure)"`
3. **Parse structured fields:** if your logs are JSON, try `{job="varlogs"} | json | duration > 500ms` to extract and filter on a field.
4. **Count log volume over time:** `sum(count_over_time({job="varlogs"}[5m]))` — turn this into a panel; it's the log-equivalent of a metrics time series.

*Why this matters:* LogQL's `|=`, `|~`, and `| json` pipeline syntax is the core mental model — once you're fluent in chaining filters, everything else in Loki is a variation on this.

### Step 5: Trigger a real correlation

1. On your host, deliberately cause a resource spike — run a CPU stress test (`stress --cpu 4 --timeout 60s` on Linux) or fill disk space temporarily.
2. At the same time, generate some log noise — restart a service, or trigger an application error if you have one running.
3. Go back to your Week 1 dashboard and add a **Logs panel** below your metrics panels, scoped to the same time range.
4. Zoom into the incident window — you should see the CPU/memory spike in your metric panels lined up in time with the relevant log lines below.

*Why this is the payoff:* this is the exact workflow of real incident response — see the anomaly in metrics, then use logs from the same moment to explain why. Practicing it deliberately now builds the reflex.

### ✅ Milestone

One dashboard combining your Week 1 metrics panels with a new Logs panel, both scoped to the same incident window, where the log lines visibly explain the metric spike.

## Week 3 — Traces + Alerting

**Goal:** instrument a small app with distributed tracing, link traces to your existing logs/metrics, and build a real alert rule — comparing the mental model to Watcher/Kibana Alert Rules as you go, since that's the piece that transfers most directly from your Elastic background.

### Step 1: Stand up a small instrumented app

1. If you don't already have an app running, use a simple example — a small Flask/Express/Go HTTP service works fine, or Grafana's own demo app (the `grafana/xk6-otel-demo` or OpenTelemetry Demo repo) if you'd rather not write one.
2. Instrument it with the OpenTelemetry SDK for its language, configured to export traces via OTLP.

*Why a real app, not a synthetic trace generator:* traces are only meaningful when they represent actual request flow — a single hand-rolled trace teaches you nothing about correlation.

### Step 2: Route traces through Alloy to Tempo

1. Add an `otelcol.receiver.otlp` component to your `config.alloy` to accept traces from your app.
2. Add an `otelcol.exporter.otlp` component pointing at your Grafana Cloud Tempo endpoint, authenticated with your access token.
3. Point your app's OTLP exporter at Alloy's receiver endpoint (typically `localhost:4317` for gRPC).
4. Restart Alloy, generate a few requests against your app, and confirm traces appear in Grafana Cloud's **Explore → Tempo**.

### Step 3: Practice trace-to-logs and trace-to-metrics correlation

1. In Grafana, configure your Tempo data source's **derived fields** to link a trace's `trace_id` to a Loki query — this is what makes "jump from a slow trace to its logs" possible.
2. Open a trace in Explore, click the linked icon, and confirm it takes you to the matching log lines.
3. If your app exposes RED metrics (rate, errors, duration) via Prometheus, configure **exemplars** so a metrics graph can link a data point directly to the trace that produced it.

*Why this matters:* this three-way linking (metrics → traces → logs) is what "full observability" actually means in practice, versus three separate tools you have to manually cross-reference.

### Step 4: Build a native Grafana alert rule

1. Go to **Alerting → Alert rules → New alert rule**.
2. Base it on a real condition from your Week 1 metrics — e.g., CPU usage > 80% for 5 minutes.
3. Set the evaluation group and interval (how often the rule is checked).
4. Create a **contact point** — Slack webhook or email — and a **notification policy** routing this alert to it.
5. Force the alert to fire (rerun your Week 2 stress test) and confirm the notification arrives.

### Step 5: Compare to Watcher/Kibana Alert Rules

As you build the rule above, note the mapping to what you already know:

- Grafana's **query condition** ≈ Watcher's input/condition block
- Grafana's **evaluation interval** ≈ Watcher's schedule/trigger
- Grafana's **contact point + notification policy** ≈ Watcher's actions, but split into two decoupled objects — this separation (who gets notified vs. what triggers it) is arguably cleaner than Watcher's model, worth noting explicitly since it's a real design difference, not just renamed concepts.

### ✅ Milestone

An alert rule that fires on a real threshold breach and successfully delivers a notification to Slack or email, plus at least one trace successfully linked to its corresponding logs.

## Week 4 — Load Testing + Polish

**Goal:** generate real synthetic load with k6, watch it flow through everything you built in Weeks 1-3 simultaneously, then consolidate into one polished overview dashboard and a portfolio writeup.

### Step 1: Install k6

1. Install k6 locally (`brew install k6` on Mac, or via the install docs for Linux/Windows).
2. Confirm it works: `k6 run --vus 1 --duration 5s -e URL=https://example.com <(echo 'import http from "k6/http"; export default function(){ http.get(__ENV.URL); }')`

### Step 2: Write a load test script against your app

1. Create a simple k6 script (`load-test.js`) that sends repeated HTTP requests to the app you instrumented in Week 3.
2. Configure it with stages — ramp up, sustain, ramp down — e.g.:

```javascript
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },
    { duration: '1m', target: 20 },
    { duration: '30s', target: 0 },
  ],
};

export default function () {
  http.get('http://localhost:3000/');
  sleep(1);
}
```

### Step 3: Run the test and send results to Grafana Cloud

1. Use the `k6` Grafana Cloud output option so your test results (request rate, latency, error rate) land in Grafana Cloud alongside your other telemetry: `k6 run --out grafana-cloud load-test.js` (uses your Cloud k6 token from **My Account → k6**).
2. While the test runs, watch it live across your existing dashboards — your Week 1 host metrics should show increased CPU/load, Week 2 logs should show increased request volume, Week 3 traces should show many concurrent requests, and if your alert threshold is tuned right, it may even fire.

*Why this is the real test:* if your correlations from Weeks 2-3 were built correctly, this is the moment they all light up together under real load — confirming the whole setup actually works as one system, not three disconnected demos.

### Step 4: Build one consolidated "SRE-style" overview dashboard

1. Create a new dashboard combining:
   - Host metrics (from Week 1)
   - Request rate/error rate/duration from your app (RED metrics)
   - A logs panel scoped to errors only
   - A link or embedded panel showing recent traces
2. Add an annotation marking when your k6 test ran, so anyone looking at the dashboard later can see load-test windows at a glance.

*Why this matters:* this single dashboard is the artifact that demonstrates the whole month's work in one view — it's also exactly the kind of dashboard a real SRE/observability role expects you to be able to build.

### Step 5: Portfolio writeup (optional but recommended)

1. Write a short README on your GitHub (github.com/olajio) or hashtagdata.io covering: what you built, the architecture (host → Alloy → Grafana Cloud → dashboards/alerts), a screenshot of the final dashboard, and what you'd do differently at production scale.
2. Frame it around the federal/high-ingest environment you already work in day-to-day — drawing that explicit parallel is what makes this project land as job-search material, not just a learning exercise.

### ✅ Milestone

A load test that visibly shows up across metrics, logs, traces, and (ideally) triggers your alert — plus one consolidated overview dashboard tying the whole month together.

## Week 5 — Scale (Mimir), Kubernetes & Interview Prep

**Goal:** understand how Grafana's metrics backend scales past a single Prometheus instance (directly relevant to your federal, multi-cluster, \~500GB/day environment), get exposure to Kubernetes-native monitoring, and turn the project into interview-ready talking points.

### Step 1: Watch Course 1's Grafana Mimir section (1h9m)

Covers what Mimir is, monolithic vs. microservices deployment modes, multi-tenancy, S3-backed storage, and Mimir's Ruler/Alertmanager setup for alerting at scale.

*Why this matters for you specifically:* Grafana Cloud's Prometheus backend IS Mimir under the hood. Its multi-tenant, horizontally-scalable design is a close architectural parallel to isolating metrics per federal client cluster (SEC, OCC, FDIC, etc.) in your own environment.

### Step 2 (optional, hands-on): Deploy Mimir locally in monolithic mode

1. Follow the course walkthrough to run Mimir via Docker or binary.
2. Point a local Prometheus instance at it via remote\_write.
3. Skip this step if short on time — Grafana Cloud already runs Mimir for you, so the value here is architectural understanding, not a requirement for the rest of the track.

### Step 3: Watch Course 2's Kubernetes monitoring lectures (\~40 min)

Specifically: "Infrastructure Monitoring of Kubernetes," "Kubernetes Logging using Loki and Grafana," "Kubernetes Tracing using Tempo and Grafana," and "Kubernetes Infrastructure Monitoring using Grafana Cloud."

*Why:* neither the hands-on plan nor Course 1 touches Kubernetes, and it's a common target environment for observability roles — this closes that gap without committing to a full cluster build-out.

### Step 4 (optional, hands-on): Point Alloy at a local Kubernetes cluster

If you have Minikube or a spare cluster, deploy Alloy via its Kubernetes Helm chart and confirm it scrapes pod metrics — a light touch, not a production setup.

### Step 5: Watch "Grafana Interview Questions for 2026" (26 min)

For each question, write a one-to-two sentence answer grounded in what you actually built in Weeks 1-4 — e.g., for an alerting question, reference your real contact point + notification policy from Week 3, not just the textbook definition.

### Step 6: Finalize your portfolio writeup

Update the Week 4 README to mention the Mimir architecture understanding and Kubernetes exposure, so it reflects the full 5-week scope.

### ✅ Milestone

You can explain Mimir's multi-tenant architecture in your own words, have seen Kubernetes-native monitoring in action, and have written answers to a real set of Grafana interview questions grounded in your own project.

## 5-Week Schedule

Designed around \~6-9 hrs/week in course-heavy weeks, \~5-7 hrs/week otherwise.

| Week | Day | Focus | Est. Time |
| --- | --- | --- | --- |
| **1** | Mon | Watch Course 1: Foundations of Observability + Installing Prometheus sections | 1h 50m |
| 1 | Tue | Create Grafana Cloud account, generate access token, pick target host | 30 min |
| 1 | Wed | Watch Course 1: Installing & Configuring Grafana + Using Grafana sections | 1h 15m |
| 1 | Thu | Install & configure Alloy, configure Prometheus scrape + remote\_write | 1.5 hrs |
| 1 | Sat | Build dashboard from scratch (milestone) | 2 hrs |
| **2** | Mon | Watch Course 1: Grafana Loki section | 40 min |
| 2 | Tue | Add Loki write endpoint + log source to Alloy config | 45 min |
| 2 | Wed | Practice LogQL | 1 hr |
| 2 | Sat | Stress test + combined metrics/logs dashboard (milestone) | 1.5 hrs |
| **3** | Mon | Watch Course 1: Alloy for OTel + Tempo sections | 1h 8m |
| 3 | Tue | Instrument app, route traces through Alloy to Tempo | 1.5 hrs |
| 3 | Wed | Watch Course 1: Alerts, Notifications and Annotations section | 20 min |
| 3 | Thu | Configure trace-to-logs/metrics correlation | 1 hr |
| 3 | Sat | Build alert rule, force-fire it (milestone) | 1.5 hrs |
| **4** | Mon | Install k6, write first test script | 45 min |
| 4 | Tue | Run test with Grafana Cloud output | 1 hr |
| 4 | Wed | Build consolidated SRE-style overview dashboard | 1.5 hrs |
| 4 | Sat | Final load test run (milestone), start portfolio README | 1.5 hrs |
| **5** | Mon | Watch Course 1: Grafana Mimir section | 1h 9m |
| 5 | Tue | (Optional) Deploy Mimir locally | 1 hr |
| 5 | Wed | Watch Course 2: Kubernetes monitoring lectures | 40 min |
| 5 | Thu | (Optional) Point Alloy at a local Kubernetes cluster | 1 hr |
| 5 | Sat | Interview-questions lecture, write answers, finalize portfolio (milestone) | 2 hrs |

**Tip:** if a step runs long, don't compress the next one — push the week's Saturday session out a day instead. The milestones are the checkpoints that matter; the daily steps and videos are just the path there.
