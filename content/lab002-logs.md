
refer to demo-phases-1-4-Sept13-2026.md

phase by phase. 
configuring on `promgraf-vm`, in `/opt/promgraf`

first - phase 1

```
lance@promgraf-vm:/opt/promgraf$ sudo docker compose -f docker-compose.yaml -f docker-compose.demo.yml up -d demo-web nginx-exporter load-gen

[+] Running 14/14

 ✔ nginx-exporter Pulled                                                                                            2.7s 

   ✔ 49619ac15983 Pull complete                                                                                     1.6s 

   ✔ 13b555b32501 Pull complete                                                                                     0.4s 

   ✔ d2f7fc3af37c Download complete                                                                                 0.0s 

   ✔ 5be3366d8afd Download complete                                                                                 0.0s 

 ✔ demo-web Pulled                                                                                                  4.6s 

   ✔ f340c1b7c1d6 Pull complete                                                                                     0.3s 

   ✔ 956faab5efb3 Pull complete                                                                                     3.2s 

   ✔ a44b5c8be616 Pull complete                                                                                     0.3s 

   ✔ 02fc02c4ab8d Pull complete                                                                                     3.3s 

   ✔ c12f394dea35 Pull complete                                                                                     1.3s 

   ✔ 07db7bf2649b Pull complete                                                                                     0.3s 

   ✔ 76f27c02d218 Download complete                                                                                 0.0s 

   ✔ 25202a7045eb Download complete                                                                                 0.0s 

WARN[0005] Found orphan containers ([promgraf-snmp-exporter-1 promgraf-blackbox-exporter-1]) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up. 

[+] Running 4/4

 ✔ Container demo-app                   Running                                                                     0.0s 

 ✔ Container promgraf-nginx-exporter-1  Started                                                                    12.0s 

 ✔ Container load-gen                   Started                                                                    12.1s 

 ✔ Container promgraf-demo-web-1        Started                                                                    12.1s 

lance@promgraf-vm:/opt/promgraf$
```


just run these to see contents of all .yaml files in one go

```
echo "=== docker-compose.yaml ===" && cat /opt/promgraf/docker-compose.yaml
echo "=== docker-compose.demo.yml ===" && cat /opt/promgraf/docker-compose.demo.yml
echo "=== docker-compose.override.yml ===" && cat /opt/promgraf/docker-compose.override.yml
echo "=== prometheus/prometheus.yml ===" && cat /opt/promgraf/prometheus/prometheus.yml
```



```
lance@promgraf-vm:/opt/promgraf$ echo "=== docker-compose.yaml ===" && cat /opt/promgraf/docker-compose.yaml

=== docker-compose.yaml ===

services:

  smtp:

    image: ixdotai/smtp:v0.7.7@sha256:4f5f64a386ef8e024b60a45f9cac171f6229dd65eaad1d772337a1db64ba7f3d

    container_name: smtp

    ports:

      - 127.0.0.1:25:25

    env_file:

      - ./environments/smtp.env

  postgres:

    image: postgres:18@sha256:52e6ffd11fddd081ae63880b635b2a61c14008c17fc98cdc7ce5472265516dd0

    container_name: postgres

    restart: always

    environment:

      POSTGRES_DB: my_data_db

      POSTGRES_USER: my_data_user

      POSTGRES_PASSWORD: my_data_pwd

    ports:

      - "5488:5432"

    volumes:

      - postgres:/var/lib/postgresql/data

  loki:

    image: grafana/loki:3.7.1@sha256:73e905b51a7f917f7a1075e4be68759df30226e03dcb3cd2213b989cc0dc8eb4

    container_name: loki

    command:

      [

        "--validation.discover-service-name=[]",

        "--validation.discover-log-levels=false",

        "-config.file=/etc/loki/loki.yaml",

      ]

    volumes:

      - ./loki:/etc/loki/

    ports:

      - "3100:3100"

  prometheus:

    image: prom/prometheus:v3.11.2@sha256:5550dc63da361dc30f6fe02ac0e4dfc736ededfef3c8d12a634db04a67824d78

    container_name: prometheus

    volumes:

      - ./prometheus/:/etc/prometheus/

    command:

      - --web.enable-remote-write-receiver

      - --web.enable-lifecycle

      - --config.file=/etc/prometheus/prometheus.yml

    ports:

      - "9090:9090"

  alertmanager:

    image: prom/alertmanager:v0.33.0@sha256:af26fbe4dd1886ac0efd7bd55cd9027da262e105b137a376522b7c14c3626e4a

    container_name: alertmanager

    volumes:

      - ./alertmanager:/etc/alertmanager/

    ports:

      - 9093:9093

  grafana:

    image: grafana/grafana:${GRAFANA_VERSION:-12.3.1}

    #image: grafana/grafana-oss:main

    container_name: grafana

    environment:

      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin

      - GF_AUTH_ANONYMOUS_ENABLED=true

      - GF_AUTH_BASIC_ENABLED=false

      - GF_SMTP_ENABLED=true

      - GF_SMTP_HOST=smtp:25

      - GF_SMTP_SKIP_VERIFY=true

      - GF_FEATURE_TOGGLES_ENABLE=alertingCentralAlertHistory,sqlExpressions,alertingMigrationUI,alertingImportYAMLUI,grafanaManagedRecordingRules

    volumes:

      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards

      - ./grafana/alerting:/etc/grafana/provisioning/alerting

      - ./grafana/datasources:/etc/grafana/provisioning/datasources

      - ./grafana/grafana.ini:/etc/grafana/grafana.ini

    ports:

      - 3000:3000

volumes:

  postgres:

    driver: local

lance@promgraf-vm:/opt/promgraf$
```

