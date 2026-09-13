
```

lance@LANC3 prom-grafana % terraform plan

  

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:

  + create

  

Terraform will perform the following actions:

  

  # google_compute_disk.promgraf_data will be created

  + resource "google_compute_disk" "promgraf_data" {

      + access_mode                 = (known after apply)

      + creation_timestamp          = (known after apply)

      + disk_id                     = (known after apply)

      + effective_labels            = (known after apply)

      + enable_confidential_compute = (known after apply)

      + id                          = (known after apply)

      + label_fingerprint           = (known after apply)

      + last_attach_timestamp       = (known after apply)

      + last_detach_timestamp       = (known after apply)

      + licenses                    = (known after apply)

      + name                        = "promgraf-data-disk"

      + physical_block_size_bytes   = (known after apply)

      + project                     = "eve-ng-lab-lanc3"

      + provisioned_iops            = (known after apply)

      + provisioned_throughput      = (known after apply)

      + self_link                   = (known after apply)

      + size                        = 30

      + source_disk_id              = (known after apply)

      + source_image_id             = (known after apply)

      + source_snapshot_id          = (known after apply)

      + terraform_labels            = (known after apply)

      + type                        = "pd-standard"

      + users                       = (known after apply)

      + zone                        = "us-central1-a"

  

      + guest_os_features (known after apply)

    }

  

  # google_compute_firewall.allow_iap_ssh will be created

  + resource "google_compute_firewall" "allow_iap_ssh" {

      + creation_timestamp = (known after apply)

      + destination_ranges = (known after apply)

      + direction          = (known after apply)

      + enable_logging     = (known after apply)

      + id                 = (known after apply)

      + name               = "promgraf-allow-iap-ssh"

      + network            = "default"

      + priority           = 1000

      + project            = "eve-ng-lab-lanc3"

      + self_link          = (known after apply)

      + source_ranges      = [

          + "35.235.240.0/20",

        ]

      + target_tags        = [

          + "iap-ssh",

        ]

  

      + allow {

          + ports    = [

              + "22",

            ]

          + protocol = "tcp"

        }

    }

  

  # google_compute_instance.promgraf will be created

  + resource "google_compute_instance" "promgraf" {

      + can_ip_forward          = false

      + cpu_platform            = (known after apply)

      + current_status          = (known after apply)

      + deletion_protection     = false

      + effective_labels        = (known after apply)

      + guest_accelerator       = (known after apply)

      + id                      = (known after apply)

      + instance_id             = (known after apply)

      + label_fingerprint       = (known after apply)

      + machine_type            = "e2-standard-2"

      + metadata                = {

          + "cloudflare-tunnel-token" = (sensitive value)

          + "grafana-admin-password"  = (sensitive value)

        }

      + metadata_fingerprint    = (known after apply)

      + metadata_startup_script = <<-EOT

            #!/bin/bash

            set -e

            # Mount the pre-formatted data disk if it isn't already mounted.

            # NOTE: this script deliberately does NOT run mkfs. Format the disk manually

            # once after first boot (see README.md), then this just mounts it on every

            # subsequent start.

            mkdir -p /data

            if ! mountpoint -q /data; then

              mount /dev/disk/by-id/google-promgraf-data /data 2>/tmp/data-mount-error.log || true

            fi

            if mountpoint -q /data; then

              logger -t promgraf-startup "/data is mounted OK - stack will persist to the data disk."

            else

              logger -p user.warning -t promgraf-startup "WARNING: /data is NOT mounted. The stack is about to write Prometheus/Grafana/Alertmanager/Postgres data to the BOOT disk instead of the persistent disk. Format the data disk per README step 4, then re-run: sudo google_metadata_script_runner startup"

            fi

            apt-get update -y

            apt-get install -y docker.io docker-compose-plugin git curl

            # Install cloudflared if not already present

            if ! command -v cloudflared &> /dev/null; then

              curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o /tmp/cloudflared.deb

              dpkg -i /tmp/cloudflared.deb

            fi

            TUNNEL_TOKEN=$(curl -s -H "Metadata-Flavor: Google" \

              "http://metadata.google.internal/computeMetadata/v1/instance/attributes/cloudflare-tunnel-token")

            GRAFANA_ADMIN_PASSWORD=$(curl -s -H "Metadata-Flavor: Google" \

              "http://metadata.google.internal/computeMetadata/v1/instance/attributes/grafana-admin-password")

            if [ -n "$TUNNEL_TOKEN" ] && [ ! -f /etc/cloudflared/cert.json ]; then

              cloudflared service install "$TUNNEL_TOKEN" || true

            fi

            # Clone the demo stack repo if not already present

            [ -d /opt/promgraf ] || git clone https://github.com/grafana/demo-prometheus-and-grafana-alerts.git /opt/promgraf

            # --- Persist data to /data instead of the boot disk -----------------------

            # The upstream docker-compose.yaml doesn't bind Prometheus, Grafana,

            # Alertmanager, or Postgres data directories to a host path at all - it

            # relies on Docker's anonymous/named volumes, which live under

            # /var/lib/docker on the BOOT disk (not /data). This override redirects all

            # four onto the persistent disk. NOTE: Loki's on-disk path depends on the

            # custom loki.yaml this repo ships - check /opt/promgraf/loki/loki.yaml's

            # storage_config directory value on the VM and add a matching bind mount

            # here if it needs to survive stop/start too.

            mkdir -p /data/prometheus /data/grafana /data/alertmanager /data/postgres

            chown -R 65534:65534 /data/prometheus /data/alertmanager   # prometheus + alertmanager images run as nobody

            chown -R 472:472 /data/grafana                              # grafana image's default uid

            chown -R 999:999 /data/postgres                              # official postgres image's default uid

            # If any container logs "permission denied" on first boot, re-check its

            # actual runtime UID with `docker inspect <container> --format '{{.Config.User}}'`

            # and chown /data/<service> to match.

            # Grafana's upstream default is GF_AUTH_ANONYMOUS_ORG_ROLE=Admin and

            # GF_AUTH_BASIC_ENABLED=false (i.e. anonymous visitors get full Admin, and

            # the username/password login form is disabled entirely). This override

            # downgrades anonymous access to Viewer and re-enables basic auth so you

            # still have an admin login path once anonymous access is public.

            cat > /opt/promgraf/docker-compose.override.yml <<EOF

            services:

              grafana:

                environment:

                  - GF_AUTH_ANONYMOUS_ENABLED=true

                  - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer

                  - GF_AUTH_BASIC_ENABLED=true

                  - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}

                volumes:

                  - /data/grafana:/var/lib/grafana

              prometheus:

                volumes:

                  - /data/prometheus:/prometheus

              alertmanager:

                volumes:

                  - /data/alertmanager:/alertmanager

            volumes:

              postgres:

                driver_opts:

                  type: none

                  device: /data/postgres

                  o: bind

            EOF

            cd /opt/promgraf && docker compose up -d

        EOT

      + min_cpu_platform        = (known after apply)

      + name                    = "promgraf-vm"

      + project                 = "eve-ng-lab-lanc3"

      + self_link               = (known after apply)

      + tags                    = [

          + "iap-ssh",

          + "promgraf",

        ]

      + tags_fingerprint        = (known after apply)

      + terraform_labels        = (known after apply)

      + zone                    = "us-central1-a"

  

      + attached_disk {

          + device_name                = "promgraf-data"

          + disk_encryption_key_sha256 = (known after apply)

          + kms_key_self_link          = (known after apply)

          + mode                       = "READ_WRITE"

          + source                     = (known after apply)

        }

  

      + boot_disk {

          + auto_delete                = true

          + device_name                = (known after apply)

          + disk_encryption_key_sha256 = (known after apply)

          + kms_key_self_link          = (known after apply)

          + mode                       = "READ_WRITE"

          + source                     = (known after apply)

  

          + initialize_params {

              + image                  = "ubuntu-os-cloud/ubuntu-2404-lts-amd64"

              + labels                 = (known after apply)

              + provisioned_iops       = (known after apply)

              + provisioned_throughput = (known after apply)

              + size                   = 20

              + type                   = (known after apply)

            }

        }

  

      + confidential_instance_config (known after apply)

  

      + network_interface {

          + internal_ipv6_prefix_length = (known after apply)

          + ipv6_access_type            = (known after apply)

          + ipv6_address                = (known after apply)

          + name                        = (known after apply)

          + network                     = "default"

          + network_ip                  = (known after apply)

          + stack_type                  = (known after apply)

          + subnetwork                  = (known after apply)

          + subnetwork_project          = (known after apply)

        }

  

      + reservation_affinity (known after apply)

  

      + scheduling (known after apply)

  

      + service_account {

          + email  = (known after apply)

          + scopes = [

              + "https://www.googleapis.com/auth/cloud-platform",

            ]

        }

    }

  

Plan: 3 to add, 0 to change, 0 to destroy.

  

Changes to Outputs:

  + data_disk_name     = "promgraf-data-disk"

  + instance_name      = "promgraf-vm"

  + instance_self_link = (known after apply)

  + ssh_command        = "gcloud compute ssh promgraf-vm --zone=us-central1-a --tunnel-through-iap"

  

─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

  

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run "terraform apply" now.

lance@LANC3 prom-grafana % 

lance@LANC3 prom-grafana % 

lance@LANC3 prom-grafana % terraform apply

  

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:

  + create

  

Terraform will perform the following actions:

  

  # google_compute_disk.promgraf_data will be created

  + resource "google_compute_disk" "promgraf_data" {

      + access_mode                 = (known after apply)

      + creation_timestamp          = (known after apply)

      + disk_id                     = (known after apply)

      + effective_labels            = (known after apply)

      + enable_confidential_compute = (known after apply)

      + id                          = (known after apply)

      + label_fingerprint           = (known after apply)

      + last_attach_timestamp       = (known after apply)

      + last_detach_timestamp       = (known after apply)

      + licenses                    = (known after apply)

      + name                        = "promgraf-data-disk"

      + physical_block_size_bytes   = (known after apply)

      + project                     = "eve-ng-lab-lanc3"

      + provisioned_iops            = (known after apply)

      + provisioned_throughput      = (known after apply)

      + self_link                   = (known after apply)

      + size                        = 30

      + source_disk_id              = (known after apply)

      + source_image_id             = (known after apply)

      + source_snapshot_id          = (known after apply)

      + terraform_labels            = (known after apply)

      + type                        = "pd-standard"

      + users                       = (known after apply)

      + zone                        = "us-central1-a"

  

      + guest_os_features (known after apply)

    }

  

  # google_compute_firewall.allow_iap_ssh will be created

  + resource "google_compute_firewall" "allow_iap_ssh" {

      + creation_timestamp = (known after apply)

      + destination_ranges = (known after apply)

      + direction          = (known after apply)

      + enable_logging     = (known after apply)

      + id                 = (known after apply)

      + name               = "promgraf-allow-iap-ssh"

      + network            = "default"

      + priority           = 1000

      + project            = "eve-ng-lab-lanc3"

      + self_link          = (known after apply)

      + source_ranges      = [

          + "35.235.240.0/20",

        ]

      + target_tags        = [

          + "iap-ssh",

        ]

  

      + allow {

          + ports    = [

              + "22",

            ]

          + protocol = "tcp"

        }

    }

  

  # google_compute_instance.promgraf will be created

  + resource "google_compute_instance" "promgraf" {

      + can_ip_forward          = false

      + cpu_platform            = (known after apply)

      + current_status          = (known after apply)

      + deletion_protection     = false

      + effective_labels        = (known after apply)

      + guest_accelerator       = (known after apply)

      + id                      = (known after apply)

      + instance_id             = (known after apply)

      + label_fingerprint       = (known after apply)

      + machine_type            = "e2-standard-2"

      + metadata                = {

          + "cloudflare-tunnel-token" = (sensitive value)

          + "grafana-admin-password"  = (sensitive value)

        }

      + metadata_fingerprint    = (known after apply)

      + metadata_startup_script = <<-EOT

            #!/bin/bash

            set -e

            # Mount the pre-formatted data disk if it isn't already mounted.

            # NOTE: this script deliberately does NOT run mkfs. Format the disk manually

            # once after first boot (see README.md), then this just mounts it on every

            # subsequent start.

            mkdir -p /data

            if ! mountpoint -q /data; then

              mount /dev/disk/by-id/google-promgraf-data /data 2>/tmp/data-mount-error.log || true

            fi

            if mountpoint -q /data; then

              logger -t promgraf-startup "/data is mounted OK - stack will persist to the data disk."

            else

              logger -p user.warning -t promgraf-startup "WARNING: /data is NOT mounted. The stack is about to write Prometheus/Grafana/Alertmanager/Postgres data to the BOOT disk instead of the persistent disk. Format the data disk per README step 4, then re-run: sudo google_metadata_script_runner startup"

            fi

            apt-get update -y

            apt-get install -y docker.io docker-compose-plugin git curl

            # Install cloudflared if not already present

            if ! command -v cloudflared &> /dev/null; then

              curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o /tmp/cloudflared.deb

              dpkg -i /tmp/cloudflared.deb

            fi

            TUNNEL_TOKEN=$(curl -s -H "Metadata-Flavor: Google" \

              "http://metadata.google.internal/computeMetadata/v1/instance/attributes/cloudflare-tunnel-token")

            GRAFANA_ADMIN_PASSWORD=$(curl -s -H "Metadata-Flavor: Google" \

              "http://metadata.google.internal/computeMetadata/v1/instance/attributes/grafana-admin-password")

            if [ -n "$TUNNEL_TOKEN" ] && [ ! -f /etc/cloudflared/cert.json ]; then

              cloudflared service install "$TUNNEL_TOKEN" || true

            fi

            # Clone the demo stack repo if not already present

            [ -d /opt/promgraf ] || git clone https://github.com/grafana/demo-prometheus-and-grafana-alerts.git /opt/promgraf

            # --- Persist data to /data instead of the boot disk -----------------------

            # The upstream docker-compose.yaml doesn't bind Prometheus, Grafana,

            # Alertmanager, or Postgres data directories to a host path at all - it

            # relies on Docker's anonymous/named volumes, which live under

            # /var/lib/docker on the BOOT disk (not /data). This override redirects all

            # four onto the persistent disk. NOTE: Loki's on-disk path depends on the

            # custom loki.yaml this repo ships - check /opt/promgraf/loki/loki.yaml's

            # storage_config directory value on the VM and add a matching bind mount

            # here if it needs to survive stop/start too.

            mkdir -p /data/prometheus /data/grafana /data/alertmanager /data/postgres

            chown -R 65534:65534 /data/prometheus /data/alertmanager   # prometheus + alertmanager images run as nobody

            chown -R 472:472 /data/grafana                              # grafana image's default uid

            chown -R 999:999 /data/postgres                              # official postgres image's default uid

            # If any container logs "permission denied" on first boot, re-check its

            # actual runtime UID with `docker inspect <container> --format '{{.Config.User}}'`

            # and chown /data/<service> to match.

            # Grafana's upstream default is GF_AUTH_ANONYMOUS_ORG_ROLE=Admin and

            # GF_AUTH_BASIC_ENABLED=false (i.e. anonymous visitors get full Admin, and

            # the username/password login form is disabled entirely). This override

            # downgrades anonymous access to Viewer and re-enables basic auth so you

            # still have an admin login path once anonymous access is public.

            cat > /opt/promgraf/docker-compose.override.yml <<EOF

            services:

              grafana:

                environment:

                  - GF_AUTH_ANONYMOUS_ENABLED=true

                  - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer

                  - GF_AUTH_BASIC_ENABLED=true

                  - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}

                volumes:

                  - /data/grafana:/var/lib/grafana

              prometheus:

                volumes:

                  - /data/prometheus:/prometheus

              alertmanager:

                volumes:

                  - /data/alertmanager:/alertmanager

            volumes:

              postgres:

                driver_opts:

                  type: none

                  device: /data/postgres

                  o: bind

            EOF

            cd /opt/promgraf && docker compose up -d

        EOT

      + min_cpu_platform        = (known after apply)

      + name                    = "promgraf-vm"

      + project                 = "eve-ng-lab-lanc3"

      + self_link               = (known after apply)

      + tags                    = [

          + "iap-ssh",

          + "promgraf",

        ]

      + tags_fingerprint        = (known after apply)

      + terraform_labels        = (known after apply)

      + zone                    = "us-central1-a"

  

      + attached_disk {

          + device_name                = "promgraf-data"

          + disk_encryption_key_sha256 = (known after apply)

          + kms_key_self_link          = (known after apply)

          + mode                       = "READ_WRITE"

          + source                     = (known after apply)

        }

  

      + boot_disk {

          + auto_delete                = true

          + device_name                = (known after apply)

          + disk_encryption_key_sha256 = (known after apply)

          + kms_key_self_link          = (known after apply)

          + mode                       = "READ_WRITE"

          + source                     = (known after apply)

  

          + initialize_params {

              + image                  = "ubuntu-os-cloud/ubuntu-2404-lts-amd64"

              + labels                 = (known after apply)

              + provisioned_iops       = (known after apply)

              + provisioned_throughput = (known after apply)

              + size                   = 20

              + type                   = (known after apply)

            }

        }

  

      + confidential_instance_config (known after apply)

  

      + network_interface {

          + internal_ipv6_prefix_length = (known after apply)

          + ipv6_access_type            = (known after apply)

          + ipv6_address                = (known after apply)

          + name                        = (known after apply)

          + network                     = "default"

          + network_ip                  = (known after apply)

          + stack_type                  = (known after apply)

          + subnetwork                  = (known after apply)

          + subnetwork_project          = (known after apply)

        }

  

      + reservation_affinity (known after apply)

  

      + scheduling (known after apply)

  

      + service_account {

          + email  = (known after apply)

          + scopes = [

              + "https://www.googleapis.com/auth/cloud-platform",

            ]

        }

    }

  

Plan: 3 to add, 0 to change, 0 to destroy.

  

Changes to Outputs:

  + data_disk_name     = "promgraf-data-disk"

  + instance_name      = "promgraf-vm"

  + instance_self_link = (known after apply)

  + ssh_command        = "gcloud compute ssh promgraf-vm --zone=us-central1-a --tunnel-through-iap"

  

Do you want to perform these actions?

  Terraform will perform the actions described above.

  Only 'yes' will be accepted to approve.

  

  Enter a value: yes

  

google_compute_firewall.allow_iap_ssh: Creating...

google_compute_disk.promgraf_data: Creating...

google_compute_disk.promgraf_data: Creation complete after 4s [id=projects/eve-ng-lab-lanc3/zones/us-central1-a/disks/promgraf-data-disk]

google_compute_instance.promgraf: Creating...

google_compute_firewall.allow_iap_ssh: Still creating... [00m10s elapsed]

google_compute_firewall.allow_iap_ssh: Creation complete after 12s [id=projects/eve-ng-lab-lanc3/global/firewalls/promgraf-allow-iap-ssh]

google_compute_instance.promgraf: Still creating... [00m10s elapsed]

google_compute_instance.promgraf: Still creating... [00m20s elapsed]

google_compute_instance.promgraf: Still creating... [00m30s elapsed]

google_compute_instance.promgraf: Creation complete after 31s [id=projects/eve-ng-lab-lanc3/zones/us-central1-a/instances/promgraf-vm]

  

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

  

Outputs:

  

data_disk_name = "promgraf-data-disk"

instance_name = "promgraf-vm"

instance_self_link = "https://www.googleapis.com/compute/v1/projects/eve-ng-lab-lanc3/zones/us-central1-a/instances/promgraf-vm"

ssh_command = "gcloud compute ssh promgraf-vm --zone=us-central1-a --tunnel-through-iap"

lance@LANC3 prom-grafana %



lance@LANC3 prom-grafana % gcloud compute ssh promgraf-vm --zone=us-central1-a --tunnel-through-iap
...

 System information as of Sat Sep 12 00:40:58 UTC 2026

  

  System load:  0.0                Processes:             113

  Usage of /:   13.0% of 18.33GB   Users logged in:       0

  Memory usage: 3%                 IPv4 address for ens4: 10.128.0.2

  Swap usage:   0%

  
...
  

lance@promgraf-vm:~$

lance@promgraf-vm:~$ lsblk

NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS

loop0     7:0    0  66.8M  1 loop /snap/core24/1643

loop1     7:1    0 332.2M  1 loop /snap/google-cloud-cli/493

loop2     7:2    0  50.1M  1 loop /snap/snapd/27710

sda       8:0    0    20G  0 disk 

├─sda1    8:1    0    19G  0 part /

├─sda14   8:14   0     4M  0 part 

├─sda15   8:15   0   106M  0 part /boot/efi

└─sda16 259:0    0   913M  0 part /boot

sdb       8:16   0    30G  0 disk 

lance@promgraf-vm:~$

lance@promgraf-vm:~$ ls -la /dev/disk/by-id/ | grep google

lrwxrwxrwx  1 root root   9 Sep 12 00:32 google-persistent-disk-0 -> ../../sda

lrwxrwxrwx  1 root root  10 Sep 12 00:32 google-persistent-disk-0-part1 -> ../../sda1

lrwxrwxrwx  1 root root  11 Sep 12 00:32 google-persistent-disk-0-part14 -> ../../sda14

lrwxrwxrwx  1 root root  11 Sep 12 00:32 google-persistent-disk-0-part15 -> ../../sda15

lrwxrwxrwx  1 root root  11 Sep 12 00:32 google-persistent-disk-0-part16 -> ../../sda16

lrwxrwxrwx  1 root root   9 Sep 12 00:32 google-promgraf-data -> ../../sdb

lance@promgraf-vm:~$


lance@promgraf-vm:~$ sudo mkfs.ext4 -m 0 /dev/disk/by-id/google-promgraf-data

sudo mkdir -p /data

sudo mount /dev/disk/by-id/google-promgraf-data /data

echo '/dev/disk/by-id/google-promgraf-data /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab

mke2fs 1.47.0 (5-Feb-2023)

Discarding device blocks: done                            

Creating filesystem with 7864320 4k blocks and 1966080 inodes

Filesystem UUID: b4e53e83-2498-4858-88e1-614a10841b61

Superblock backups stored on blocks: 

32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 

4096000

  

Allocating group tables: done                            

Writing inode tables: done                            

Creating journal (32768 blocks): done

Writing superblocks and filesystem accounting information: done   

  

/dev/disk/by-id/google-promgraf-data /data ext4 defaults,nofail 0 2

lance@promgraf-vm:~$



lance@promgraf-vm:~$ df -h /data

mount | grep /data

Filesystem      Size  Used Avail Use% Mounted on

/dev/sdb         30G   24K   30G   1% /data

/dev/sdb on /data type ext4 (rw,relatime)

lance@promgraf-vm:~$


```

