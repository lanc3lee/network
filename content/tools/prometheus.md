
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

k6 is an open-source load-testing tool from Grafana Labs — its usual job is generating synthetic traffic against an API or service to see how it holds up under load (spawning virtual users, hitting endpoints, measuring response times). 

Write the test scenario as a JavaScript file, and `k6 run` executes it.

In this repo, though, it's being used slightly outside its normal purpose: 
instead of testing an application under load, the scripts use an extension (`k6/x/remotewrite`) to push fake metric values _directly_ into Prometheus's remote-write API. 

So rather than "hit this API 1000 times and measure latency," these scripts are really just "generate synthetic CPU-usage numbers on a schedule and inject them straight into the time-series database" 
— a convenient way to make Prometheus/Grafana/Alertmanager have data to alert on and visualize, without needing to stand up a real monitored application first.

So for our purposes here, think of k6 less as "the load tester" and more as "the fake-data generator that lets us see the alerting pipeline actually fire"

For now, just a simple practice test. 

here's a screenshot of a new grafana dashboard built to monitor CPU usage. 
![[grafana-dashboard-cpu-usage-00.png]]

**8. Trigger an actual alert**  
Find where the repo defines its alert rules (check the `prometheus/` or `alerting/` folder in the repo for the rules file), then either:

- Stop a container to trigger a "target down"-style alert, or
- Look for a documented way in the repo's README to force a threshold breach

Then watch it flow: Prometheus `Alerts` tab (pending → firing) → Alertmanager `http://localhost:9093` (grouped/routed) → wherever the repo routes notifications (likely a stub/webhook, check Alertmanager config for the receiver).

**9. Read the two config files that matter most**  
Once it's running and you trust it works, open:

-  `alertmanager.yml` — compare it against the routing/grouping/inhibition concepts we discussed
- alert rules file — compare the PromQL patterns against `rate()`, `for:`, thresholds

**10. Tear down when done**

bash

```bash
docker-compose down
```

Add `-v` if you also want to wipe stored data/volumes for a clean restart later.

![[docker-compose-down.png]]

--------------

**Exercises to practise with**

```
lance@LANC3 demo-prometheus-and-grafana-alerts % ls testdata/

1.cpu-usage.js 
2.send-logs.js 
3.add-instances.js 
3.add-instances2.js 
4.resolve-alerts.js

```