```
lance@promgraf-vm:/opt/promgraf$ echo "=== docker-compose.demo.yml ===" && cat /opt/promgraf/docker-compose.demo.yml

=== docker-compose.demo.yml ===

services:

  demo-app:

    build: ./demo-app

    container_name: demo-app

    environment:

      DATABASE_URL: "postgresql://demo:demo@demo-db:5432/demo"

    networks:

      - default

    restart: unless-stopped

  

  demo-db:

    image: postgres:16

    container_name: demo-db

    environment:

      POSTGRES_USER: demo

      POSTGRES_PASSWORD: demo

      POSTGRES_DB: demo

      PGDATA: /var/lib/postgresql/data/pgdata

    networks:

      - default

    restart: unless-stopped

  

  demo-web:

    image: nginx:latest

    volumes:

      - ./demo-web/nginx.conf:/etc/nginx/conf.d/default.conf:ro

    networks:

      - default

  

  nginx-exporter:

    image: nginx/nginx-prometheus-exporter:latest

    command:

      - '--nginx.scrape-uri=http://demo-web:8080/stub_status'

    ports:

      - "9113:9113"

    networks:

      - default

  

  load-gen:

    image: alpine:latest

    container_name: load-gen

    command: sh -c "apk add --no-cache curl >/dev/null && while true; do curl -s demo-web:8080/ > /dev/null; sleep 0.3; done"

    depends_on:

      - demo-app

    networks:

      - default

    restart: unless-stopped

  

networks:

  default:

    name: promgraf_default

lance@promgraf-vm:/opt/promgraf$
```

```
lance@promgraf-vm:/opt/promgraf$ echo "=== docker-compose.override.yml ===" && cat /opt/promgraf/docker-compose.override.yml

=== docker-compose.override.yml ===

services:

  grafana:

    environment:

      - GF_AUTH_ANONYMOUS_ENABLED=true

      - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer

      - GF_AUTH_BASIC_ENABLED=true

      - GF_SECURITY_ADMIN_PASSWORD=CitiSRE

    volumes:

      - /data/grafana:/var/lib/grafana

  prometheus:

    volumes:

      - /data/prometheus:/prometheus

  alertmanager:

    volumes:

      - /data/alertmanager:/alertmanager

    # Alertmanager's HTTP client has no per-request DNS override (unlike

    # curl's --resolve), so pin n8n.lanc3.com to n8n-host's Tailscale IP

    # directly on the container. See note above TAILSCALE_AUTHKEY.

    extra_hosts:

      - "n8n.lanc3.com:100.122.88.54"

  # postgres:18's official image expects a single mount at /var/lib/postgresql

  # (it manages the versioned subdirectory itself) - the base compose file's

  # ./var/lib/postgresql/data mount is written for an older Postgres major

  # version and is now unused/stale. Adding the correct mount here rather

  # than fighting the merge to remove the old one; the leftover old mount

  # just sits empty and harmless.

  postgres:

    volumes:

      - /data/postgres:/var/lib/postgresql

  loki:

    volumes:

      - ./loki:/etc/loki/

      - /data/loki:/loki

  snmp-exporter:

    image: prom/snmp-exporter:latest

    ports:

      - "9116:9116"

    restart: unless-stopped

  

  blackbox-exporter:

    image: prom/blackbox-exporter:latest

    ports:

      - "9115:9115"

    restart: unless-stopped

lance@promgraf-vm:/opt/promgraf$
```

