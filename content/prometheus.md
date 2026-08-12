
Network operations teams have spent decades treating monitoring as a polling problem — an NMS reaches out to each device on a fixed interval, pulls back a snapshot, and raises an alarm when a threshold is crossed. 

That model still works, but it was built for a world of hundreds of devices, not the thousands of interfaces, containers, and services that make up a modern infrastructure fabric.

**Prometheus**, **Alertmanager**, and **Grafana** offers a open-source answer to that shift: a purpose-built pipeline for collecting metrics as continuous time series.
It decides intelligently what deserves a human's attention, and visualises all of it in a way that turns raw numbers into operational insight.

Each tool plays a distinct role in that pipeline. 

**Prometheus** pulls metrics from targets on a scrape interval and stores them as timestamped series — the same conceptual data an SNMP poller collects, but with a query language (PromQL) built specifically for computing rates, thresholds, and trends over that data rather than just displaying a current value. 

![[promethus-9090.png]]

**Alertmanager** sits downstream of Prometheus and handles what happens once a rule condition is met: grouping related alerts so a single root cause doesn't flood an on-call engineer with duplicate pages, routing different alert types to the right team or channel, and suppressing lower-priority alerts when a more severe, related one is already firing. 

![[alert-manager.png]]


**Grafana** completes the loop as the visualization layer — dashboards that turn those same time series into something a NOC can watch in real time, correlate across metrics, and drill into during an incident.

![[grafana.png]]


What makes this stack worth understanding, beyond the individual pieces, is how cleanly it maps onto problems network engineers already know by different names. Alertmanager's inhibition rules do, in software, roughly what topology-aware root-cause correlation does in commercial fault-management platforms — just expressed as explicit configuration rather than derived automatically. And because the whole stack is open source, free to run locally, and genuinely used in production at scale by companies well beyond the traditional NMS vendor world, it's one of the most practical ways to build hands-on intuition for how modern time-series monitoring actually works — not just read about it.

**1. Prerequisites check**

```bash
docker --version
docker compose version
```

![[docker-verify.png|265]]


**2. Clone the repo**

bash

```bash
cd ~/Documents
git clone https://github.com/grafana/demo-prometheus-and-grafana-alerts.git
cd demo-prometheus-and-grafana-alerts
```

![[github-promethus.png|468]]

**3. Look before you launch**

bash

```bash
ls -la
cat docker-compose.yml
```

this is the "read before you run" habit 

![[docker-compose-yaml.png]]

Worth a skim to know what's about to start

**4. Copy the required example config files**

The repo ships `.example` templates instead of the live files it expects — this step is easy to miss and causes the `env file not found` error:

bash

```bash
cp environments/smtp.env.example environments/smtp.env
cp alertmanager/smtp_auth_password.example alertmanager/smtp_auth_password
```


**5. Start the stack**

bash

```bash
docker-compose up -d
```

This pulls the images first time (may take a couple minutes), then starts everything in the background.

**5. Confirm everything's actually running**

bash

```bash
docker-compose ps
```

All services should show `Up` — if anything shows `Exit` or `Restarting`, check its logs before moving on:

bash

```bash
docker-compose logs <service-name>
```

**6. Open each UI and get familiar with UI**

- Prometheus: `http://localhost:9090`
- Alertmanager: `http://localhost:9093`
- Grafana: `http://localhost:3000`

![[promethus-9090.png|562]]


![[alert-manager.png|561]]

![[grafana.png|551]]

In Grafana, check **Connections → Data Sources** to confirm Prometheus (and Loki, since this repo includes it) are already wired up via provisioning — you shouldn't need to add them manually.

![[grafana-data-sources.png]]


**7. Watch live data land**  
The repo uses k6 to generate synthetic load, so within a minute or two you should see metrics moving in Prometheus (`Status → Targets` to confirm scrape health) and panels populating in Grafana's pre-built dashboards.

**8. Trigger an actual alert**  
Find where the repo defines its alert rules (check the `prometheus/` or `alerting/` folder in the repo for the rules file), then either:

- Stop a container to trigger a "target down"-style alert, or
- Look for a documented way in the repo's README to force a threshold breach

Then watch it flow: Prometheus `Alerts` tab (pending → firing) → Alertmanager `http://localhost:9093` (grouped/routed) → wherever the repo routes notifications (likely a stub/webhook, check their Alertmanager config for the receiver).

**9. Read the two config files that matter most for your interview prep**  
Once it's running and you trust it works, open:

- Their `alertmanager.yml` — compare it against the routing/grouping/inhibition concepts we discussed
- Their alert rules file — compare the PromQL patterns against `rate()`, `for:`, thresholds

**10. Tear down when done**

bash

```bash
docker-compose down
```

Add `-v` if you also want to wipe stored data/volumes for a clean restart later.