nano testdata/1.cpu-usage.js 
to see what's written in the javascript
![[K6-cpu-usage.png]```
import { check, sleep } from "k6";

import remote from "k6/x/remotewrite";  

export let options = {

  iterations: 500

};
  

const client = new remote.Client({

  url: "http://localhost:9090/api/v1/write",

});

// Example query:

// avg_over_time(cpu_usage[5m]) > 80

export default function () {

    sendMetricData("server1", Math.floor(Math.random() * 21) + 80); // 80-100

}

function sendMetricData(instanceValue, value) {

  const res = client.store([

    {

      labels: [

        { name: "__name__", value: `cpu_usage` },

        { name: "job", value: "exporter" },

        { name: "instance", value: instanceValue },

      ],

      samples: [{ value: value }],

    },

  ]);

  check(res, {

    "is status 204": (r) => r.status === 204,

  });

  sleep(0.001);

}
```

```

![[K6-cpu-usage.png]]


![[prometheus-CPU-alerts.png]]

![[AlertManager-CPU-alerts.png]]



![[grafana-data-sources.png]]
In grafana, check for Data sources under Connections. 

Since Alertmanager and Prometheus are listed (see above), especially as Prometheus is listed as default data source, you can expect to "cpu_usage" as metric (see below)


![[grafana-select-CPU-usage.png]]![[grafana-data-sources.png]]

here's the grafana dashboard visualization of the CPU alerts

![[grafana-dashboard-cpu-usage.png]]

----

Below are other .js scripts you can run to generate or simulate alerts

2.send-logs.js 
3.add-instances.js 
3.add-instances2.js 
4.resolve-alerts.js


```
lance@LANC3 demo-prometheus-and-grafana-alerts % ls testdata

1.cpu-usage.js 2.send-logs.js 3.add-instances.js 3.add-instances2.js 4.resolve-alerts.js

lance@LANC3 demo-prometheus-and-grafana-alerts % cat testdata/2.send-logs.js 

import { check } from "k6";

import loki from "k6/x/loki";

  

let labels = loki.Labels({

  format: ["logfmt"],

  // detected_level: ["warn", "error"],

  // instance: ["foo", "bar"],

  detected_level: ["error"],

  instance: ["foo"],

  service_name: ["backend"],

});

  

// Example query:

// count_over_time({detected_level="error", service_name="backend"}[1m])

  

const conf = new loki.Config("http://localhost:3100", 10000, 1.0, {}, labels);

const client = new loki.Client(conf);

  

export default () => {

  // push random data (~800-900 log lines) according to the labels

  const res = client.push();

  check(res, { "successful write": (res) => res.status == 204 });

};

lance@LANC3 demo-prometheus-and-grafana-alerts % 

lance@LANC3 demo-prometheus-and-grafana-alerts % cat testdata/3.add-instances.js 

import { check, sleep } from "k6";

import remote from "k6/x/remotewrite";

  

export let options = {

  iterations: 500

};

  

const client = new remote.Client({

  url: "http://localhost:9090/api/v1/write",

});

  

// Example query:

// avg_over_time(cpu_usage[5m]) > 80

  

export default function () {

    sendMetricData("server1", Math.floor(Math.random() * 21) + 80); // 80-100

    sendMetricData("server2", Math.floor(Math.random() * 21) + 0); // 0-20

    sendMetricData("server3", Math.floor(Math.random() * 21) + 80); // 80-100

}

  

function sendMetricData(instanceValue, value) {

  const res = client.store([

    {

      labels: [

        { name: "__name__", value: `cpu_usage` },

        { name: "job", value: "exporter" },

        { name: "instance", value: instanceValue },

      ],

      samples: [{ value: value }],

    },

  ]);

  check(res, {

    "is status 204": (r) => r.status === 204,

  });

  sleep(0.001);

}

lance@LANC3 demo-prometheus-and-grafana-alerts % cat testdata/3.add-instances2.js

import { check, sleep } from "k6";

import remote from "k6/x/remotewrite";

  

export let options = {

  iterations: 500

};

  

const client = new remote.Client({

  url: "http://localhost:9090/api/v1/write",

});

  

// avg_over_time(cpu_usage[5m])

// exclude region

// - avg(avg_over_time(cpu_usage[5m])) by (instance)

// - avg(avg_over_time(cpu_usage[5m]) ) without (region)

  

export default function () {

    sendMetricData("server1", Math.floor(Math.random() * 21) + 80); // 80-100

    sendMetricData("server2", Math.floor(Math.random() * 21) + 0); // 0-20

    sendMetricData("server3", Math.floor(Math.random() * 21) + 80); // 80-100

}

  

function sendMetricData(instanceValue, value) {

  const regions = ["emea", "amer", "apac"];

  const res = client.store([

    {

      labels: [

        { name: "__name__", value: `cpu_usage` },

        { name: "job", value: "exporter" },

        { name: "region", value: regions[Math.floor(Math.random() * regions.length)] },

        { name: "instance", value: instanceValue },

      ],

      samples: [{ value: value }],

    },

  ]);

  

  check(res, {

    "is status 204": (r) => r.status === 204,

  });

  sleep(0.001);

}

lance@LANC3 demo-prometheus-and-grafana-alerts % cat testdata/4.resolve-alerts.js 

import { check, sleep } from "k6";

import remote from "k6/x/remotewrite";

  

export let options = {

  iterations: 500

};

  

const client = new remote.Client({

  url: "http://localhost:9090/api/v1/write",

});

  

// Example query:

// avg_over_time(cpu_usage[5m]) > 80

  

export default function () {

    sendMetricData("server1", Math.floor(Math.random() * 21) + 0); // 0-20

    sendMetricData("server2", Math.floor(Math.random() * 21) + 0); // 0-20

    sendMetricData("server3", Math.floor(Math.random() * 21) + 0); // 0-20

}

  

function sendMetricData(instanceValue, value) {

  const res = client.store([

    {

      labels: [

        { name: "__name__", value: `cpu_usage` },

        { name: "job", value: "exporter" },

        { name: "instance", value: instanceValue },

      ],

      samples: [{ value: value }],

    },

  ]);

  check(res, {

    "is status 204": (r) => r.status === 204,

  });

  sleep(0.001);

}

lance@LANC3 demo-prometheus-and-grafana-alerts %
```