```
lance@promgraf-vm:/opt/promgraf$ echo "=== prometheus/prometheus.yml ===" && cat /opt/promgraf/prometheus/prometheus.yml

=== prometheus/prometheus.yml ===

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

      - targets: ['demo-app:8000']

  

  - job_name: nginx_exporter

    static_configs:

      - targets:

        - nginx-exporter:9113

lance@promgraf-vm:/opt/promgraf$
```

-------

phase 2

```
lance@promgraf-vm:/opt/promgraf$ cat /opt/promgraf/demo-app/app.py

import os, random, time

from flask import Flask

import psycopg2

from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

  

app = Flask(__name__)

  

DB_DSN = os.environ.get('DATABASE_URL', 'postgresql://demo:demo@demo-db:5432/demo')

REQS = Counter('app_requests_total', 'Total requests')

LATENCY = Histogram('app_request_latency_seconds', 'Request latency')

DB_LATENCY = Histogram('db_query_duration_seconds', 'DB query latency')

  

@DB_LATENCY.time()

def run_query(conn):

    cur = conn.cursor()

    cur.execute("SELECT pg_sleep(random() * 0.1); SELECT 1;")

    cur.close()

  

@app.route('/')

@LATENCY.time()

def index():

    REQS.inc()

    if random.random() < 0.15:

        time.sleep(random.uniform(0.3, 0.8))

    try:

        conn = psycopg2.connect(DB_DSN, connect_timeout=2)

        run_query(conn)

        conn.close()

        return "demo app ok"

    except Exception as e:

        return f"db error: {e}", 500

  

@app.route('/metrics')

def metrics():

    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

  

if __name__ == '__main__':

    app.run(host='0.0.0.0', port=8000)

lance@promgraf-vm:/opt/promgraf$
```


```
lance@promgraf-vm:/opt/promgraf$ sudo docker compose -f docker-compose.yaml -f docker-compose.demo.yml up -d --build demo-app

WARN[0000] Docker Compose is configured to build using Bake, but buildx isn't installed 

[+] Building 1.2s (10/10) FINISHED                                                                        docker:default

 => [demo-app internal] load build definition from Dockerfile                                                       0.1s

 => => transferring dockerfile: 196B                                                                                0.0s

 => [demo-app internal] load metadata for docker.io/library/python:3.12-slim                                        0.5s

 => [demo-app internal] load .dockerignore                                                                          0.0s

 => => transferring context: 2B                                                                                     0.0s

 => [demo-app 1/4] FROM docker.io/library/python:3.12-slim@sha256:78387bc3881b8273120a12ebe6c1ab22b018ccc2c9adf565  0.0s

 => => resolve docker.io/library/python:3.12-slim@sha256:78387bc3881b8273120a12ebe6c1ab22b018ccc2c9adf565ae1ac9b53  0.0s

 => [demo-app internal] load build context                                                                          0.0s

 => => transferring context: 1.16kB                                                                                 0.0s

 => CACHED [demo-app 2/4] WORKDIR /app                                                                              0.0s

 => CACHED [demo-app 3/4] RUN pip install --no-cache-dir flask psycopg2-binary prometheus_client                    0.0s

 => [demo-app 4/4] COPY app.py .                                                                                    0.0s

 => [demo-app] exporting to image                                                                                   0.3s

 => => exporting layers                                                                                             0.1s

 => => exporting manifest sha256:c8790d3ef489b07205a64dc30177e02570440be7da20d532f4ca814022146f09                   0.0s

 => => exporting config sha256:7be4b6845086fc0d863be3a863ee1693910e4a83248fd291e0e902eadd6a1564                     0.0s

 => => exporting attestation manifest sha256:68fee017716953ce44d48f2a2f81d9c89e5d6db7c2389bda86584ca6fbdb141a       0.0s

 => => exporting manifest list sha256:6b7286636233ce80a7c5ced96a1200ee0d2d923129d8171e355697153eaa124a              0.0s

 => => naming to docker.io/library/promgraf-demo-app:latest                                                         0.0s

 => => unpacking to docker.io/library/promgraf-demo-app:latest                                                      0.0s

 => [demo-app] resolving provenance for metadata file                                                               0.0s

[+] Running 1/1

 ✔ demo-app  Built                                                                                                  0.0s 

WARN[0001] Found orphan containers ([promgraf-snmp-exporter-1 promgraf-blackbox-exporter-1]) for this project. If you rem[+] Running 2/2 this service in your compose file, you can run this command with the --remove-orphans flag to clean it up ✔ demo-app            Built                                                                                        0.0s 

 ✔ Container demo-app  Started                                                                                     10.8s 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker exec load-gen sh -c 'for i in $(seq 1 20); do curl -s http://demo-web:8080/ > /dev/null; done'

curl -s http://localhost:9090/api/v1/query --data-urlencode 'query=db_query_duration_seconds_count' | python3 -m json.tool

{

    "status": "success",

    "data": {

        "resultType": "vector",

        "result": [

            {

                "metric": {

                    "__name__": "db_query_duration_seconds_count",

                    "instance": "demo-app:8000",

                    "job": "app_tier"

                },

                "value": [

                    1789274829.251,

                    "62"

                ]

            }

        ]

    }

}

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ curl -s http://localhost:8000/metrics 2>/dev/null | grep db_query_duration

lance@promgraf-vm:/opt/promgraf$
```

