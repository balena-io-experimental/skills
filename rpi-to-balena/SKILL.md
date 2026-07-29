---
name: rpi-to-balena
description: "Guide for porting Raspberry Pi projects to balenaOS, particularly when custom kernel modules, device-tree overlays, or installation scripts modify config.txt, run modprobe/dtparam, or install .ko drivers. Use when converting RPi hardware projects to balena, debugging install scripts that fail in containers, handling out-of-tree kernel modules on balenaOS, migrating config.txt and dtoverlay settings to balenaCloud dashboard variables, or analysing vendor install scripts that fail during Docker build."
---

# Port Raspberry Pi Projects to balenaOS

## What This Skill Covers

Converting Raspberry Pi hardware projects — especially those shipping install scripts that modify `config.txt`, load kernel modules, or call `dtparam` — to run on balenaOS inside containers. This includes adapting vendor scripts, building out-of-tree kernel modules, and wiring up device-tree configuration via the balenaCloud dashboard.

Use as a reference whenever an RPi install script fails in a Docker build, or you need to load a custom `.ko` on balenaOS.

## The Core Problem

RPi hardware install scripts assume bare-metal Raspberry Pi OS:

1. Direct writes to `/boot/config.txt` → **read-only on balenaOS**
2. `modprobe`/`insmod` at setup time → **no kernel during Docker build**
3. `dtparam` calls → **not available in containers**
4. Platform detection via `uname -r`, `dpkg-query raspberrypi-kernel` → **returns different values on balenaOS**
5. Pre-compiled `.ko` matched to RPi kernel versions → **balenaOS runs a different kernel**

## Conversion Procedure

### Step 1: Analyse the Install Script

Read the vendor script end-to-end and tag every action:

| Action                                         | Category          | Where it runs on balena                |
| ---------------------------------------------- | ----------------- | -------------------------------------- |
| `apt install pkg.deb` / `wget .deb && dpkg -i` | Package install   | Dockerfile (build-time)                |
| `echo "dtoverlay=X" >> /boot/config.txt`       | Config.txt        | Dashboard variable — not in code       |
| `dtparam i2c_arm=on`                           | Device-tree param | Dashboard variable — not in code       |
| `modprobe module` / `insmod file.ko`           | Module load       | Runtime only — kmod sidecar            |
| `wget precompiled.ko`                          | Driver download   | **Do not use** — recompile from source |
| `sed -i /boot/config.txt ...`                  | Config.txt        | Dashboard variable — not in code       |
| `i2cdetect`, `vcgencmd`, `raspi-config`        | Hardware probe    | Runtime only — privileged container    |
| Read `uname -r`, `dpkg-query`                  | Platform detect   | **Caution** — returns different values |

### Step 2: Split Build-time vs Runtime

- **Build-time (Dockerfile):** Package installation, library compilation, application setup. Works as long as no kernel or hardware is needed.
- **Runtime (entry.sh / sidecar):** Module loading, hardware probing. Needs a running kernel and real hardware.
- **Dashboard only:** All `config.txt` / `dtoverlay` / `dtparam` changes. Cannot be shipped in code.

### Step 3: Handle Build-time Script Failures

If the install script fails during `docker build` because it calls runtime-only tools:

1. **Stub the trigger condition.** If the script checks for `/dev/i2c-0` and runs `modprobe`/`dtparam` when missing:
   ```dockerfile
   RUN mkdir -p /dev && touch /dev/i2c-0
   RUN ./vendor-install-script.sh -p library_package
   ```
2. **Run only the safe steps.** If the script separates library install from driver install, call it multiple times with only the package flags.
3. **Extract the downloads.** Read the script, find the `.deb` URLs, and replicate them directly in the Dockerfile.

### Step 4: Map config.txt to Dashboard Variables

Every `config.txt` change becomes a `BALENA_HOST_CONFIG_*` variable:

