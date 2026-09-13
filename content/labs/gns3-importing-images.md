---
title: Importing Images into GNS3 — A Field Guide
description: Practical, tested steps for getting QEMU-based and Docker-based appliances (like VyOS and Arista cEOS) working in a remote GNS3 lab.
tags: [gns3, networking, homelab]
---

# Importing Images into GNS3 — A Field Guide

GNS3's documentation covers the happy path for importing images, but a few gotchas only show up once you actually try it — especially when your GNS3 server runs remotely (e.g. on a cloud VM) rather than on your local machine. This guide walks through both major import paths: **QEMU-based appliances** (like VyOS) and **Docker-based appliances** (like Arista cEOS), including the specific failures we hit and how to fix them.

## Before you start: find where GNS3 actually stores images

Don't trust commonly-cited default paths like `~/GNS3/images/QEMU/` — the real location depends on your install and which user account runs the `gns3` service. Guessing wrong wastes time chasing "No such file or directory" errors.

The reliable way to find it: GNS3 ships with a set of pre-created blank disk images (`empty8G.qcow2`, `empty20G.qcow2`, etc.). Search for one of these known filenames on the server:

```bash
sudo find / -name "empty8G.qcow2" 2>/dev/null
```

This typically returns two hits — one inside the installed Python package itself (the source template GNS3 ships with), and one in the actual runtime images directory. The second one is where you should drop new images. For example, on our install this was `/opt/gns3/images/QEMU/`.

If the `gns3` service runs as a dedicated system user (check with `getent passwd gns3`), download or copy new images as that user so ownership is correct from the start:

```bash
sudo -u gns3 wget -P /opt/gns3/images/QEMU/ <image-url>
```

## Importing a QEMU-based appliance (example: VyOS)

QEMU appliances need a bit more manual setup than Docker ones, since there's usually no bundled "appliance" file — you're building the template yourself.

1. **Download the image.** For free/community images (like VyOS's rolling release), you'll often only get an **ISO**, not a pre-built `.qcow2` — pre-built disk images are frequently a paid/subscriber perk. Plan to install from ISO instead.
2. **Create a new QEMU VM template**: *Edit → Preferences → QEMU VMs → New*. Pick a standard `qemu-system-x86_64` binary (avoid `-spice`, `-i386`, or `-microvm` variants unless you specifically need them) and set adequate RAM — the wizard's default (256 MB) is too low for most modern network OS images; 1024 MB is a safer starting point.
3. **Set up storage correctly — this is the step most likely to trip you up:**
   - **HDA (hard disk)**: point this at a **blank** disk image (e.g. `empty8G.qcow2`), *not* the ISO.
   - **CD/DVD**: point this at the **ISO** you downloaded.
   - These are easy to mix up, especially since the setup wizard's disk-selection step doesn't always make the distinction obvious. If you install and get an error like *"No suitable disk was found,"* check this first.
4. **Boot the node and run the vendor's install command** (e.g. VyOS's `install image`), pointing it at the disk you attached — not the CD-ROM.
5. **After installation completes, remove the ISO** from the CD/DVD field so future boots load the installed system instead of relaunching the installer.

### A gotcha worth knowing: stale per-node disk files

GNS3 copies your selected HDA image into a per-node working directory the first time a node boots. If you initially select the wrong disk (e.g. accidentally leave the ISO in HDA) and then correct it in the GUI afterward, GNS3 doesn't always regenerate that per-node copy — it can keep silently using the stale first-boot version, even though the config screen shows the corrected filename.

**Symptom:** installer reports the wrong disk size, or fails to find a suitable disk at all, even though the GUI looks correctly configured.

**Fix:** stop the node, find its per-node disk file, and delete it so GNS3 regenerates it from the currently-selected source:

```bash
sudo find / -iname "hda_disk*" 2>/dev/null
```

Delete the stale file, then start the node again — GNS3 will recreate it fresh from whatever image is currently selected in HDA.

### Permissions note: running QEMU/ubridge tools without sudo

If you need to run `ubridge` or similar helper binaries directly (e.g. for troubleshooting), check the binary's actual owning group rather than assuming your main GNS3 group membership covers it:

```bash
ls -l /usr/bin/ubridge
```

You may need to be added to a *separate* group specific to that binary (e.g. `ubridge`, distinct from the general `gns3` group):

```bash
sudo usermod -aG ubridge $USER
newgrp ubridge   # activates the new group in your current shell without logging out
```

Note that service accounts (like the `gns3` user itself) need a full `systemctl restart` to pick up new group membership — `newgrp` only affects your own interactive shell.

## Importing a Docker-based appliance (example: Arista cEOS)

Docker-based network OS images (like Arista's cEOS-lab) follow a different, generally simpler path — especially if a `.gns3a` appliance template file is available.

1. **Load the image into Docker manually first.** Vendor-distributed images are often raw filesystem tarballs rather than standard Docker save/export archives:

   ```bash
   sudo docker import /path/to/image.tar.xz your-tag:version
   ```

   If this errors out complaining about format, try `docker load -i` instead — depends on how the vendor packaged it.

2. **Import the `.gns3a` appliance template** via *File → Import appliance* in the GNS3 GUI. If your file picker's "Open" dialog seems to not find the file even though it clearly exists on disk, **check the file type filter at the bottom of the dialog** — it may be restricted to a specific extension (e.g. `*.gns3appliance`) that doesn't match your file's actual extension (e.g. `*.gns3a`). Switch the filter to "All Files" and try again.

3. **Match the tag exactly.** Appliance templates often expect the Docker image to exist under one *specific* tag baked into the template (e.g. `ceosimage:GNS3`). If you tagged your image something else during import, GNS3 will try to pull that expected tag from Docker Hub instead of using your local image — and fail with something like:

   ```
   404 pull access denied for ceosimage, repository does not exist or may require 'docker login'
   ```

   The fix isn't to re-import — just add a second tag pointing at the same image:

   ```bash
   sudo docker tag your-existing-tag:version ceosimage:GNS3
   ```

   Confirm both tags now point at the same image ID:

   ```bash
   sudo docker images | grep ceos
   ```

4. Retry creating the node from the template — it should now resolve locally without attempting a Docker Hub pull.

## Quick troubleshooting checklist

- **"No suitable disk was found"** → check HDA vs. CD/DVD assignment; also check for a stale per-node `hda_disk.qcow2`.
- **Permission denied running a helper binary** → check the binary's actual group ownership (`ls -l`), not just your main GNS3 group.
- **File picker can't find a file you know exists** → check the dialog's file-type filter.
- **"404 pull access denied"** on a Docker-based node → the appliance template expects a specific image tag; retag your local image to match rather than trying to pull from a registry.
- **Unsure where images actually live on a remote server** → search for a known bundled file (like `empty8G.qcow2`) rather than trusting documented default paths.