-----

phase 3

```
lance@promgraf-vm:/opt/promgraf$ cat -A /opt/promgraf/prometheus/prometheus.yml | sed -n '1,50p'

global:$

  scrape_interval: 1s$

  evaluation_interval: 5s$

$

alerting:$

  alertmanagers:$

    - static_configs:$

        - targets:$

            - 'alertmanager:9093'$

$

rule_files:$

  - 'rules/1.basic.yml'$

  #- 'rules/2.absent.yml'$

  #- 'rules/3.group.yml'$

  #- 'rules/4.templating.yml'$

  # restart prometheus$

$

scrape_configs:$

  - job_name: 'snmp_edge_router'$

    static_configs:$

      - targets: ['10.10.0.1']$

    metrics_path: /snmp$

    params:$

      module: [if_mib]$

    relabel_configs:$

      - source_labels: [__address__]$

        target_label: __param_target$

      - source_labels: [__param_target]$

        target_label: instance$

      - target_label: __address__$

        replacement: snmp-exporter:9116$

$

  - job_name: 'blackbox_web_tier'$

    metrics_path: /probe$

    params:$

      module: [http_2xx]$

    static_configs:$

      - targets: ['http://10.10.0.10/']$

    relabel_configs:$

      - source_labels: [__address__]$

        target_label: __param_target$

      - source_labels: [__param_target]$

        target_label: instance$

      - target_label: __address__$

        replacement: blackbox-exporter:9115$

$

  - job_name: 'app_tier'$

    static_configs:$

      - targets: ['demo-app:8000']$

$

lance@promgraf-vm:/opt/promgraf$
```

```
lance@promgraf-vm:/opt/promgraf$ cat  /opt/promgraf/prometheus/prometheus.yml

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

      - targets: ['demo-app:8000']

  

  - job_name: nginx_exporter

    static_configs:

      - targets:

        - nginx-exporter:9113

  

  - job_name: 'blackbox_icmp_internal'

    metrics_path: /probe

    params:

      module: [icmp]

    static_configs:

      - targets: ['demo-web', 'demo-app', 'demo-db']

    relabel_configs:

      - source_labels: [__address__]

        target_label: __param_target

      - source_labels: [__param_target]

        target_label: instance

      - target_label: __address__

        replacement: blackbox-exporter:9115

lance@promgraf-vm:/opt/promgraf$
```