Adding NAT router

```
lance@LANC3 prom-grafana % terraform apply

google_compute_firewall.allow_iap_ssh: Refreshing state... [id=projects/eve-ng-lab-lanc3/global/firewalls/promgraf-allow-iap-ssh]

google_compute_disk.promgraf_data: Refreshing state... [id=projects/eve-ng-lab-lanc3/zones/us-central1-a/disks/promgraf-data-disk]

google_compute_instance.promgraf: Refreshing state... [id=projects/eve-ng-lab-lanc3/zones/us-central1-a/instances/promgraf-vm]


Terraform will perform the following actions:

  

  # google_compute_router.default will be created

  + resource "google_compute_router" "default" {

      + creation_timestamp = (known after apply)

      + id                 = (known after apply)

      + name               = "promgraf-nat-router"

      + network            = "default"

      + project            = "eve-ng-lab-lanc3"

      + region             = "us-central1"

      + self_link          = (known after apply)

    }

  

  # google_compute_router_nat.default will be created

  + resource "google_compute_router_nat" "default" {

      + auto_network_tier                   = (known after apply)

      + enable_dynamic_port_allocation      = (known after apply)

      + enable_endpoint_independent_mapping = (known after apply)

      + endpoint_types                      = (known after apply)

      + icmp_idle_timeout_sec               = 30

      + id                                  = (known after apply)

      + min_ports_per_vm                    = (known after apply)

      + name                                = "promgraf-nat"

      + nat_ip_allocate_option              = "AUTO_ONLY"

      + project                             = "eve-ng-lab-lanc3"

      + region                              = "us-central1"

      + router                              = "promgraf-nat-router"

      + source_subnetwork_ip_ranges_to_nat  = "ALL_SUBNETWORKS_ALL_IP_RANGES"

      + tcp_established_idle_timeout_sec    = 1200

      + tcp_time_wait_timeout_sec           = 120

      + tcp_transitory_idle_timeout_sec     = 30

      + udp_idle_timeout_sec                = 30

    }

  

Plan: 2 to add, 0 to change, 0 to destroy.

  


google_compute_router.default: Creating...

google_compute_router.default: Creation complete after 3s [id=projects/eve-ng-lab-lanc3/regions/us-central1/routers/promgraf-nat-router]

google_compute_router_nat.default: Creating...

google_compute_router_nat.default: Still creating... [00m10s elapsed]

google_compute_router_nat.default: Creation complete after 18s [id=eve-ng-lab-lanc3/us-central1/promgraf-nat-router/promgraf-nat]

  

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

  

Outputs:

  

data_disk_name = "promgraf-data-disk"

instance_name = "promgraf-vm"

instance_self_link = "https://www.googleapis.com/compute/v1/projects/eve-ng-lab-lanc3/zones/us-central1-a/instances/promgraf-vm"

ssh_command = "gcloud compute ssh promgraf-vm --zone=us-central1-a --tunnel-through-iap"

lance@LANC3 prom-grafana %
```


