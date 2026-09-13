
GNS3 

```
lance@promgraf-vm:/opt/promgraf$ sudo docker compose up -d grafana loki postgres prometheus alertmanager snmp-exporter blackbox-exporter

[+] Running 65/65

 ✔ prometheus Pulled                                                                                                                                              22.4s 

 ✔ alertmanager Pulled                                                                                                                                            10.3s 

 ✔ grafana Pulled                                                                                                                                                 27.9s 

 ✔ loki Pulled                                                                                                                                                    10.8s 

 ✔ postgres Pulled                                                                                                                                                22.4s 

[+] Running 8/8

 ✔ Volume promgraf_postgres                Created                                                                                                                 0.0s 

 ✔ Container postgres                      Started                                                                                                                 7.4s 

 ✔ Container promgraf-snmp-exporter-1      Running                                                                                                                 0.0s 

 ✔ Container promgraf-blackbox-exporter-1  Running                                                                                                                 0.0s 

 ✔ Container prometheus                    Started                                                                                                                 6.0s 

 ✔ Container alertmanager                  Started                                                                                                                 6.5s 

 ✔ Container grafana                       Started                                                                                                                 7.1s 

 ✔ Container loki                          Started                                                                                                                 6.9s 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker compose ps

NAME                           IMAGE                                                                                               COMMAND                  SERVICE             CREATED              STATUS              PORTS

alertmanager                   prom/alertmanager:v0.33.0@sha256:af26fbe4dd1886ac0efd7bd55cd9027da262e105b137a376522b7c14c3626e4a   "/bin/alertmanager -…"   alertmanager        About a minute ago   Up About a minute   0.0.0.0:9093->9093/tcp, [::]:9093->9093/tcp

grafana                        grafana/grafana:12.3.1                                                                              "/run.sh"                grafana             About a minute ago   Up About a minute   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp

loki                           grafana/loki:3.7.1@sha256:73e905b51a7f917f7a1075e4be68759df30226e03dcb3cd2213b989cc0dc8eb4          "/usr/bin/loki --val…"   loki                About a minute ago   Up About a minute   0.0.0.0:3100->3100/tcp, [::]:3100->3100/tcp

postgres                       postgres:18@sha256:52e6ffd11fddd081ae63880b635b2a61c14008c17fc98cdc7ce5472265516dd0                 "docker-entrypoint.s…"   postgres            About a minute ago   Up About a minute   0.0.0.0:5488->5432/tcp, [::]:5488->5432/tcp

prometheus                     prom/prometheus:v3.11.2@sha256:5550dc63da361dc30f6fe02ac0e4dfc736ededfef3c8d12a634db04a67824d78     "/bin/prometheus --w…"   prometheus          About a minute ago   Up About a minute   0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp

promgraf-blackbox-exporter-1   prom/blackbox-exporter:latest                                                                       "/bin/blackbox_expor…"   blackbox-exporter   16 minutes ago       Up 16 minutes       0.0.0.0:9115->9115/tcp, [::]:9115->9115/tcp

promgraf-snmp-exporter-1       prom/snmp-exporter:latest                                                                           "/bin/snmp_exporter …"   snmp-exporter       16 minutes ago       Up 16 minutes       0.0.0.0:9116->9116/tcp, [::]:9116->9116/tcp

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker network inspect promgraf_default | grep '"Name"'

        "Name": "promgraf_default",

                "Name": "alertmanager",

                "Name": "loki",

                "Name": "prometheus",

                "Name": "grafana",

                "Name": "postgres",

                "Name": "promgraf-snmp-exporter-1",

                "Name": "promgraf-blackbox-exporter-1",

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ date

Sat Sep 12 13:10:52 UTC 2026

lance@promgraf-vm:/opt/promgraf$
```

All 7 services are now up and confirmed on the same `promgraf_default` network — prometheus, grafana, alertmanager, loki, postgres, and both exporters.

Prometheus will be able to reach `snmp-exporter:9116` and `blackbox-exporter:9115` by name once we add the scrape jobs.


