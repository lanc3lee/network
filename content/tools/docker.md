# Setting Up Docker on Windows (Before You Start the Lab Guides)

If you're following along with the Prometheus/Grafana/Alertmanager or Zabbix lab guides on this site and don't have Docker running yet, start here. This covers a clean Windows install from scratch.

## What You Need

- Windows 10 (64-bit, version 2004+) or Windows 11
- 4GB RAM minimum, 8GB+ recommended if you'll run multiple stacks side by side
- Virtualization enabled in your BIOS/UEFI (most modern PCs have this on by default — see Step 2 if you hit issues)
- Administrator access on the machine

## Step 1: Install WSL 2 (Windows Subsystem for Linux)

Docker Desktop on Windows runs on top of WSL 2, which gives you a real Linux kernel running alongside Windows without a traditional VM.

Open **PowerShell as Administrator** and run:

```powershell
wsl --install
```

This installs WSL 2 and a default Ubuntu distribution. Restart your PC when prompted.

After restarting, verify it installed correctly:

```powershell
wsl --status
```

You want to see WSL version 2 listed as the default.

## Step 2: Enable Virtualization (Only If Step 1 Complains)

If `wsl --install` throws a virtualization error, it usually means it's disabled in your BIOS/UEFI:

1. Restart and enter your BIOS/UEFI setup (usually `Del`, `F2`, or `F10` at boot — varies by manufacturer)
2. Find the setting called **Intel VT-x**, **AMD-V**, or **SVM Mode** (naming depends on your CPU vendor)
3. Enable it, save, and exit

This is a one-time check — most machines bought in the last several years have this on already.

## Step 3: Install Docker Desktop

1. Download Docker Desktop for Windows from [docker.com](https://www.docker.com/products/docker-desktop/)
2. Run the installer, and make sure **"Use WSL 2 instead of Hyper-V"** is checked during setup (this is the default on modern installers)
3. Restart if prompted
4. Launch Docker Desktop from the Start menu and wait for the whale icon in your system tray to show it's running

## Step 4: Verify Everything Works

Open a terminal (PowerShell, Command Prompt, or your WSL Ubuntu shell) and run:

```powershell
docker --version
docker compose version
```

You should see version output for both, similar to:

```
Docker version 29.x.x, build xxxxxxx
Docker Compose version v5.x.x
```

Then confirm Docker can actually run a container:

```powershell
docker run hello-world
```

If you see a "Hello from Docker!" message, you're ready to go — head back to the lab guide you were following.

## Common Issues

**"WSL 2 installation is incomplete" on Docker Desktop startup**
Run `wsl --update` in an Administrator PowerShell, then restart Docker Desktop.

**Docker commands hang or time out**
Check the system tray — Docker Desktop needs to fully start (whale icon stops animating) before the CLI will respond. First launch after install can take a minute or two.

**Ports already in use when starting a lab stack**
Something else on your machine (often IIS, Skype, or another local service) may already be bound to a port a lab uses — commonly 80, 8080, or 3000. Either stop that service or edit the lab's `docker-compose.yml` to map to a different host port, e.g. `"3001:3000"` instead of `"3000:3000"`.

**Everything's slow**
Make sure your project files live inside the WSL filesystem (e.g. `\\wsl$\Ubuntu\home\yourname\`) rather than on the Windows `C:\` drive — cross-filesystem access between Windows and WSL is noticeably slower for anything Docker-heavy.

---

Once `docker run hello-world` works, you're set up the same way as the Mac instructions elsewhere on this site — every lab guide from here on assumes this baseline.