------

```
lance@promgraf-vm:/opt/promgraf$ sudo docker compose up -d prometheus alertmanager grafana loki postgres

[+] Running 65/65

 ✔ postgres Pulled                                                                                                                                  33.7s 

 ✔ loki Pulled                                                                                                                                      17.5s 

 ✔ alertmanager Pulled                                                                                                                              13.0s 

 ✔ grafana Pulled                                                                                                                                   46.3s 

 ✔ prometheus Pulled                                                                                                                                30.2s 

[+] Running 7/7

 ✔ Network promgraf_default  Created                                                                                                                 0.3s 

 ✔ Volume promgraf_postgres  Created                                                                                                                 0.0s 

 ✔ Container grafana         Started                                                                                                                 7.6s 

 ✔ Container postgres        Started                                                                                                                 7.6s 

 ✔ Container loki            Started                                                                                                                 7.6s 

 ✔ Container prometheus      Started                                                                                                                 7.6s 

 ✔ Container alertmanager    Started                                                                                                                 7.6s 

lance@promgraf-vm:/opt/promgraf$

```

```
lance@promgraf-vm:/opt/promgraf$ sudo journalctl -u cloudflared --no-pager | grep -i "registered tunnel connection"

Sep 12 01:58:44 promgraf-vm.us-central1-a.c.eve-ng-lab-lanc3.internal cloudflared[2573]: 2026-09-12T01:58:44Z INF Registered tunnel connection connIndex=0 connection=7865826d-9af5-429c-bafb-e6ad2e6f3024 event=0 ip=198.41.192.47 location=ord06 protocol=quic

Sep 12 01:58:45 promgraf-vm.us-central1-a.c.eve-ng-lab-lanc3.internal cloudflared[2573]: 2026-09-12T01:58:45Z INF Registered tunnel connection connIndex=1 connection=5b1a4977-51d1-4ccc-880b-48cc8c86cf69 event=0 ip=198.41.200.13 location=ord08 protocol=quic

Sep 12 01:58:46 promgraf-vm.us-central1-a.c.eve-ng-lab-lanc3.internal cloudflared[2573]: 2026-09-12T01:58:46Z INF Registered tunnel connection connIndex=2 connection=6949fac3-d6c4-49bf-9451-96fb8bb23cdc event=0 ip=198.41.192.107 location=ord16 protocol=quic

Sep 12 01:58:47 promgraf-vm.us-central1-a.c.eve-ng-lab-lanc3.internal cloudflared[2573]: 2026-09-12T01:58:47Z INF Registered tunnel connection connIndex=3 connection=5da405c4-19fc-4722-8139-71823a45cf96 event=0 ip=198.41.200.63 location=ord07 protocol=quic

lance@promgraf-vm:/opt/promgraf$
```