```
lance@promgraf-vm:/opt/promgraf$ sudo tee -a /opt/promgraf/prometheus/prometheus.yml > /dev/null <<'EOF'

  

scrape_configs:

  - job_name: 'snmp_edge_router'

    static_configs:

      - targets: ['10.10.0.1']

    metrics_path: /snmp

    params:

      module: [if_mib]

    relabel_configs:

      - source_labels: [__address__]

        target_label: __param_target

      - source_labels: [__param_target]

        target_label: instance

      - target_label: __address__

        replacement: snmp-exporter:9116

  

  - job_name: 'blackbox_web_tier'

    metrics_path: /probe

    params:

      module: [http_2xx]

    static_configs:

      - targets: ['http://10.10.0.10/']

    relabel_configs:

EOF   - targets: ['10.10.0.20:8000']er:9115

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo cat /opt/promgraf/prometheus/prometheus.yml

global:

  scrape_interval: 1s

  evaluation_interval: 5s

  

alerting:

  alertmanagers:

    - static_configs:

        - targets:

            - 'alertmanager:9093'

  

rule_files:

  - 'rules/1.basic.yml'

  #- 'rules/2.absent.yml'

  #- 'rules/3.group.yml'

  #- 'rules/4.templating.yml'

  # restart prometheus

  

scrape_configs:

  - job_name: 'snmp_edge_router'

    static_configs:

      - targets: ['10.10.0.1']

    metrics_path: /snmp

    params:

      module: [if_mib]

    relabel_configs:

      - source_labels: [__address__]

        target_label: __param_target

      - source_labels: [__param_target]

        target_label: instance

      - target_label: __address__

        replacement: snmp-exporter:9116

  

  - job_name: 'blackbox_web_tier'

    metrics_path: /probe

    params:

      module: [http_2xx]

    static_configs:

      - targets: ['http://10.10.0.10/']

    relabel_configs:

      - source_labels: [__address__]

        target_label: __param_target

      - source_labels: [__param_target]

        target_label: instance

      - target_label: __address__

        replacement: blackbox-exporter:9115

  

  - job_name: 'app_tier'

    static_configs:

      - targets: ['10.10.0.20:8000']

lance@promgraf-vm:/opt/promgraf$
```



```
lance@promgraf-vm:/opt/promgraf$ sudo docker compose kill -s SIGHUP prometheus

[+] Killing 1/1

 ✔ Container prometheus  Killed                                                                                                                                    1.1s 


```

SIGHUP reload worked: `status/config` confirms all three jobs — `snmp_edge_router`, `blackbox_web_tier`, `app_tier` — are loaded with the right targets, relabeling, and ports. Prometheus picked it up without needing a restart.


```
lance@promgraf-vm:/opt/promgraf$ curl -s http://localhost:9116/ | head -5

<html lang="en">

  <head>

    <meta charset="UTF-8">

    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>SNMP Exporter</title>

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ curl -s http://localhost:9115/ | head -5

<html>

    <head><title>Blackbox Exporter</title></head>

    <body>

    <h1>Blackbox Exporter</h1>

    <p><a href="probe?target=prometheus.io&module=http_2xx">Probe prometheus.io for http_2xx</a></p>

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ curl -s 'http://localhost:9090/api/v1/query?query=up' | jq

{

  "status": "success",

  "data": {

    "resultType": "vector",

    "result": [

      {

        "metric": {

          "__name__": "up",

          "instance": "http://10.10.0.10/",

          "job": "blackbox_web_tier"

        },

        "value": [

          1789219463.830,

          "1"

        ]

      },

      {

        "metric": {

          "__name__": "up",

          "instance": "10.10.0.20:8000",

          "job": "app_tier"

        },

        "value": [

          1789219463.830,

          "0"

        ]

      },

      {

        "metric": {

          "__name__": "up",

          "instance": "10.10.0.1",

          "job": "snmp_edge_router"

        },

        "value": [

          1789219463.830,

          "0"

        ]

      }

    ]

  }

}

lance@promgraf-vm:/opt/promgraf$
```

`up{job="blackbox_web_tier"}=1` is expected, but \It doesn't mean 10.10.0.10 is reachable (it isn't — no lab device exists there yet). 

