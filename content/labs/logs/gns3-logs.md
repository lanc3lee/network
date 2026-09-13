


```
lance@LANC3 gns3 % nano terraform.tfvars

lance@LANC3 gns3 % terraform init

Initializing provider plugins found in the configuration...

- Finding hashicorp/google versions matching "~> 5.0"...

- Installing hashicorp/google v5.45.2...

- Installed hashicorp/google v5.45.2 (signed by HashiCorp)

  

Initializing the backend...

  

  

Terraform has created a lock file .terraform.lock.hcl to record the provider

selections it made above. Include this file in your version control repository

so that Terraform can guarantee to make the same selections by default when

you run "terraform init" in the future.

  

Terraform has been successfully initialized!

  

You may now begin working with Terraform. Try running "terraform plan" to see

any changes that are required for your infrastructure. All Terraform commands

should now work.

  

If you ever set or change modules or backend configuration for Terraform,

rerun this command to reinitialize your working directory. If you forget, other

commands will detect it and remind you to do so if necessary.

lance@LANC3 gns3 % terraform plan

data.google_compute_network.default: Reading...

data.google_compute_network.default: Read complete after 0s [id=projects/eve-ng-lab-lanc3/global/networks/default]

  

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following

symbols:

  + create

  

Terraform will perform the following actions:

  

  # google_compute_firewall.allow_promgraf_to_gns3_topology will be created

  + resource "google_compute_firewall" "allow_promgraf_to_gns3_topology" {

      + creation_timestamp = (known after apply)

      + destination_ranges = [

          + "10.10.0.0/24",

        ]

      + direction          = (known after apply)

      + enable_logging     = (known after apply)

      + id                 = (known after apply)

      + name               = "allow-promgraf-to-gns3-topology"

      + network            = "default"

      + priority           = 1000

      + project            = "eve-ng-lab-lanc3"

      + self_link          = (known after apply)

      + source_tags        = [

          + "promgraf",

        ]

      + target_tags        = [

          + "gns3",

        ]

  

      + allow {

          + ports    = [

              + "161",

            ]

          + protocol = "udp"

        }

      + allow {

          + ports    = [

              + "9115",

              + "80",

              + "443",

            ]

          + protocol = "tcp"

        }

    }

  

  # google_compute_instance.gns3 will be created

  + resource "google_compute_instance" "gns3" {

      + can_ip_forward          = true

      + cpu_platform            = (known after apply)

      + current_status          = (known after apply)

      + deletion_protection     = false

      + effective_labels        = (known after apply)

      + guest_accelerator       = (known after apply)

      + id                      = (known after apply)

      + instance_id             = (known after apply)

      + label_fingerprint       = (known after apply)

      + machine_type            = "n1-standard-4"

      + metadata                = {

          + "tailscale-authkey" = (sensitive value)

        }

      + metadata_fingerprint    = (known after apply)

      + metadata_startup_script = <<-EOT

            #!/bin/bash

            set -e

            # --- Tailscale: private GUI access, no public IP, no open firewall rule ---

            TAILSCALE_AUTHKEY=$(curl -s -H "Metadata-Flavor: Google" \

              "http://metadata.google.internal/computeMetadata/v1/instance/attributes/tailscale-authkey")

            if ! command -v tailscale &> /dev/null; then

              curl -fsSL https://tailscale.com/install.sh | sh

            fi

            if [ -n "$TAILSCALE_AUTHKEY" ] && ! tailscale status &> /dev/null; then

              tailscale up --authkey="$TAILSCALE_AUTHKEY" --hostname=gns3-vm --ssh=false

            fi

            # --- Sanity-check nested virtualization actually landed ---

            if [ "$(grep -cw vmx /proc/cpuinfo)" -eq 0 ]; then

              logger -p user.warning -t gns3-startup "WARNING: no vmx flag - nested virtualization is NOT active. Check advanced_machine_features + min_cpu_platform in Terraform."

            fi

            # --- IP forwarding at the OS level (can_ip_forward only permits it at the

            # GCP network layer - the kernel still has to actually do it) ---

            sysctl -w net.ipv4.ip_forward=1

            echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-gns3-forwarding.conf

            apt-get update -y

            # --- GNS3 server via the official remote-install script ---

            # https://docs.gns3.com/docs/getting-started/installation/remote-server/

            cd /tmp

            curl -s https://raw.githubusercontent.com/GNS3/gns3-server/master/scripts/remote-install.sh > gns3-remote-install.sh

            bash gns3-remote-install.sh --with-welcome

            logger -t gns3-startup "GNS3 install script finished. A reboot is recommended before first use - do it manually (sudo reboot) rather than from within this script."

        EOT

      + min_cpu_platform        = "Intel Haswell"

      + name                    = "gns3-vm"

      + project                 = "eve-ng-lab-lanc3"

      + self_link               = (known after apply)

      + tags                    = [

          + "gns3",

          + "iap-ssh",

        ]

      + tags_fingerprint        = (known after apply)

      + terraform_labels        = (known after apply)

      + zone                    = "us-central1-a"

  

      + advanced_machine_features {

          + enable_nested_virtualization = true

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

              + size                   = 60

              + type                   = (known after apply)

            }

        }

  

      + confidential_instance_config (known after apply)

  

      + network_interface {

          + internal_ipv6_prefix_length = (known after apply)

          + ipv6_access_type            = (known after apply)

          + ipv6_address                = (known after apply)

          + name                        = (known after apply)

          + network                     = "https://www.googleapis.com/compute/v1/projects/eve-ng-lab-lanc3/global/networks/default"

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

  

  # google_compute_route.gns3_topology will be created

  + resource "google_compute_route" "gns3_topology" {

      + dest_range             = "10.10.0.0/24"

      + id                     = (known after apply)

      + name                   = "gns3-topology-route"

      + network                = "default"

      + next_hop_instance      = (known after apply)

      + next_hop_instance_zone = (known after apply)

      + next_hop_ip            = (known after apply)

      + next_hop_network       = (known after apply)

      + priority               = 1000

      + project                = "eve-ng-lab-lanc3"

      + self_link              = (known after apply)

    }

  

Plan: 3 to add, 0 to change, 0 to destroy.

  

Changes to Outputs:

  + gns3_topology_cidr = "10.10.0.0/24"

  + instance_name      = "gns3-vm"

  + ssh_command        = "gcloud compute ssh gns3-vm --zone=us-central1-a --tunnel-through-iap"

  

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

  

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run

"terraform apply" now.

lance@LANC3 gns3 % 

lance@LANC3 gns3 % terraform apply

data.google_compute_network.default: Reading...

data.google_compute_network.default: Read complete after 1s [id=projects/eve-ng-lab-lanc3/global/networks/default]

  

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following

symbols:

  + create

  

Terraform will perform the following actions:

  

  # google_compute_firewall.allow_promgraf_to_gns3_topology will be created

  + resource "google_compute_firewall" "allow_promgraf_to_gns3_topology" {

      + creation_timestamp = (known after apply)

      + destination_ranges = [

          + "10.10.0.0/24",

        ]

      + direction          = (known after apply)

      + enable_logging     = (known after apply)

      + id                 = (known after apply)

      + name               = "allow-promgraf-to-gns3-topology"

      + network            = "default"

      + priority           = 1000

      + project            = "eve-ng-lab-lanc3"

      + self_link          = (known after apply)

      + source_tags        = [

          + "promgraf",

        ]

      + target_tags        = [

          + "gns3",

        ]

  

      + allow {

          + ports    = [

              + "161",

            ]

          + protocol = "udp"

        }

      + allow {

          + ports    = [

              + "9115",

              + "80",

              + "443",

            ]

          + protocol = "tcp"

        }

    }

  

  # google_compute_instance.gns3 will be created

  + resource "google_compute_instance" "gns3" {

      + can_ip_forward          = true

      + cpu_platform            = (known after apply)

      + current_status          = (known after apply)

      + deletion_protection     = false

      + effective_labels        = (known after apply)

      + guest_accelerator       = (known after apply)

      + id                      = (known after apply)

      + instance_id             = (known after apply)

      + label_fingerprint       = (known after apply)

      + machine_type            = "n1-standard-4"

      + metadata                = {

          + "tailscale-authkey" = (sensitive value)

        }

      + metadata_fingerprint    = (known after apply)

      + metadata_startup_script = <<-EOT

            #!/bin/bash

            set -e

            # --- Tailscale: private GUI access, no public IP, no open firewall rule ---

            TAILSCALE_AUTHKEY=$(curl -s -H "Metadata-Flavor: Google" \

              "http://metadata.google.internal/computeMetadata/v1/instance/attributes/tailscale-authkey")

            if ! command -v tailscale &> /dev/null; then

              curl -fsSL https://tailscale.com/install.sh | sh

            fi

            if [ -n "$TAILSCALE_AUTHKEY" ] && ! tailscale status &> /dev/null; then

              tailscale up --authkey="$TAILSCALE_AUTHKEY" --hostname=gns3-vm --ssh=false

            fi

            # --- Sanity-check nested virtualization actually landed ---

            if [ "$(grep -cw vmx /proc/cpuinfo)" -eq 0 ]; then

              logger -p user.warning -t gns3-startup "WARNING: no vmx flag - nested virtualization is NOT active. Check advanced_machine_features + min_cpu_platform in Terraform."

            fi

            # --- IP forwarding at the OS level (can_ip_forward only permits it at the

            # GCP network layer - the kernel still has to actually do it) ---

            sysctl -w net.ipv4.ip_forward=1

            echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-gns3-forwarding.conf

            apt-get update -y

            # --- GNS3 server via the official remote-install script ---

            # https://docs.gns3.com/docs/getting-started/installation/remote-server/

            cd /tmp

            curl -s https://raw.githubusercontent.com/GNS3/gns3-server/master/scripts/remote-install.sh > gns3-remote-install.sh

            bash gns3-remote-install.sh --with-welcome

            logger -t gns3-startup "GNS3 install script finished. A reboot is recommended before first use - do it manually (sudo reboot) rather than from within this script."

        EOT

      + min_cpu_platform        = "Intel Haswell"

      + name                    = "gns3-vm"

      + project                 = "eve-ng-lab-lanc3"

      + self_link               = (known after apply)

      + tags                    = [

          + "gns3",

          + "iap-ssh",

        ]

      + tags_fingerprint        = (known after apply)

      + terraform_labels        = (known after apply)

      + zone                    = "us-central1-a"

  

      + advanced_machine_features {

          + enable_nested_virtualization = true

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

              + size                   = 60

              + type                   = (known after apply)

            }

        }

  

      + confidential_instance_config (known after apply)

  

      + network_interface {

          + internal_ipv6_prefix_length = (known after apply)

          + ipv6_access_type            = (known after apply)

          + ipv6_address                = (known after apply)

          + name                        = (known after apply)

          + network                     = "https://www.googleapis.com/compute/v1/projects/eve-ng-lab-lanc3/global/networks/default"

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

  

  # google_compute_route.gns3_topology will be created

  + resource "google_compute_route" "gns3_topology" {

      + dest_range             = "10.10.0.0/24"

      + id                     = (known after apply)

      + name                   = "gns3-topology-route"

      + network                = "default"

      + next_hop_instance      = (known after apply)

      + next_hop_instance_zone = (known after apply)

      + next_hop_ip            = (known after apply)

      + next_hop_network       = (known after apply)

      + priority               = 1000

      + project                = "eve-ng-lab-lanc3"

      + self_link              = (known after apply)

    }

  

Plan: 3 to add, 0 to change, 0 to destroy.

  

Changes to Outputs:

  + gns3_topology_cidr = "10.10.0.0/24"

  + instance_name      = "gns3-vm"

  + ssh_command        = "gcloud compute ssh gns3-vm --zone=us-central1-a --tunnel-through-iap"

  

Do you want to perform these actions?

  Terraform will perform the actions described above.

  Only 'yes' will be accepted to approve.

  

  Enter a value: yes

  

google_compute_firewall.allow_promgraf_to_gns3_topology: Creating...

google_compute_instance.gns3: Creating...

google_compute_firewall.allow_promgraf_to_gns3_topology: Still creating... [00m10s elapsed]

google_compute_instance.gns3: Still creating... [00m10s elapsed]

google_compute_firewall.allow_promgraf_to_gns3_topology: Creation complete after 13s [id=projects/eve-ng-lab-lanc3/global/firewalls/allow-promgraf-to-gns3-topology]

google_compute_instance.gns3: Creation complete after 20s [id=projects/eve-ng-lab-lanc3/zones/us-central1-a/instances/gns3-vm]

google_compute_route.gns3_topology: Creating...

google_compute_route.gns3_topology: Still creating... [00m10s elapsed]

google_compute_route.gns3_topology: Creation complete after 15s [id=projects/eve-ng-lab-lanc3/global/routes/gns3-topology-route]

  

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

  

Outputs:

  

gns3_topology_cidr = "10.10.0.0/24"

instance_name = "gns3-vm"

ssh_command = "gcloud compute ssh gns3-vm --zone=us-central1-a --tunnel-through-iap"

lance@LANC3 gns3 %
```

