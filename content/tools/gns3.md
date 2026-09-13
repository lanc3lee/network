---
title: "Running GNS3 on GCP the Reproducible, Private Way (September 2026)"
date: 2026-09-12
tags: [gns3, gcp, terraform, network-emulation, tailscale]
description: "A Terraform-managed, Tailscale-only GNS3 build on GCP that shares a VPC with existing infrastructure — provisioning, the nested-virtualization setup, and the gotchas that only show up once GNS3 stops being a solo lab."
---
**GNS3** (Graphical Network Simulator-3) is ==a powerful software tool that lets network engineers build, design, and test virtual networks using real router and switch software without buying expensive hardware==. 

While running it on a personal computer can quickly slow down the machine, hosting GNS3 on **Google Cloud** gives engineers access to massive, flexible computing power to run huge network labs smoothly. This cloud setup allows for easy teamwork from anywhere, ensures heavy network simulations do not crash your local computer, and lets you pay only for the exact amount of server time you use.

GNS3 needs a server component and a GUI client talking to it, and GNS3's own docs recommend running the server as a dedicated VM. GCP is a fine place to put that VM — but the moment you want it reproducible, private, and living alongside other infrastructure rather than standing alone for a weekend, a few things need more care than a quick console click-through gives you. 

This is that build, end to end, as of September 2026.

## Design goals

Three constraints shaped this setup:

1. **Reproducible.** Provisioned via Terraform, not console clicks — so it can be torn down and rebuilt without re-deriving every setting from memory.
2. **Private.** No public IP, no firewall rule open to the internet. Access only over a private mesh network (Tailscale).
3. **Integrated, not standalone.** The GNS3 VM shares a VPC with existing infrastructure — in my case, a Prometheus/Grafana monitoring stack — so that Prometheus can eventually scrape the emulated network devices for latency and health data. That last goal is what introduces the trickiest gotcha below.

## The GCP nested-virtualization constraint

GNS3's router/firewall nodes run under QEMU, which needs nested virtualization — a VM running its own virtual machines. GCP restricts this to specific machine families: **N1, N2, or N4 on Intel only.** No AMD, no Arm, and on Intel you're excluded from the E2 general-purpose, memory-optimized, and A3 accelerator-optimized families. This isn't a default that changes often, but it does mean your instance-type choice is constrained before anything else.

In Terraform, enabling it is a single block on the instance resource:

```hcl
resource "google_compute_instance" "gns3" {
  name             = "gns3-vm"
  machine_type     = "n1-standard-4"
  min_cpu_platform = "Intel Haswell"

  advanced_machine_features {
    enable_nested_virtualization = true
  }
  # ...
}
```

`min_cpu_platform` matters too — nested virtualization needs a Haswell-generation host or later, and leaving it unset risks landing on older hardware in some zones.

Verify it actually took, once the instance is up:

```bash
grep -cw vmx /proc/cpuinfo
```

Any non-zero result confirms nested virtualization is active. Zero means something didn't land — check the Terraform plan output for the `advanced_machine_features` block before going further.

## No public IP: access over Tailscale

The GNS3 GUI talks to the server over a handful of TCP ports (`3080` for the API, plus a range for node consoles). The common pattern is opening those ports to `0.0.0.0/0` on a tagged firewall rule. That's fine for a lab you'll tear down in an afternoon; it's a liability for anything longer-lived.

The alternative: skip the public IP and firewall rule entirely, and join the instance to a private mesh network instead.

```hcl
resource "google_compute_instance" "gns3" {
  # ...
  network_interface {
    network = data.google_compute_network.default.self_link
    # deliberately no access_config block -> no external IP
  }
}
```