Blackbox exporter works by proxy: 
Prometheus scrapes the _blackbox-exporter container's_ `/probe` endpoint, and `up` just reflects whether that scrape of the exporter itself succeeded — which it will, since the exporter is running, regardless of whether the actual target answers. 

The real result of the probe against 10.10.0.10 lives in a different metric: `probe_success`.

```
lance@promgraf-vm:/opt/promgraf$ curl -s 'http://localhost:9090/api/v1/query?query=probe_success' | jq

{

  "status": "success",

  "data": {

    "resultType": "vector",

    "result": [

      {

        "metric": {

          "__name__": "probe_success",

          "instance": "http://10.10.0.10/",

          "job": "blackbox_web_tier"

        },

        "value": [

          1789219655.202,

          "0"

        ]

      }

    ]

  }

}

lance@promgraf-vm:/opt/promgraf$
```

value `0` — that's what proves the probe of 10.10.0.10 genuinely failed, matching `snmp_edge_router` and `app_tier` correctly showing `up=0` for their direct scrapes.

----


Block 1 — build the app image:

```
lance@gns3-vm:~$ mkdir -p ~/lab-images/app-tier && cd ~/lab-images/app-tier

  

cat > requirements.txt <<'EOF'

flask

prometheus_client

psycopg2-binary

EOF

  

cat > app.py <<'EOF'

from flask import Flask

from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

import os, psycopg2

  

app = Flask(__name__)

REQUEST_COUNT = Counter('app_requests_total', 'Total requests')

REQUEST_LATENCY = Histogram('app_request_latency_seconds', 'Request latency')

  

@app.route('/health')

def health():

    return 'ok'

  

@app.route('/metrics')

def metrics():

docker build -t lab-app-tier:latest .uirements.txt'labuser', password='labpass')

[+] Building 11.2s (10/10) FINISHED                                                                                        docker:default

 => [internal] load build definition from Dockerfile                                                                                 0.1s

 => => transferring dockerfile: 200B                                                                                                 0.1s

 => [internal] load metadata for docker.io/library/python:3.12-slim                                                                  0.8s

 => [internal] load .dockerignore                                                                                                    0.0s

 => => transferring context: 2B                                                                                                      0.0s

 => [1/5] FROM docker.io/library/python:3.12-slim@sha256:78387bc3881b8273120a12ebe6c1ab22b018ccc2c9adf565ae1ac9b536e184ea            2.6s

 => => resolve docker.io/library/python:3.12-slim@sha256:78387bc3881b8273120a12ebe6c1ab22b018ccc2c9adf565ae1ac9b536e184ea            0.0s

 => => sha256:56235e4245636e401a28cf88944d543b29ee028ae3b6c27e03d6b43e7b4eac5f 249B / 249B                                           0.1s

 => => sha256:1560cfed01dc4c72c07f789d860078d74466a1f9a26b33a2e35d887b5f90dd3f 12.12MB / 12.12MB                                     0.3s

 => => sha256:dbd0b7849e6c06b61e1fa6b7ffe053f246ce0075890620e3db64b47bb662433a 4.27MB / 4.27MB                                       0.3s

 => => sha256:6310eb16bf4251731feab01e8f633bf5e2d75a657ccad97f420b1f83cce457be 29.79MB / 29.79MB                                     0.6s

 => => extracting sha256:6310eb16bf4251731feab01e8f633bf5e2d75a657ccad97f420b1f83cce457be                                            1.1s

 => => extracting sha256:dbd0b7849e6c06b61e1fa6b7ffe053f246ce0075890620e3db64b47bb662433a                                            0.2s

 => => extracting sha256:1560cfed01dc4c72c07f789d860078d74466a1f9a26b33a2e35d887b5f90dd3f                                            0.6s

 => => extracting sha256:56235e4245636e401a28cf88944d543b29ee028ae3b6c27e03d6b43e7b4eac5f                                            0.0s

 => [internal] load build context                                                                                                    0.0s

 => => transferring context: 996B                                                                                                    0.0s

 => [2/5] WORKDIR /app                                                                                                               1.1s

 => [3/5] COPY requirements.txt .                                                                                                    0.0s

 => [4/5] RUN pip install --no-cache-dir -r requirements.txt                                                                         4.2s

 => [5/5] COPY app.py .                                                                                                              0.1s 

 => exporting to image                                                                                                               2.2s 

 => => exporting layers                                                                                                              1.6s

 => => exporting manifest sha256:5eae9f30d6e84921b7221e802c26f31cd631ef3bae8516bc01e0efcc8e64a3db                                    0.0s

 => => exporting config sha256:5cc9eb2d113d1da1da3e47a9c2b69b4c0a14f38a390073478d8d06abe654437b                                      0.0s

 => => exporting attestation manifest sha256:e0e6928d1b718cfedbb29c8b29f93e8766200da63023b0342dca73d7d71e0f6f                        0.0s

 => => exporting manifest list sha256:439d6d38689d82adbf462c34d353f1aa59a4613c6eec36c1ca33e90a141981a5                               0.0s

 => => naming to docker.io/library/lab-app-tier:latest                                                                               0.0s

 => => unpacking to docker.io/library/lab-app-tier:latest                                                                            0.5s

lance@gns3-vm:~/lab-images/app-tier$ 

lance@gns3-vm:~/lab-images/app-tier$ 


```

