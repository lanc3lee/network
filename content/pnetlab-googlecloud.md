

startup.sh
```
#!/bin/bash
# Runs as root on every boot (GCE startup-script behavior).
# Guarded by a marker file so the install only happens once.

set -e

MARKER=/var/lib/pnetlab-install-done
LOG=/var/log/pnetlab-startup.log

exec >> "$LOG" 2>&1
echo "=== startup-script run: $(date) ==="

if [ -f "$MARKER" ]; then
  echo "Marker found, install already completed. Exiting."
  exit 0
fi

export DEBIAN_FRONTEND=noninteractive

echo "--- Creating swap file (half of RAM) ---"
if [ ! -f /swapfile ]; then
  RAM_MB=$(free -m | awk '/^Mem:/{print $2}')
  SWAP_MB=$((RAM_MB / 2))
  fallocate -l "${SWAP_MB}M" /swapfile
  chmod 600 /swapfile
  mkswap /swapfile
  swapon /swapfile
  echo '/swapfile none swap sw 0 0' >> /etc/fstab
fi

echo "--- Confirming nested virtualization is visible to the guest ---"
grep -cw vmx /proc/cpuinfo || echo "WARNING: vmx not found in /proc/cpuinfo"

echo "--- Adding PNetLab apt repo ---"
echo "deb [trusted=yes] http://repo.pnetlab.com ./" | tee -a /etc/apt/sources.list

echo "--- Updating package lists ---"
apt-get update

echo "--- Installing PNetLab (this pulls a large dependency set, be patient) ---"
apt-get install -y pnetlab

echo "--- GCP-specific kernel/grub cleanup (per official PNetLab GCP guide) ---"
rm -f /etc/default/grub.d/50-cloudimg-settings.cfg
update-grub

touch "$MARKER"

echo "--- Install finished, rebooting to complete setup ---"
reboot
```


Cost impact is small — Ubuntu Pro adds a per-vCPU/RAM license fee (roughly $0.01–0.02/hr for an n2-standard-8), on top of normal compute cost.