With [Tailscale](https://tailscale.com) installed via the startup script and joined using a tagged, reusable auth key, the GNS3 desktop client connects to the VM's Tailscale IP (a `100.x.x.x` address) instead of a public one. Tailscale's own NAT traversal handles the connection — no inbound firewall rule needed at all.

One easy-to-miss detail: the GNS3 *server process itself* needs to bind to `0.0.0.0` on its listening port, not just the VPC's internal IP. Check with:

```bash
sudo ss -tlnp | grep 3080
```

If it shows a specific internal IP instead of `0.0.0.0`, connections over the Tailscale interface will fail with a plain "connection refused" — which reads like a network problem but is actually a server-config one. It's set in `/etc/gns3/gns3_server.conf` under `[Server] host = 0.0.0.0`; if the config already says that but the running process disagrees, a `systemctl restart` (rather than reload) usually resolves it, since GNS3 reads this at startup and doesn't hot-reload.

## Sharing a VPC and its NAT

If you already have other infrastructure running in a GCP project — with a Cloud NAT already provisioned — GNS3 doesn't need its own. Point a `data` source at the existing network rather than declaring a new one:

```hcl
data "google_compute_network" "default" {
  name = "default"
}
```

Reference it in the instance's `network_interface` block, and the GNS3 VM inherits the existing NAT for outbound internet automatically — Cloud NAT is scoped at the network/region level in GCP, not per-instance, so there's nothing extra to provision. 
Keep this as its own Terraform root module with its own state, though: referencing another module's network as read-only data means neither module's `apply` can accidentally modify the other's resources.

## Installing the server

Once the instance is up, the official install script does the work:

```bash
cd /tmp
curl -s https://raw.githubusercontent.com/GNS3/gns3-server/master/scripts/remote-install.sh > gns3-remote-install.sh
sudo bash gns3-remote-install.sh --with-welcome
```

This installs from GNS3's stable branch — as of this build, that's the 2.2.x line. A reboot afterward is recommended before first use. Confirm the service is actually running:

```bash
sudo systemctl status gns3
```

## Making emulated devices reachable

This is the part that only shows up once GNS3 needs to talk to something *else* on the network, and it's easy to miss in a single-user lab.

If your emulated devices (virtual routers, firewalls) sit directly on the shared VPC — rather than behind their own bridge or NAT hidden inside the GNS3 VM — GCP will silently drop any packet whose source or destination address doesn't match the receiving instance's own assigned IP. Emulated-device traffic looks exactly like that from the VPC's point of view, since those addresses were never actually assigned to a real GCP instance. Three things fix it:

**1. Allow IP forwarding at the GCP layer:**
```hcl
resource "google_compute_instance" "gns3" {
  # ...
  can_ip_forward = true
}
```

**2. Allow it at the OS/kernel layer too** — GCP's setting only permits it at the network layer; the guest kernel still needs to actually do the forwarding:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-gns3-forwarding.conf
```

**3. Tell the VPC where the emulated-device range lives**, with a route pointing at the GNS3 instance as next hop, plus a firewall rule scoped to whichever real instance actually needs to reach it:

```hcl
resource "google_compute_route" "gns3_topology" {
  name              = "gns3-topology-route"
  network           = data.google_compute_network.default.name
  dest_range        = "10.10.0.0/24"
  next_hop_instance = google_compute_instance.gns3.self_link
  priority          = 1000
}

resource "google_compute_firewall" "allow_scraper_to_gns3_topology" {
  name    = "allow-scraper-to-gns3-topology"
  network = data.google_compute_network.default.name

  allow {
    protocol = "udp"
    ports    = ["161"] # SNMP
  }
  allow {
    protocol = "tcp"
    ports    = ["9115", "80", "443"] # blackbox exporter, device web UIs
  }

  source_tags        = ["scraper-host"]
  target_tags        = ["gns3"]
  destination_ranges = ["10.10.0.0/24"]
}
```

Pick your emulated-device CIDR outside the VPC's auto-allocated ranges — GCP's default auto-mode network carves per-region `/20` blocks out of `10.128.0.0/9`, so something like `10.10.0.0/24` avoids collision.

Inside the GNS3 topology itself, this means using a **Cloud node bridged to the VM's primary interface** for anything that needs a real presence on the VPC, with those devices addressed from the reserved range above.

## Pin your client version to your server version

GNS3 currently has an active 3.x line (alpha, as of this writing) alongside its stable 2.2.x branch, and the default install script tracks stable. If your desktop GUI happens to be a 3.x build, it speaks the v3 API against a server that only exposes v2 — the symptom is a `404` on `/v3/version`, which has nothing to do with networking and everything to do with mismatched versions. Check what's actually running before assuming the newest client download is correct:

```bash
gns3server --version
```

Match your GUI client to that exact line. 


## Summary

A GNS3 server that's meant to last longer than an afternoon benefits from three things a quick lab setup doesn't need: 
1) Terraform for reproducibility, 
2) private mesh network instead of an open firewall rule, and — if it's going to share a VPC with anything else 
3) explicit IP forwarding and routing so that other traffic can actually reach the emulated topology. 

None of it is complicated once you know it's needed; the IP-forwarding requirement in particular is the one most likely to cost an afternoon of "why can't I ping my routers" if you don't see it coming.


-------


**Finding GNS3's actual QEMU images directory**

Don't assume the commonly-cited default path (`~/GNS3/images/QEMU/`) is correct for your install — it depends on which user account runs the `gns3` service and how it was packaged, and guessing wrong wastes time chasing "No such file or directory" errors. The reliable way to find it is to ask the server itself, since it ships with a set of pre-created blank disk images (`empty8G.qcow2`, `empty20G.qcow2`, etc.) that already live in the real directory. Search the filesystem for one of these known filenames:

```
sudo find / -name "empty8G.qcow2" 2>/dev/null
```

This will likely return two hits — one inside the installed Python package itself (e.g. `.../site-packages/gns3server/disks/empty8G.qcow2`), which is just the source template GNS3 ships with, and one in the actual runtime images directory (e.g. `/opt/gns3/images/QEMU/empty8G.qcow2`), which is where you should drop any new images you download. 

If the service runs as a dedicated `gns3` user (check with `getent passwd gns3`), download new images with `sudo -u gns3 wget -P <path> <url>` so they're owned correctly from the start, rather than fixing permissions after the fact.

--------

is: **`/opt/gns3/images/QEMU/`** (not `~/GNS3/images/QEMU/` — my earlier guess was off; the real path drops the extra `GNS3` folder).

Now download the ISO straight into it:

```
sudo -u gns3 wget -P /opt/gns3/images/QEMU/ https://github.com/vyos/vyos-nightly-build/releases/download/2026.09.09-0029-rolling/vyos-2026.09.09-0029-rolling-generic-amd64.iso
```

Running it via `sudo -u gns3` writes the file as the `gns3` user directly, so you won't hit ownership/permission issues later when the service tries to read it.

Once that finishes, verify it landed correctly:

```
ls -la /opt/gns3/images/QEMU/ | grep vyos
```

Then head back into the GNS3 GUI — **Preferences → QEMU VMs → New** — and it should show up as a selectable ISO/image in that directory.


-----

```
vyos login: vyos

Password: 

Welcome to VyOS!

  

   ┌── ┐

   . VyOS 2026.09.09-0029-rolling

   └ ──┘  rolling

  

 * Documentation:  https://docs.vyos.io/en/latest

 * Project news:   https://blog.vyos.io

 * Bug reports:    https://vyos.dev

  

You can change this banner using "set system login banner post-login" command.

  

VyOS is a free software distribution that includes multiple components,

you can check individual component licenses under /usr/share/doc/*/copyright

  

---

WARNING: This VyOS system is not a stable long-term support version and

         is not intended for production use.

  

vyos@vyos:~$ lsblk

NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS

loop0   7:0    0 570.4M  1 loop /usr/lib/live/mount/rootfs/filesystem.squashfs

sr0    11:0    1   653M  1 rom  /usr/lib/live/mount/medium

vda   252:0    0     8G  0 disk 

vyos@vyos:~$ install image

Welcome to VyOS installation!

This command will install VyOS to your permanent storage.

Would you like to continue? [y/N] y

What would you like to name this image? (Default: 2026.09.09-0029-rolling) 

Please enter a password for the "vyos" user: 

  

WARNING: Default password used. Consider changing it on next login.

  

Please confirm password for the "vyos" user: 

What console should be used by default? (K: KVM, S: Serial)? (Default: S) 

Probing disks

1 disk(s) found

The following disks were found:

Drive: /dev/vda (8.0 GB)

Which one should be used for installation? (Default: /dev/vda) 

Installation will delete all data on the drive. Continue? [y/N] y

Searching for data from previous installations

No previous installation found

Would you like to use all the free space on the drive? [Y/n] 

Creating partition table...

The following config files are available for boot:

1: /opt/vyatta/etc/config/config.boot

2: /opt/vyatta/etc/config.boot.default

Which file would you like as boot config? (Default: 1) 

Creating temporary directories

Mounting new partitions

Creating a configuration file

Copying system image files

Installing GRUB configuration files

Installing GRUB to the drive

Cleaning up

Unmounting target filesystems

Removing temporary files

The image installed successfully; please reboot now.

vyos@vyos:~$ reboot

```

**While it's rebooting, go remove the ISO from the node config** so it doesn't boot back into the installer:

- Right-click `vyOS-1` → **Configure** → **CD/DVD** tab
- Clear out the `vyos-2026.09.09-0029-rolling-generic-amd64.iso` field
- Click **OK**


------

```
lance@gns3-vm:~$ sudo docker import /opt/gns3/images/cEOS64-lab-4.33.9M.tar.xz ceos-image:4.33.9M

sha256:c2a06f093b1d68ed3e35e1e79bec7290a52abdee17abc871f777eed80d5748ea

lance@gns3-vm:~$ 

lance@gns3-vm:~$ sudo docker images | grep ceos

ceos-image:4.33.9M    c2a06f093b1d       3.75GB            1GB        

lance@gns3-vm:~$

```