Block 2 — confirm the image built:

```
lance@gns3-vm:~/lab-images/app-tier$ docker images | grep lab-app-tier

lab-app-tier:latest   439d6d38689d        226MB         55.8MB        

lance@gns3-vm:~/lab-images/app-tier$

```






Leave it empty. 
Dockerfile already defines `CMD ["python", "app.py"]` as the container's default command, and GNS3's "start command" field only exists to _override_ that default 
— it's for images where we want GNS3 to run something different than what's baked in (e.g. forcing a shell like `/bin/bash` for a barebones Alpine/Ubuntu image with no real entrypoint).

Since `python app.py` is exactly what we want running when the node boots, an empty field is correct here — GNS3 will just use the image's built-in `CMD`.



**in the GNS3 GUI** :

1. Edit → Preferences → Docker containers → New → existing image `lab-app-tier:latest`, 1 adapter.
2. Add a second Docker template using the official `postgres:16` image, env vars `POSTGRES_DB=labdb`, `POSTGRES_USER=labuser`, `POSTGRES_PASSWORD=labpass`.
3. Drag both onto the canvas, wire eth0–eth0 directly (no switch needed yet).
4. Start both nodes, then right-click each → Console.





```
lance@gns3-vm:~/lab-images/app-tier$ docker images

                                                                                                              i Info →   U  In Use

IMAGE                 ID             DISK USAGE   CONTENT SIZE   EXTRA

lab-app-tier:latest   439d6d38689d        226MB         55.8MB        

lance@gns3-vm:~/lab-images/app-tier$ 

lance@gns3-vm:~/lab-images/app-tier$ docker pull postgres:16

16: Pulling from library/postgres

af7708f26521: Pull complete 

9bd242f25e75: Pull complete 

988d57697ad2: Pull complete 

8a082189cfe1: Pull complete 

9f809ec5ff35: Pull complete 

f122bc3261fc: Pull complete 

4d710b756578: Pull complete 

5322d81b9b21: Pull complete 

3fa541d8cac9: Pull complete 

00034b17b963: Pull complete 

9d1e9c1e61f4: Pull complete 

48dd761b6bfe: Pull complete 

b3ffe1fd6e79: Pull complete 

344d251825bb: Download complete 

266e5ab4e0e8: Download complete 

Digest: sha256:f1c3376c26f2609ab9f29f71f824103fe2fcd8ee0346485cb6122a4f93df6f94

Status: Downloaded newer image for postgres:16

docker.io/library/postgres:16

lance@gns3-vm:~/lab-images/app-tier$ docker images

                                                                                                              i Info →   U  In Use

IMAGE                 ID             DISK USAGE   CONTENT SIZE   EXTRA

lab-app-tier:latest   439d6d38689d        226MB         55.8MB        

postgres:16           f1c3376c26f2        642MB          166MB        

lance@gns3-vm:~/lab-images/app-tier$
```


POSTGRES_DB=labdb

POSTGRES_USER=labuser