```
lance@promgraf-vm:/opt/promgraf$ curl -X POST http://localhost:9090/-/reload

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep '"job"'

                    "job": "app_tier"

                    "job": "app_tier"

                    "job": "blackbox_icmp_internal"

                    "job": "blackbox_icmp_internal"

                    "job": "blackbox_icmp_internal"

                    "job": "blackbox_icmp_internal"

                    "job": "blackbox_icmp_internal"

                    "job": "blackbox_icmp_internal"

                    "job": "blackbox_web_tier"

                    "job": "blackbox_web_tier"

                    "job": "nginx_exporter"

                    "job": "nginx_exporter"

                    "job": "snmp_edge_router"

                    "job": "snmp_edge_router"

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ curl -s http://localhost:9090/api/v1/query --data-urlencode 'query=probe_success{job="blackbox_icmp_internal"}' | python3 -m json.tool

{

    "status": "success",

    "data": {

        "resultType": "vector",

        "result": [

            {

                "metric": {

                    "__name__": "probe_success",

                    "instance": "demo-db",

                    "job": "blackbox_icmp_internal"

                },

                "value": [

                    1789275936.211,

                    "1"

                ]

            },

            {

                "metric": {

                    "__name__": "probe_success",

                    "instance": "demo-app",

                    "job": "blackbox_icmp_internal"

                },

                "value": [

                    1789275936.211,

                    "1"

                ]

            },

            {

                "metric": {

                    "__name__": "probe_success",

                    "instance": "demo-web",

                    "job": "blackbox_icmp_internal"

                },

                "value": [

                    1789275936.211,

                    "1"

                ]

            }

        ]

    }

}

lance@promgraf-vm:/opt/promgraf$
```






