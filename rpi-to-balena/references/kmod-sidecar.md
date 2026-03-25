# Kmod Sidecar Pattern

Build and load custom out-of-tree kernel modules on balenaOS using a dedicated sidecar container. This is the proven pattern from production balena projects (e.g. ai-hat-demo).

## When to Use

- The hardware requires a kernel module that is **not** in the stock balenaOS kernel.
- You have a **custom/modified** version of a driver that replaces the stock one.
- The vendor ships pre-compiled `.ko` files for RPi kernel versions, but balenaOS uses a different kernel.

**Do NOT use if** the driver is already in the balenaOS kernel and the stock version is sufficient — just set the dtoverlay and `modprobe` it.

## Architecture

```
docker-compose.yml
├── kmod-service/              # Privileged sidecar
│   ├── Dockerfile.template    # Multi-stage: ubuntu build → alpine runtime
│   ├── build.sh               # Fetches balenaOS kernel headers, compiles module
│   ├── load.sh                # insmod custom .ko or modprobe stock fallback
│   └── src/                   # Driver source + Makefile
└── app-service/               # Application container
    ├── Dockerfile
    └── entry.sh               # Polls lsmod, waits for module, launches app
```

## Why a Sidecar?

| Benefit                | Detail                                                                    |
| ---------------------- | ------------------------------------------------------------------------- |
| Separation of concerns | Kmod is tiny (Alpine). App carries the application stack.                 |
| Independent restart    | `restart: on-failure` retries module loading without restarting the app.  |
| Module persistence     | `sleep infinity` keeps the module loaded. Container stop = module unload. |
| OS version isolation   | Only the sidecar needs updating when the host OS changes.                 |

## Multi-Stage Dockerfile.template

### Stage 1 — Build (ubuntu:jammy)

Install build dependencies:

```dockerfile
FROM ubuntu:jammy AS kernel-build

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    bison build-essential flex libelf-dev libssl-dev bc wget ca-certificates kmod \
    && rm -rf /var/lib/apt/lists/*
```

Accept `OS_VERSIONS` build arg and compile for each:

```dockerfile
ARG OS_VERSIONS
ARG SRC_DIR=/usr/src
ARG OUT_DIR=/out

COPY build.sh /build.sh
COPY src/ ${SRC_DIR}/driver/

RUN if [ -n "${OS_VERSIONS}" ]; then \
      for version in ${OS_VERSIONS}; do \
        mkdir -p ${OUT_DIR}/${version} && \
        /build.sh -s ${SRC_DIR}/driver -o ${OUT_DIR}/${version} \
                  -v ${version} -m %%BALENA_MACHINE_NAME%%; \
      done; \
    else \
      echo "OS_VERSIONS not set — skipping build. Stock module used at runtime."; \
    fi
```

### Stage 2 — Runtime (alpine)

```dockerfile
FROM alpine:latest
RUN apk add --no-cache kmod bash
COPY --from=kernel-build /out/ /opt/lib/modules/
COPY load.sh /load.sh
RUN chmod +x /load.sh
ENTRYPOINT ["/load.sh"]
```

## build.sh — Fetching Kernel Headers

balena publishes kernel headers for every OS release:

```
https://files.balena-cloud.com/images/{device-slug}/{version}/kernel_modules_headers.tar.gz
```

For ESR releases, use `esr-images` instead of `images`.

### Detecting ESR

ESR versions follow `YYYY.MM.PATCH` (e.g. `2024.10.0`):

```bash
if [[ "${version}" =~ ^[1-3][0-9]{3}\.(1|01|4|04|7|07|10)\.[0-9]* ]]; then
    image_path="esr-images"
else
    image_path="images"
fi
```

### Version encoding

The `+` in version strings like `6.10.22+rev1` must be URL-encoded as `%2B`:

```bash
encoded_version=$(echo "${version}" | sed 's/+/%2B/g')
```

### Extracting headers

The tar archive has varying directory depth. Find it dynamically:

```bash
strip_depth=$(tar -tzf headers.tar.gz | grep "/\.config$" | tr -dc / | wc -c)
tar -xzf headers.tar.gz -C /usr/src/kernel-headers --strip-components="${strip_depth}"
```

### Compiling

```bash
make -C /usr/src/kernel-headers modules_prepare
make -C /usr/src/kernel-headers M=/path/to/driver/source modules
```

Copy the resulting `.ko` file(s) to the output directory.

## load.sh — Runtime Loading

```bash
#!/bin/bash
set -e

MODULE_NAME="my_module"
OS_VERSION=$(echo "$BALENA_HOST_OS_VERSION" | cut -d " " -f 2)
MOD_PATH="/opt/lib/modules/${OS_VERSION}"

# Search for custom .ko (handle hyphen/underscore variants)
MODULE_FILE=""
for candidate in "${MOD_PATH}/my-module.ko" "${MOD_PATH}/my_module.ko"; do
    [ -f "${candidate}" ] && MODULE_FILE="${candidate}" && break
done

if [[ -n "${MODULE_FILE}" ]]; then
    # Unload stock module if present, then load custom
    lsmod | grep -q "^${MODULE_NAME}" && rmmod "${MODULE_NAME}" || true
    insmod "${MODULE_FILE}"
else
    # Fall back to stock in-tree module
    lsmod | grep -q "^${MODULE_NAME}" || modprobe "${MODULE_NAME}"
fi

# Verify
lsmod | grep -q "^${MODULE_NAME}" || { echo "Module load failed"; exit 1; }

# Keep alive so module stays loaded
sleep infinity
```

## OS Version Syncing

The main friction: a `.ko` compiled against kernel X won't load on kernel Y.

| Strategy                            | When to use                                                                      |
| ----------------------------------- | -------------------------------------------------------------------------------- |
| **Pin OS version**                  | Production fleets. Update `OS_VERSIONS` when you deliberately upgrade the fleet. |
| **Pre-build for multiple versions** | Rolling upgrades. List multiple in `OS_VERSIONS` (e.g. `"6.10.22 6.10.24"`).     |
| **Fall back to stock module**       | When stock works for most cases and custom is only for specific features.        |

## Compose Configuration

```yaml
services:
  kmod:
    build:
      context: ./kmod
      dockerfile: Dockerfile.template
      args:
        OS_VERSIONS: ""
    privileged: true
    restart: on-failure
    labels:
      io.balena.features.kernel-modules: "1"
    network_mode: host

  app:
    build: ./app
    privileged: true
    depends_on:
      - kmod
```

## Reference Projects

- [balena-solutions/ai-hat-demo](https://github.com/balena-solutions/ai-hat-demo) — production HAILO AI Hat kmod sidecar
- [balena-os/kernel-module-build](https://github.com/balena-os/kernel-module-build) — generic kernel module build reference
- [balena blog: Building out-of-tree Linux kernel modules](https://blog.balena.io/building-out-of-tree-linux-kernel-modules/)