POSTGRES_PASSWORD=labpass  
PGDATA=/var/lib/postgresql/data/pgdata


-------

`app tier ok` proves the Flask app is serving _and_ successfully completed its `psycopg2.connect` + `SELECT 1` against `postgres-1` at 172.16.20.12, and `app_requests_total 1.0` confirms the metrics counter incremented correctly. 

evidence of real DB connection path rather than a synthetic ping.

```
/app # python3 -c "

> import urllib.request

> print(urllib.request.urlopen('http://localhost:8000/').read().decode())

> print(urllib.request.urlopen('http://localhost:8000/metrics').read().decode())

> "

app tier ok

# HELP python_gc_objects_collected_total Objects collected during gc

# TYPE python_gc_objects_collected_total counter

python_gc_objects_collected_total{generation="0"} 266.0

python_gc_objects_collected_total{generation="1"} 83.0

python_gc_objects_collected_total{generation="2"} 0.0

# HELP python_gc_objects_uncollectable_total Uncollectable objects found during GC

# TYPE python_gc_objects_uncollectable_total counter

python_gc_objects_uncollectable_total{generation="0"} 0.0

python_gc_objects_uncollectable_total{generation="1"} 0.0

python_gc_objects_uncollectable_total{generation="2"} 0.0

# HELP python_gc_collections_total Number of times this generation was collected

# TYPE python_gc_collections_total counter

python_gc_collections_total{generation="0"} 67.0

python_gc_collections_total{generation="1"} 6.0

python_gc_collections_total{generation="2"} 0.0

# HELP python_info Python platform information

# TYPE python_info gauge

python_info{implementation="CPython",major="3",minor="12",patchlevel="14",version="3.12.14"} 1.0

# HELP process_virtual_memory_bytes Virtual memory size in bytes.

# TYPE process_virtual_memory_bytes gauge

process_virtual_memory_bytes 1.49639168e+08

# HELP process_resident_memory_bytes Resident memory size in bytes.

# TYPE process_resident_memory_bytes gauge

process_resident_memory_bytes 4.3421696e+07

# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.

# TYPE process_start_time_seconds gauge

process_start_time_seconds 1.78922258297e+09

# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.

# TYPE process_cpu_seconds_total counter

process_cpu_seconds_total 0.6499999999999999

# HELP process_open_fds Number of open file descriptors.

# TYPE process_open_fds gauge

process_open_fds 6.0

# HELP process_max_fds Maximum number of open file descriptors.

# TYPE process_max_fds gauge

process_max_fds 1024.0

# HELP app_requests_total Total requests

# TYPE app_requests_total counter

app_requests_total 1.0

# HELP app_requests_created Total requests

# TYPE app_requests_created gauge

app_requests_created 1.7892225851835766e+09

# HELP app_request_latency_seconds Request latency

# TYPE app_request_latency_seconds histogram

app_request_latency_seconds_bucket{le="0.005"} 0.0

app_request_latency_seconds_bucket{le="0.01"} 0.0

app_request_latency_seconds_bucket{le="0.025"} 1.0

app_request_latency_seconds_bucket{le="0.05"} 1.0

app_request_latency_seconds_bucket{le="0.075"} 1.0

app_request_latency_seconds_bucket{le="0.1"} 1.0

app_request_latency_seconds_bucket{le="0.25"} 1.0

app_request_latency_seconds_bucket{le="0.5"} 1.0

app_request_latency_seconds_bucket{le="0.75"} 1.0

app_request_latency_seconds_bucket{le="1.0"} 1.0

app_request_latency_seconds_bucket{le="2.5"} 1.0

app_request_latency_seconds_bucket{le="5.0"} 1.0

app_request_latency_seconds_bucket{le="7.5"} 1.0

app_request_latency_seconds_bucket{le="10.0"} 1.0

app_request_latency_seconds_bucket{le="+Inf"} 1.0

app_request_latency_seconds_count 1.0

app_request_latency_seconds_sum 0.02058687099997769

# HELP app_request_latency_seconds_created Request latency

# TYPE app_request_latency_seconds_created gauge

app_request_latency_seconds_created 1.7892225851836207e+09

  

/app #
```