---

```
lance@promgraf-vm:/opt/promgraf$ curl -I https://grafana.lanc3.com

HTTP/2 200 

date: Sat, 12 Sep 2026 02:24:22 GMT

content-type: text/html; charset=UTF-8

cache-control: no-store

x-content-type-options: nosniff

x-frame-options: deny

x-xss-protection: 1; mode=block

cf-cache-status: DYNAMIC

report-to: {"group":"cf-nel","max_age":604800,"endpoints":[{"url":"https://a.nel.cloudflare.com/report/v4?s=pRblcekvUuQ17P545mRD8v3MEyJj8%2FkQLsucAsHe9AK4VgUJeCuR7upZKB0c9DteOT%2BWmP8xXf3P0XdYE9ESdyrEW4vTiad28xk%2BnNW81PgA01GdnFD8zx5op1M%2F3gM1WqvVQw%3D%3D"}]}

nel: {"report_to":"cf-nel","success_fraction":0.0,"max_age":604800}

server: cloudflare

cf-ray: a39b7497f93c5eb1-ORD

alt-svc: h3=":443"; ma=86400

  

lance@promgraf-vm:/opt/promgraf$
```

-----

security issue exposed

```
lance@promgraf-vm:/opt/promgraf$ curl -I https://prom-api.lanc3.com/-/healthy

HTTP/2 200 

date: Sat, 12 Sep 2026 02:25:36 GMT

cf-cache-status: DYNAMIC

report-to: {"group":"cf-nel","max_age":604800,"endpoints":[{"url":"https://a.nel.cloudflare.com/report/v4?s=TuZhDSAblexBz75fEZUE7LrV0bI3a2zsUYVVuwwDgSP9%2BN0pBVMmkBgF%2FMIxmbefPQbe%2BFYylugipb8gdnDXZGKjYUw4kLRi7%2FXtGdKSyN3HZ8ex%2FaZqW2Os1TrdaSfjxmzMiZc%3D"}]}

nel: {"report_to":"cf-nel","success_fraction":0.0,"max_age":604800}

server: cloudflare

cf-ray: a39b76677b147a5f-ORD

alt-svc: h3=":443"; ma=86400

  

lance@promgraf-vm:/opt/promgraf$
```