| Script action         | Dashboard variable                                       |
| --------------------- | -------------------------------------------------------- |
| `dtoverlay=my-device` | `BALENA_HOST_CONFIG_dtoverlay` = `"my-device"`           |
| `dtparam=i2c_arm=on`  | `BALENA_HOST_CONFIG_dtparam` = `"i2c_arm=on"`            |
| `gpu_mem=128`         | `BALENA_HOST_CONFIG_gpu_mem` = `128`                     |
| Multiple dtoverlays   | `BALENA_HOST_CONFIG_dtoverlay` = `"overlay1","overlay2"` |

Rules:
- The part after `BALENA_HOST_CONFIG_` becomes the config key.
- Multiple values for the same key: quote each and comma-separate.
- Changes take effect on reboot (automatic when saved in dashboard).
- **Document these prominently** in the project README.

### Step 5: Handle Kernel Modules

**If the module is already in the balenaOS kernel** (most common RPi drivers are):
- Set `BALENA_HOST_CONFIG_dtoverlay` to enable the driver at boot.
- Or `modprobe module_name` at runtime from a privileged container.
- No compilation needed.

**If you need a custom/modified module:**
- Use the kmod sidecar pattern — see [Kmod Sidecar Reference](./references/kmod-sidecar.md).

### Step 6: Wire Up the Application Service

```bash
# entry.sh — wait for module, then launch
echo "Waiting for my_module kernel module..."
for i in $(seq 1 60); do
    if lsmod | grep -q "my_module"; then
        echo "Module loaded."
        break
    fi
    if [ "$i" -eq 60 ]; then
        echo "ERROR: Module not loaded after 60s. Check kmod service logs."
        sleep infinity
        exit 1
    fi
    sleep 1
done
exec python3 /app/main.py
```

### Step 7: Document Dashboard Requirements

Every `config.txt` change the original script made must be documented for the user:

```markdown
## Required Device Configuration

Set in balenaCloud dashboard → **Fleet → Device Configuration**:

| Variable                       | Value        | Purpose              |
| ------------------------------ | ------------ | -------------------- |
| `BALENA_HOST_CONFIG_dtoverlay` | `my-device`  | Enable device driver |
| `BALENA_HOST_CONFIG_dtparam`   | `i2c_arm=on` | Enable I2C bus       |
```

## Quick-Reference: Compose Template

```yaml
version: "2.1"

services:
  kmod:
    build:
      context: ./kmod
      dockerfile: Dockerfile.template
      args:
        OS_VERSIONS: ""   # Space-separated balenaOS versions, or empty for stock module
    privileged: true
    restart: on-failure
    labels:
      io.balena.features.kernel-modules: "1"
    network_mode: host

  app:
    build: ./app
    depends_on:
      - kmod
    volumes:
      - app-data:/data

volumes:
  app-data:
```

While the `kmod` service handles building and loading the kernel module, the `app` service can be your main application that depends on the module being loaded.
The `app` service should prefer to use the `devices` key within within the `docker-compose.yml` file to access the hardware interfaces exposed by the kernel module, rather using privileged mode.
However, privileged mode is still required for the `kmod` service to load the kernel module.

## Quick-Reference: Common Dockerfile Pattern for running installation scripts from peripheral vendors

```dockerfile
FROM debian:bookworm

RUN apt-get update && apt-get install -y wget ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Run vendor install script — package steps only, skip kernel driver
RUN wget -O install.sh https://vendor.example.com/install.sh \
    && chmod +x install.sh \
    && ./install.sh \
    && rm install.sh

COPY entry.sh /entry.sh
RUN chmod +x /entry.sh
ENTRYPOINT ["/entry.sh"]
```

Use `Dockerfile.template` only when you need `%%BALENA_MACHINE_NAME%%` or `%%BALENA_ARCH%%` variable substitution. Plain `Dockerfile` works for everything else.

## Further Reference

- [Kmod Sidecar Pattern](./references/kmod-sidecar.md) — building and loading custom out-of-tree kernel modules
- [Common Pitfalls](./references/pitfalls.md) — wrong driver repo, version mismatches, naming conventions, silent failures
- [Blog Article: Use Out-of-Tree Linux Kernel Modules in balena](./references/blog-use-out-of-tree-modules.md) — detailed guide on building and loading custom kernel modules on balenaOS, including handling kernel headers and multi-stage Dockerfiles.