ssh in via IAP


```
lance@LANC3 gns3 % gcloud compute ssh gns3-vm --zone=us-central1-a --tunnel-through-iap

WARNING: 

  

To increase the performance of the tunnel, consider installing NumPy. For instructions,

please see https://cloud.google.com/iap/docs/using-tcp-forwarding#increasing_the_tcp_upload_bandwidth

  

Warning: Permanently added 'compute.6408308963494370175' (ED25519) to the list of known hosts.

Welcome to Ubuntu 24.04.5 LTS (GNU/Linux 7.0.0-1011-gcp x86_64)

  

 * Documentation:  https://help.ubuntu.com

 * Management:     https://landscape.canonical.com

 * Support:        https://ubuntu.com/pro

  

 System information as of Sat Sep 12 10:16:18 UTC 2026

  

  System load:  1.12              Processes:             155

  Usage of /:   6.8% of 57.08GB   Users logged in:       0

  Memory usage: 4%                IPv4 address for ens4: 10.128.0.5

  Swap usage:   0%

  

Expanded Security Maintenance for Applications is not enabled.

  

0 updates can be applied immediately.

  

Enable ESM Apps to receive additional future security updates.

See https://ubuntu.com/esm or run: sudo pro status

  

  

*** System restart required ***

  

The programs included with the Ubuntu system are free software;

the exact distribution terms for each program are described in the

individual files in /usr/share/doc/*/copyright.

  

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by

applicable law.

  

lance@gns3-vm:~$
```