Prometheus's HTTP API is fully open on the public internet right now, no auth at all. Given `prometheus.yml`'s `--web.enable-remote-write-receiver` flag is active (from the base compose command), anyone who finds this URL can currently push arbitrary fake metrics into your Prometheus. This is live and urgent — let's lock it down now before touching anything else.

**Step 1 — Create a Service Token** (the credential n8n will use to authenticate):

1. Cloudflare dashboard → **Zero Trust** → **Access** → **Service Auth** → **Service Tokens**.
2. **Create Service Token**, name it something identifiable like `n8n-prom-query`.
3. Cloudflare shows a **Client ID** and **Client Secret** — copy both now; the secret is shown only once.

**Step 2 — Create an Access Application gating the hostname:**

1. **Zero Trust** → **Access** → **Applications** → **Add an application** → **Self-hosted**.
2. **Application domain**: `prom-api.lanc3.com`
3. Under **Policies**, create one policy:
    - **Action**: Allow
    - **Include**: Service Auth → select the `n8n-prom-query` token you just created
4. Save.

Once that's live, requests to `prom-api.lanc3.com` need two headers to get through — `CF-Access-Client-Id` and `CF-Access-Client-Secret` (the values from Step 1) — anything without them gets a Cloudflare Access login/deny page instead of reaching Prometheus. That's exactly what n8n's HTTP Request node config should send