```
lance@promgraf-vm:/opt/promgraf$ sudo docker compose -f docker-compose.yaml -f docker-compose.demo.yml up -d demo-app

WARN[0000] Found orphan containers ([promgraf-snmp-exporter-1 promgraf-blackbox-exporter-1]) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up. 

[+] Running 1/1

 ✔ Container demo-app  Started                                                                                     10.8s 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker exec demo-app which tc

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker exec demo-app apt-get update

Get:1 http://deb.debian.org/debian trixie InRelease [140 kB]

Get:2 http://deb.debian.org/debian trixie-updates InRelease [47.3 kB]

Get:3 http://deb.debian.org/debian-security trixie-security InRelease [43.4 kB]

Get:4 http://deb.debian.org/debian trixie/main amd64 Packages [9678 kB]

Get:5 http://deb.debian.org/debian trixie-updates/main amd64 Packages [4412 B]

Get:6 http://deb.debian.org/debian-security trixie-security/main amd64 Packages [261 kB]

Fetched 10.2 MB in 1s (7045 kB/s)

Reading package lists...

lance@promgraf-vm:/opt/promgraf$ sudo docker exec demo-app apt-get install -y iproute2

Reading package lists...

Building dependency tree...

Reading state information...

The following additional packages will be installed:

  krb5-locales libbpf1 libcap2 libcap2-bin libcom-err2 libelf1t64

  libgssapi-krb5-2 libk5crypto3 libkeyutils1 libkrb5-3 libkrb5support0 libmnl0

  libpam-cap libtirpc-common libtirpc3t64 libxtables12

Suggested packages:

  python3:any krb5-doc krb5-user

The following NEW packages will be installed:

  iproute2 krb5-locales libbpf1 libcap2-bin libcom-err2 libelf1t64

  libgssapi-krb5-2 libk5crypto3 libkeyutils1 libkrb5-3 libkrb5support0 libmnl0

  libpam-cap libtirpc-common libtirpc3t64 libxtables12

The following packages will be upgraded:

  libcap2

1 upgraded, 16 newly installed, 0 to remove and 11 not upgraded.

Need to get 2382 kB of archives.

After this operation, 8862 kB of additional disk space will be used.

Get:1 http://deb.debian.org/debian trixie/main amd64 libcap2 amd64 1:2.75-10+deb13u1+b3 [29.0 kB]

Get:2 http://deb.debian.org/debian trixie/main amd64 libelf1t64 amd64 0.192-4 [189 kB]

Get:3 http://deb.debian.org/debian trixie/main amd64 libbpf1 amd64 1:1.5.0-3 [169 kB]

Get:4 http://deb.debian.org/debian trixie/main amd64 libmnl0 amd64 1.0.5-3 [11.9 kB]

Get:5 http://deb.debian.org/debian trixie/main amd64 libkrb5support0 amd64 1.21.3-5+deb13u1 [33.1 kB]

Get:6 http://deb.debian.org/debian trixie/main amd64 libcom-err2 amd64 1.47.2-3+b12 [25.0 kB]

Get:7 http://deb.debian.org/debian trixie/main amd64 libk5crypto3 amd64 1.21.3-5+deb13u1 [81.2 kB]

Get:8 http://deb.debian.org/debian trixie/main amd64 libkeyutils1 amd64 1.6.3-6 [9456 B]

Get:9 http://deb.debian.org/debian trixie/main amd64 libkrb5-3 amd64 1.21.3-5+deb13u1 [326 kB]

Get:10 http://deb.debian.org/debian trixie/main amd64 libgssapi-krb5-2 amd64 1.21.3-5+deb13u1 [138 kB]

Get:11 http://deb.debian.org/debian trixie/main amd64 libtirpc-common all 1.3.6+ds-1 [11.0 kB]

Get:12 http://deb.debian.org/debian trixie/main amd64 libtirpc3t64 amd64 1.3.6+ds-1 [83.3 kB]

Get:13 http://deb.debian.org/debian trixie/main amd64 libxtables12 amd64 1.8.11-2 [31.9 kB]

Get:14 http://deb.debian.org/debian trixie/main amd64 libcap2-bin amd64 1:2.75-10+deb13u1+b3 [36.5 kB]

Get:15 http://deb.debian.org/debian trixie/main amd64 iproute2 amd64 6.15.0-1 [1089 kB]

Get:16 http://deb.debian.org/debian trixie/main amd64 krb5-locales all 1.21.3-5+deb13u1 [101 kB]

Get:17 http://deb.debian.org/debian trixie/main amd64 libpam-cap amd64 1:2.75-10+deb13u1+b3 [16.6 kB]

debconf: unable to initialize frontend: Dialog

debconf: (TERM is not set, so the dialog frontend is not usable.)

debconf: falling back to frontend: Readline

debconf: unable to initialize frontend: Readline

debconf: (Can't locate Term/ReadLine.pm in @INC (you may need to install the Term::ReadLine module) (@INC entries checked: /etc/perl /usr/local/lib/x86_64-linux-gnu/perl/5.40.1 /usr/local/share/perl/5.40.1 /usr/lib/x86_64-linux-gnu/perl5/5.40 /usr/share/perl5 /usr/lib/x86_64-linux-gnu/perl-base /usr/lib/x86_64-linux-gnu/perl/5.40 /usr/share/perl/5.40 /usr/local/lib/site_perl) at /usr/share/perl5/Debconf/FrontEnd/Readline.pm line 8, <STDIN> line 17.)

debconf: falling back to frontend: Teletype

debconf: unable to initialize frontend: Teletype

debconf: (This frontend requires a controlling tty.)

debconf: falling back to frontend: Noninteractive

Preconfiguring packages ...

Fetched 2382 kB in 0s (6592 kB/s)

(Reading database ... 5652 files and directories currently installed.)

Preparing to unpack .../libcap2_1%3a2.75-10+deb13u1+b3_amd64.deb ...

Unpacking libcap2:amd64 (1:2.75-10+deb13u1+b3) over (1:2.75-10+deb13u1+b1) ...

Setting up libcap2:amd64 (1:2.75-10+deb13u1+b3) ...

Selecting previously unselected package libelf1t64:amd64.

(Reading database ... 5652 files and directories currently installed.)

Preparing to unpack .../00-libelf1t64_0.192-4_amd64.deb ...

Unpacking libelf1t64:amd64 (0.192-4) ...

Selecting previously unselected package libbpf1:amd64.

Preparing to unpack .../01-libbpf1_1%3a1.5.0-3_amd64.deb ...

Unpacking libbpf1:amd64 (1:1.5.0-3) ...

Selecting previously unselected package libmnl0:amd64.

Preparing to unpack .../02-libmnl0_1.0.5-3_amd64.deb ...

Unpacking libmnl0:amd64 (1.0.5-3) ...

Selecting previously unselected package libkrb5support0:amd64.

Preparing to unpack .../03-libkrb5support0_1.21.3-5+deb13u1_amd64.deb ...

Unpacking libkrb5support0:amd64 (1.21.3-5+deb13u1) ...

Selecting previously unselected package libcom-err2:amd64.

Preparing to unpack .../04-libcom-err2_1.47.2-3+b12_amd64.deb ...

Unpacking libcom-err2:amd64 (1.47.2-3+b12) ...

Selecting previously unselected package libk5crypto3:amd64.

Preparing to unpack .../05-libk5crypto3_1.21.3-5+deb13u1_amd64.deb ...

Unpacking libk5crypto3:amd64 (1.21.3-5+deb13u1) ...

Selecting previously unselected package libkeyutils1:amd64.

Preparing to unpack .../06-libkeyutils1_1.6.3-6_amd64.deb ...

Unpacking libkeyutils1:amd64 (1.6.3-6) ...

Selecting previously unselected package libkrb5-3:amd64.

Preparing to unpack .../07-libkrb5-3_1.21.3-5+deb13u1_amd64.deb ...

Unpacking libkrb5-3:amd64 (1.21.3-5+deb13u1) ...

Selecting previously unselected package libgssapi-krb5-2:amd64.

Preparing to unpack .../08-libgssapi-krb5-2_1.21.3-5+deb13u1_amd64.deb ...

Unpacking libgssapi-krb5-2:amd64 (1.21.3-5+deb13u1) ...

Selecting previously unselected package libtirpc-common.

Preparing to unpack .../09-libtirpc-common_1.3.6+ds-1_all.deb ...

Unpacking libtirpc-common (1.3.6+ds-1) ...

Selecting previously unselected package libtirpc3t64:amd64.

Preparing to unpack .../10-libtirpc3t64_1.3.6+ds-1_amd64.deb ...

Adding 'diversion of /lib/x86_64-linux-gnu/libtirpc.so.3 to /lib/x86_64-linux-gnu/libtirpc.so.3.usr-is-merged by libtirpc3t64'

Adding 'diversion of /lib/x86_64-linux-gnu/libtirpc.so.3.0.0 to /lib/x86_64-linux-gnu/libtirpc.so.3.0.0.usr-is-merged by libtirpc3t64'

Unpacking libtirpc3t64:amd64 (1.3.6+ds-1) ...

Selecting previously unselected package libxtables12:amd64.

Preparing to unpack .../11-libxtables12_1.8.11-2_amd64.deb ...

Unpacking libxtables12:amd64 (1.8.11-2) ...

Selecting previously unselected package libcap2-bin.

Preparing to unpack .../12-libcap2-bin_1%3a2.75-10+deb13u1+b3_amd64.deb ...

Unpacking libcap2-bin (1:2.75-10+deb13u1+b3) ...

Selecting previously unselected package iproute2.

Preparing to unpack .../13-iproute2_6.15.0-1_amd64.deb ...

Unpacking iproute2 (6.15.0-1) ...

Selecting previously unselected package krb5-locales.

Preparing to unpack .../14-krb5-locales_1.21.3-5+deb13u1_all.deb ...

Unpacking krb5-locales (1.21.3-5+deb13u1) ...

Selecting previously unselected package libpam-cap:amd64.

Preparing to unpack .../15-libpam-cap_1%3a2.75-10+deb13u1+b3_amd64.deb ...

Unpacking libpam-cap:amd64 (1:2.75-10+deb13u1+b3) ...

Setting up libkeyutils1:amd64 (1.6.3-6) ...

Setting up libtirpc-common (1.3.6+ds-1) ...

Setting up krb5-locales (1.21.3-5+deb13u1) ...

Setting up libcom-err2:amd64 (1.47.2-3+b12) ...

Setting up libelf1t64:amd64 (0.192-4) ...

Setting up libkrb5support0:amd64 (1.21.3-5+deb13u1) ...

Setting up libcap2-bin (1:2.75-10+deb13u1+b3) ...

Setting up libmnl0:amd64 (1.0.5-3) ...

Setting up libk5crypto3:amd64 (1.21.3-5+deb13u1) ...

Setting up libxtables12:amd64 (1.8.11-2) ...

Setting up libkrb5-3:amd64 (1.21.3-5+deb13u1) ...

Setting up libpam-cap:amd64 (1:2.75-10+deb13u1+b3) ...

debconf: unable to initialize frontend: Dialog

debconf: (TERM is not set, so the dialog frontend is not usable.)

debconf: falling back to frontend: Readline

debconf: unable to initialize frontend: Readline

debconf: (Can't locate Term/ReadLine.pm in @INC (you may need to install the Term::ReadLine module) (@INC entries checked: /etc/perl /usr/local/lib/x86_64-linux-gnu/perl/5.40.1 /usr/local/share/perl/5.40.1 /usr/lib/x86_64-linux-gnu/perl5/5.40 /usr/share/perl5 /usr/lib/x86_64-linux-gnu/perl-base /usr/lib/x86_64-linux-gnu/perl/5.40 /usr/share/perl/5.40 /usr/local/lib/site_perl) at /usr/share/perl5/Debconf/FrontEnd/Readline.pm line 8.)

debconf: falling back to frontend: Teletype

debconf: unable to initialize frontend: Teletype

debconf: (This frontend requires a controlling tty.)

debconf: falling back to frontend: Noninteractive

Setting up libbpf1:amd64 (1:1.5.0-3) ...

Setting up libgssapi-krb5-2:amd64 (1.21.3-5+deb13u1) ...

Setting up libtirpc3t64:amd64 (1.3.6+ds-1) ...

Setting up iproute2 (6.15.0-1) ...

debconf: unable to initialize frontend: Dialog

debconf: (TERM is not set, so the dialog frontend is not usable.)

debconf: falling back to frontend: Readline

debconf: unable to initialize frontend: Readline

debconf: (Can't locate Term/ReadLine.pm in @INC (you may need to install the Term::ReadLine module) (@INC entries checked: /etc/perl /usr/local/lib/x86_64-linux-gnu/perl/5.40.1 /usr/local/share/perl/5.40.1 /usr/lib/x86_64-linux-gnu/perl5/5.40 /usr/share/perl5 /usr/lib/x86_64-linux-gnu/perl-base /usr/lib/x86_64-linux-gnu/perl/5.40 /usr/share/perl/5.40 /usr/local/lib/site_perl) at /usr/share/perl5/Debconf/FrontEnd/Readline.pm line 8.)

debconf: falling back to frontend: Teletype

debconf: unable to initialize frontend: Teletype

debconf: (This frontend requires a controlling tty.)

debconf: falling back to frontend: Noninteractive

Processing triggers for libc-bin (2.41-12+deb13u3) ...

lance@promgraf-vm:/opt/promgraf$
```