**Confirm nested virtualization actually landed:**

Should return a non-zero number (one per vCPU). If it's `0`, something's off with `min_cpu_platform`/`advanced_machine_features` and GNS3's QEMU-based nodes won't run — worth stopping and fixing before going further, since everything downstream depends on this.

```bash
grep -cw vmx /proc/cpuinfo
```


**Check the GNS3 install actually finished:**

bash

```bash
sudo journalctl -u google-startup-scripts.service | tail -50
```

Look for the `"GNS3 install script finished..."` line from the startup script's final `logger` call — that confirms `remote-install.sh` ran to completion rather than erroring out partway (the script installs a fair number of packages, so this is worth confirming before assuming it's ready).

**Reboot once, as the install script recommends:**

bash

```bash
sudo reboot
```

```
lance@gns3-vm:~$ sudo reboot

Broadcast message from root@gns3-vm.us-central1-a.c.eve-ng-lab-lanc3.internal on pts/1 (Sat 2026-09-12 10:24:15 UTC):

The system will reboot now!
```

Give it a minute or two, then SSH back in.

**After reboot, confirm the GNS3 service is actually up:**

bash

```bash
sudo systemctl status gns3
```

(or `sudo systemctl status gns3server` — the exact unit name depends on what `remote-install.sh` created; if `gns3` doesn't resolve, `systemctl list-units | grep -i gns3` will show you the real name)

```
lance@gns3-vm:~$ sudo systemctl status gns3

gns3.service - GNS3 server

     Loaded: loaded (/usr/lib/systemd/system/gns3.service; enabled; preset: enabled)

     Active: active (running) since Sat 2026-09-12 10:25:14 UTC; 1min 42s ago

   Main PID: 700 (gns3server)

      Tasks: 1 (limit: 16307)

     Memory: 63.7M (peak: 67.7M)

        CPU: 1.298s

     CGroup: /system.slice/gns3.service

             └─700 /usr/share/gns3/gns3-server/bin/python /usr/bin/gns3server --log /var/log/gns3/gns3.log

  

Sep 12 10:25:13 gns3-vm systemd[1]: Starting gns3.service - GNS3 server...

Sep 12 10:25:14 gns3-vm systemd[1]: Started gns3.service - GNS3 server.

lance@gns3-vm:~$
```

------

Installing GNS3 client on your laptop or desktop 
Go to github.com/GNS3/gns3-gui/releases and grab the latest build for your OS. 
This skips needing a GNS3.com account, which the official download page normally asks for.


-----

```
lance@gns3-vm:~$ gns3server --version

2.2.61

lance@gns3-vm:~$
```