---

```
curl -I -L https://prom-api.lanc3.com/-/healthy \
  -H "CF-Access-Client-Id: xxxxx.access" \
  -H "CF-Access-Client-Secret: cfast_yyyyy"
```

to confirm that prom-api.lanc3.com can only be accessed with credentials

-------

handoff prompt

**Project: Prometheus/Grafana/Alertmanager PoC on GCP — "AI-assisted latency triage pipeline" 

**Status: stack is live and verified reachable. Next up: GNS3 deployment + finishing loose ends below.**

**Infrastructure:**

- GCP project `eve-ng-lab-lanc3`, VM `promgraf-vm` (e2-standard-2, us-central1-a, no public IP, IAP SSH only)
- Terraform at `~/Documents/gcp-infra/prom-grafana/` — includes a Cloud Router + Cloud NAT (scoped to `default` network/us-central1 only) added after discovering the project had none
- 30GB persistent disk mounted at `/data`, manually formatted once (not automated in `startup.sh`, deliberately)
- Stack: Grafana's `demo-prometheus-and-grafana-alerts` repo — Prometheus, Alertmanager, Grafana, Loki, Postgres all running via `docker compose up -d` (the `smtp` service is excluded — dead upstream image, not needed since alerting routes to n8n, not email)

**Fixed along the way (all committed in `startup.sh`/`main.tf`):**