```
lance@promgraf-vm:/opt/promgraf$ sudo docker exec demo-app tc qdisc add dev eth0 root netem delay 40ms 10ms loss 2%

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker exec demo-app which tc

/usr/sbin/tc

lance@promgraf-vm:/opt/promgraf$ sudo docker inspect demo-app --format '{{.HostConfig.CapAdd}}'

[CAP_NET_ADMIN]

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker exec demo-app tc qdisc show dev eth0

qdisc netem 8001: root refcnt 3 limit 1000 delay 40ms  10ms loss 2% seed 16609973896226025455

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker exec load-gen sh -c 'for i in $(seq 1 30); do curl -s http://demo-web:8080/ > /dev/null; sleep 0.3; done'

sleep 10

curl -s http://localhost:9090/api/v1/query --data-urlencode 'query=probe_duration_seconds{job="blackbox_icmp_internal",instance="demo-app"}' | python3 -m json.tool

  

{

    "status": "success",

    "data": {

        "resultType": "vector",

        "result": [

            {

                "metric": {

                    "__name__": "probe_duration_seconds",

                    "instance": "demo-app",

                    "job": "blackbox_icmp_internal"

                },

                "value": [

                    1789276516.751,

                    "0.041696938"

                ]

            }

        ]

    }

}

lance@promgraf-vm:/opt/promgraf$
```


lance@promgraf-vm:/opt/promgraf$ for i in 1 2 3 4 5; do

  curl -s http://localhost:9090/api/v1/query --data-urlencode 'query=probe_success{job="blackbox_icmp_internal",instance="demo-app"}' | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['data']['result'][0]['value'])"

  sleep 5

done

[1789276579.232, '1']

[1789276584.271, '1']

[1789276589.312, '1']

[1789276594.349, '1']

[1789276599.386, '1']

lance@promgraf-vm:/opt/promgraf$

-------

phase 4 

— adding the network RTT and packet loss panels to Grafana alongside the web/DB latency panels from Phases 1-2

**Full Phase 4 panel set to add now:**

1. **Web-tier Request Rate**: `rate(nginx_http_requests_total[1m])`
2. **DB-tier Latency (p95)**: `histogram_quantile(0.95, rate(db_query_duration_seconds_bucket[5m]))`
3. **Network RTT per hop**: `probe_duration_seconds{job="blackbox_icmp_internal"}` — set as a multi-series time series, one line per `instance`
4. **Packet Loss**: `1 - avg_over_time(probe_success{job="blackbox_icmp_internal"}[5m])`

Add these four to your existing dashboard (Request Rate / p95 Latency / Target Health), same 3-column layout style.