- Ubuntu package name is `docker-compose-v2`, not `docker-compose-plugin`
- All four data-bearing services now bind-mount to `/data/*` with correct UID ownership (upstream compose didn't persist any of them by default)
- Postgres 18 needed a mount at `/var/lib/postgresql` (not `.../data`) using the `!override` YAML tag to fully replace the base compose's stale mount — Compose merges/appends by default, doesn't replace
- Grafana's upstream defaults (anonymous **Admin** access, login form disabled) fixed — anonymous is now Viewer-only, real admin password set via a new `grafana_admin_password` sensitive Terraform var

**Exposure — both working:**

- `grafana.lanc3.com` — public, anonymous Viewer, confirmed HTTP 200
- `prom-api.lanc3.com` — gated behind Cloudflare Access (Service Token policy `LANC3-n8n-prom-query`), confirmed working end-to-end (403 without creds, 200 with valid service token headers)


**Other loose ends:**

- Loki's actual data path wasn't verified/fixed for persistence — depends on its own `loki.yaml` in the repo, still on the boot disk as far as we know
- Alertmanager → n8n webhook receiver itself not yet built/tested (n8n-lanc3 project, separate)
- Network emulation decision made: **GNS3** (not EVE-NG Pro/Community, not CML) — plan is to deploy it in the _same_ VPC/project as `promgraf-vm` to reuse the Cloud NAT and get private-IP reachability for SNMP/blackbox scraping. Not yet started.
- Old EVE-NG VM (`eveng`, separate `eveng-vpc`) is `TERMINATED` but not destroyed — fate undecided, low priority.


----

```
lance@promgraf-vm:/opt/promgraf$ find / -iname "docker-compose*.y*ml" 2>/dev/null

/opt/promgraf/docker-compose.yaml

/opt/promgraf/docker-compose.override.yml

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ whoami

hostname

lance

promgraf-vm.us-central1-a.c.eve-ng-lab-lanc3.internal

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ docker compose ls

permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.51/containers/json?filters=%7B%22label%22%3A%7B%22com.docker.compose.config-hash%22%3Atrue%2C%22com.docker.compose.project%22%3Atrue%7D%7D": dial unix /var/run/docker.sock: connect: permission denied

lance@promgraf-vm:/opt/promgraf$ sudo docker compose ls

NAME                STATUS              CONFIG FILES

promgraf            running(5)          /opt/promgraf/docker-compose.yaml,/opt/promgraf/docker-compose.override.yml

lance@promgraf-vm:/opt/promgraf$ sudo usermod -aG docker $USER

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker ps

CONTAINER ID   IMAGE                       COMMAND                  CREATED             STATUS             PORTS                                         NAMES

1f7833bc59d9   postgres:18                 "docker-entrypoint.s…"   About an hour ago   Up About an hour   0.0.0.0:5488->5432/tcp, [::]:5488->5432/tcp   postgres

80b3942b18a1   grafana/loki:3.7.1          "/usr/bin/loki --val…"   About an hour ago   Up About an hour   0.0.0.0:3100->3100/tcp, [::]:3100->3100/tcp   loki

aa06127d1bc0   prom/prometheus:v3.11.2     "/bin/prometheus --w…"   About an hour ago   Up About an hour   0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp   prometheus

5fa6af9d59d3   prom/alertmanager:v0.33.0   "/bin/alertmanager -…"   About an hour ago   Up About an hour   0.0.0.0:9093->9093/tcp, [::]:9093->9093/tcp   alertmanager

717b0436e265   grafana/grafana:12.3.1      "/run.sh"                About an hour ago   Up About an hour   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp   grafana

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ docker inspect <loki_container_id_or_name> | grep -A20 '"Mounts"'

-bash: syntax error near unexpected token `|'

lance@promgraf-vm:/opt/promgraf$

```

------

hooking up alertmanager with n8n
test drive

```
lance@promgraf-vm:/opt/promgraf$ curl -X POST --resolve n8n.lanc3.com:443:100.122.88.54 https://n8n.lanc3.com/webhook/alertmanager -H "Content-Type: application/json" -d '{"test":"hello"}'

{"message":"Workflow was started"}lance@promgraf-vm:/opt/promgraf$
```


```
lance@promgraf-vm:/opt/promgraf$ cat /opt/promgraf/alertmanager/alertmanager.yml

# SMTP/email alerting intentionally removed - the smtp service is excluded

# from this stack (dead upstream image), and alerting routes to n8n instead.

route:

  receiver: "n8n-webhook"

  group_by: [alertname]

  group_wait: 10s

  group_interval: 30s

  repeat_interval: 1h

  

receivers:

  - name: "n8n-webhook"

    webhook_configs:

      - url: "https://n8n.lanc3.com/webhook/alertmanager"

        send_resolved: true

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo nano /opt/promgraf/docker-compose.override.yml

lance@promgraf-vm:/opt/promgraf$ cd /opt/promgraf

sudo docker compose up -d alertmanager

[+] Running 1/1

 ✔ Container alertmanager  Started                                                                                                                   1.6s 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ sudo docker compose logs alertmanager --tail 30

alertmanager  | time=2026-09-12T04:14:47.536Z level=INFO source=main.go:196 msg="Starting Alertmanager" version="(version=0.33.0, branch=HEAD, revision=5d3ceb55bf3775ea152dcdf3803bbbb2b4afed54)"

alertmanager  | time=2026-09-12T04:14:47.536Z level=INFO source=main.go:199 msg="Build context" build_context="(go=go1.26.4, platform=linux/amd64, user=root@7a8b562c66e8, date=20260612-15:35:56, tags=netgo)"

alertmanager  | time=2026-09-12T04:14:47.537Z level=INFO source=cluster.go:192 msg="setting advertise address explicitly" component=cluster addr=172.18.0.2 port=9094

alertmanager  | time=2026-09-12T04:14:47.542Z level=INFO source=cluster.go:682 msg="Waiting for gossip to settle..." component=cluster interval=2s

alertmanager  | time=2026-09-12T04:14:47.615Z level=INFO source=tls_config.go:354 msg="Listening on" address=[::]:9093

alertmanager  | time=2026-09-12T04:14:47.615Z level=INFO source=tls_config.go:357 msg="TLS is disabled." http2=false address=[::]:9093

alertmanager  | time=2026-09-12T04:14:49.543Z level=INFO source=cluster.go:707 msg="gossip not settled" component=cluster polls=0 before=0 now=1 elapsed=2.000589477s

alertmanager  | time=2026-09-12T04:14:57.547Z level=INFO source=cluster.go:699 msg="gossip settled; proceeding" component=cluster elapsed=10.004283811s

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ 

lance@promgraf-vm:/opt/promgraf$ curl -XPOST http://localhost:9093/api/v2/alerts -H "Content-Type: application/json" -d '[{

  "labels": {"alertname": "TestAlert", "severity": "warning"},

  "annotations": {"summary": "manual test alert"},

  "startsAt": "'"$(date -u +%Y-%m-%dT%H:%M:%S.000Z)"'"

}]'

lance@promgraf-vm:/opt/promgraf$
```