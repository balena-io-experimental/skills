---
name: balenify
description: "Guide for converting Docker projects to run on balenaCloud. Use this skill when balenifying or converting an existing Docker/docker-compose project for balena, writing or modifying Dockerfile.template files, setting up docker-compose.yml for balena fleets, choosing between Dockerfile.template and plain Dockerfile for balena services, debugging balena build failures, converting volume mounts for balena, handling environment variables on balena, configuring hardware access (GPIO, USB, serial) for containers, or understanding balena networking. Also use when the user mentions balena device types, %%BALENA_ARCH%%, %%BALENA_MACHINE_NAME%%, balenalib base images, bh.cr registry references, or asks how to run something on balena."
---

# Balenify — Convert Docker Projects to balenaCloud

## What This Skill Covers

This skill covers the full workflow for converting a standard Docker or docker-compose project to run on balenaCloud. That includes Dockerfile templates for multi-architecture builds, volume mount conversion, networking differences, environment variable handling, hardware access, docker-compose.yml adaptations, and balena.yml fleet metadata.

Use this as a reference whenever you're porting an existing project to balena or building a new balena fleet from scratch.

## Balenify Checklist

Quick reference for the conversion steps:

- [ ] Replace host bind mounts (`./data:/app`) with named volumes
- [ ] Remove `.env` files — use dashboard/CLI variables instead
- [ ] Add `balena.yml` with fleet name and supported device types
- [ ] Convert Dockerfiles to `.template` if arch-specific base images are needed
- [ ] Add balena-specific labels only where strictly needed — each label weakens container isolation (see security table)
- [ ] Use selective `devices:`/`cap_add:` for hardware access — if `privileged: true` or `cap_add: [SYS_ADMIN]` is unavoidable, add a `# WARNING:` comment in the composition explaining why
- [ ] Use custom internal networks for inter-service communication — never use `network_mode: host` for this purpose
- [ ] Only use `network_mode: host` as a last resort for protocols requiring raw host network access (mDNS, DHCP) — with user acknowledgment
- [ ] Remove `links:` — bridge network provides service discovery by container name
- [ ] Pin `version: "2.1"` in docker-compose.yml (balena uses Compose v2/2.1)
- [ ] Pin image versions — never use `:latest` in docker-compose.yml
- [ ] Review `restart` policies — balena defaults to `always`; the supervisor does not override the compose restart policy, so `restart: "no"` works for one-shot containers

## Dockerfile Templates

Balena's build system supports Dockerfile templates — files named `Dockerfile.template` that undergo variable substitution before Docker processes them. The template system exists because Docker images are architecture-specific, and balena fleets often span multiple device types (e.g., Raspberry Pi 3, Pi 4, Intel NUC) with different CPU architectures.

### Core Concept: Template Variable Substitution

Template variables use `%%VARIABLE%%` syntax. The balena builder replaces them with fleet/device-specific values *before* the Docker build begins — Docker never sees the `%%` tokens.

#### Available Variables

| Variable | Description | Example Value |
|----------|-------------|---------------|
| `%%BALENA_ARCH%%` | Target CPU architecture | `aarch64`, `amd64`, `armv7hf` |
| `%%BALENA_MACHINE_NAME%%` | Yocto machine name for the device type | `raspberrypi4-64`, `genericx86-64-ext` |
| `%%BALENA_APP_NAME%%` | Fleet name | `my-fleet` |
| `%%BALENA_RELEASE_HASH%%` | Release hash | `abc123...` |
| `%%BALENA_SERVICE_NAME%%` | Service name from docker-compose.yml | `pihole` |

**Use `%%BALENA_ARCH%%` by default.** All template variables are resolved at build time to fleet-level values — there is no per-device substitution. `%%BALENA_MACHINE_NAME%%` resolves to the fleet's default device type, so in mixed fleets it produces the wrong value for non-default device types. `%%BALENA_ARCH%%` is preferred because all devices in a fleet share the same architecture.

### Dockerfile Resolution Order

The balena builder picks the *first match* from this priority list:

1. `Dockerfile.<device-type>` — exact device type match (e.g., `Dockerfile.raspberrypi4-64`)
2. `Dockerfile.<arch>` — architecture match (e.g., `Dockerfile.aarch64`)
3. `Dockerfile.template` — universal fallback with variable substitution
4. `Dockerfile` — plain Dockerfile, no substitution

This applies during `balena push`, `balena build`, `balena deploy`, and `git push`.

**When to use templates vs plain Dockerfiles:**

- If your base image is multi-platform (published with Docker manifest lists) or you only target one architecture → use a plain `Dockerfile`
- If you need to select between architecture-specific images that lack multi-platform manifests → use `Dockerfile.template`
- Most services in a fleet can use plain Dockerfiles. Only the ones pulling arch-specific images need templates.

### Balena Arch vs Docker/Go Arch

Balena has its own architecture naming convention, distinct from Docker and Go:

| Balena (`%%BALENA_ARCH%%`) | Docker Platform | Go GOARCH |
|---|---|---|
| `aarch64` | `linux/arm64` | `arm64` |
| `amd64` | `linux/amd64` | `amd64` |
| `armv7hf` | `linux/arm/v7` | `arm` (GOARM=7) |
| `rpi` | `linux/arm/v6` | `arm` (GOARM=6) |
| `i386` | `linux/386` | `386` |

This matters because each balenaCloud fleet has a default device type and all devices in the fleet share the same architecture. Blocks on bh.cr are published as separate images per balena arch (e.g., `my-block-aarch64`, `my-block-amd64`), not as multi-platform manifests. Templates with `%%BALENA_ARCH%%` map directly to these naming conventions.

### The Simple Pattern (Use This by Default)

For most cases, a single FROM line with `%%BALENA_ARCH%%` interpolated into the image reference is all you need:

```dockerfile
FROM bh.cr/some-org/my-block-%%BALENA_ARCH%%/1.0.0
```

Or with a standard registry:

```dockerfile
FROM myregistry/myimage-%%BALENA_ARCH%%:1.0.0

COPY . /app
CMD ["/app/start.sh"]
```

This pattern works especially well with bh.cr, where blocks and fleets are published per-architecture using balena's naming convention. The builder substitutes `%%BALENA_ARCH%%` → `aarch64` (or `amd64`, `armv7hf`, etc.) and Docker pulls the right image. Note that bh.cr uses path segments for versioning — the Docker tag after `:` is not used for image resolution (see `references/patterns.md` for details).

### Multi-Stage Pattern

Use this when the simple pattern won't work. The two main reasons:

1. **Arch naming mismatch**: The upstream image uses Docker/Go arch names (`arm64`, `arm64v8`) instead of balena names (`aarch64`, `armv7hf`). A simple `FROM image-%%BALENA_ARCH%%:tag` would try to pull `image-aarch64:tag` which doesn't exist.
2. **Renovate/Dependabot pinning**: You want automated dependency tooling to independently update per-arch image references. The simple `%%BALENA_ARCH%%` pattern embeds a variable inside the image reference, which Renovate can't parse.

```dockerfile
ARG BALENA_ARCH=%%BALENA_ARCH%%

# Stage names use balena arch convention — map to whatever the upstream publishes
FROM linuxserver/wireguard:amd64-latest AS wireguard-amd64
FROM linuxserver/wireguard:arm64v8-latest AS wireguard-aarch64

# hadolint ignore=DL3006
FROM wireguard-${BALENA_ARCH}
```

The key trick: name each stage with the **balena** arch suffix (e.g., `wireguard-aarch64`) regardless of what the upstream tag calls it (e.g., `arm64v8-latest`). Then `FROM wireguard-${BALENA_ARCH}` resolves correctly.

**How it works:**

1. `%%BALENA_ARCH%%` is substituted by the balena builder before Docker runs
2. Docker ARG captures the resolved value
3. Named FROM stages alias each upstream image to balena arch names
4. `FROM wireguard-${BALENA_ARCH}` resolves to the correct stage
5. Docker pulls all FROM images during the build (balenaEngine does not use BuildKit), but only the selected stage ends up in the final image

The `# hadolint ignore=DL3006` comment suppresses the "always tag the version of an image explicitly" lint warning on the dynamic FROM line.

## Volume Mounts

Host bind mounts (`./data:/app/data`) do **not** work on balena. The host filesystem layout is managed by balenaOS and is not directly accessible to containers.

### Use Named Volumes

Declare volumes in the top-level `volumes:` section and reference them in services:

```yaml
version: "2.1"

volumes:
  app-data:
  config:

services:
  my-service:
    build: my-service
    volumes:
      - app-data:/app/data
      - config:/etc/myapp
```

### Key behaviors

- Named volumes **persist across container updates** as long as the volume name stays the same
- Shared volumes between services: mount the same named volume in multiple services for inter-service data exchange
- **Single-container apps** get a default `resin-data` volume mounted at `/data` — no explicit volume config needed
- **Warning**: volumes are purged if a device is moved to a different fleet

### Converting bind mounts

| Docker (original) | Balena (converted) |
|---|---|
| `./data:/app/data` | `app-data:/app/data` (+ declare `app-data:` in `volumes:`) |
| `./config.json:/etc/app/config.json` | Bake into image with `COPY`, or use an env var |
| `/var/run/docker.sock:/var/run/docker.sock` | Use `io.balena.features.balena-socket: 1` label instead |

## Networking

### Networking: single-container apps

> **Deprecation warning:** Single-container apps run with `privileged: true` and `network_mode: host` by default, which breaks all container isolation. **Always prefer multicontainer apps** (even for a single service) to maintain proper security boundaries. Single-container mode exists for legacy compatibility only.

### Networking: multicontainer apps

Bridge network by default — each service gets its own network namespace. Services are routable by container name.

**Prefer custom networks for inter-service communication** when services don't need external access:

```yaml
services:
  frontend:
    build: frontend
    ports:
      - "80:3000"   # expose to host/external network
    networks:
      - default
      - internal

  backend:
    build: backend
    # no ports needed — frontend reaches backend at http://backend:8080
    networks:
      - internal

networks:
  internal:
    driver: bridge
    internal: true   # no external access — services only
```

### Key points

- **Use custom internal networks** for inter-service communication when no external access is needed — this limits blast radius if a container is compromised
- `ports:` only needed to expose a service to the host network or external clients — **not** for inter-service communication
- No need for `links:` — bridge networks provide automatic service discovery by container name
- `network_mode: host` — **dangerous, last resort only.** Gives the container full access to the host network stack. Only use when strictly required (e.g., mDNS, DHCP, network scanning) and warn the user about the security implications

## Environment Variables

### No `.env` files

Balena does not read `.env` files. Variables can be set through:

- **docker-compose.yml** → `environment:` under each service (baked into the release)
- **balenaCloud dashboard** → fleet or device variables (with optional per-service scope)
- **CLI**: `balena env set MY_VAR my_value --fleet my-fleet` or `--device <uuid>`

### Variable priority (lowest to highest)

1. `ENV` in Dockerfile
2. `environment:` in docker-compose.yml service
3. Fleet variable (all services)
4. Fleet variable (service-specific)
5. Device variable (all services)
6. Device variable (service-specific)

### Balena-injected variables

These are automatically available in every container (partial list — see [balena docs](https://docs.balena.io/reference/supervisor/supervisor-api/#environment-variables) for the complete set):

| Variable | Value |
|----------|-------|
| `BALENA` | `1` (use to detect running on balena) |
| `BALENA_DEVICE_UUID` | Device UUID |
| `BALENA_DEVICE_TYPE` | Device type slug (e.g., `raspberrypi4-64`) |
| `BALENA_APP_ID` | Fleet/application numeric ID |
| `BALENA_APP_NAME` | Fleet name |
| `BALENA_SERVICE_NAME` | Service name from docker-compose.yml |

The following are **only** injected when the `io.balena.features.supervisor-api: 1` label is set on the service:

| Variable | Value |
|----------|-------|
| `BALENA_SUPERVISOR_ADDRESS` | Supervisor API URL (usually `http://127.0.0.1:48484`) |
| `BALENA_SUPERVISOR_API_KEY` | API key for supervisor requests |

Values can be up to 1MB each.

### Converting from `.env` files

Do **not** pipe `.env` files through automated tooling — they likely contain secrets. Instead, have the user set each variable manually:

1. Open the `.env` file and review each variable
2. For each `KEY=VALUE` pair, run: `balena env set KEY value --fleet my-fleet`
3. Or set them in the balenaCloud dashboard under fleet/device variables

## Hardware Access

### Hardware: single-container apps

> **Deprecation warning:** Single-container apps run privileged by default — full access to `/dev` and the entire host. This is a legacy behavior. **Always prefer multicontainer apps** where you explicitly grant only the access needed.

### Hardware: multicontainer apps

**Not** privileged by default. Follow the **least-privilege principle** — grant only the minimum access required and always explain to the user what security boundary is being relaxed and why.

**Option 1 — Selective device access (preferred):**

```yaml
services:
  my-service:
    build: my-service
    devices:
      - "/dev/i2c-1:/dev/i2c-1"
      - "/dev/ttyUSB0:/dev/ttyUSB0"
    cap_add:
      - SYS_RAWIO
```

**Option 2 — Full privileged mode (dangerous — last resort):**

> **Warning:** `privileged: true` disables ALL container security boundaries. Only use when selective access is genuinely impossible. Always warn the user and get explicit acknowledgment before adding this.

```yaml
services:
  my-service:
    build: my-service
    privileged: true
```

**Always start with the lowest-privilege option that works** — selective `devices:` mappings + specific `cap_add:` entries (e.g., `SYS_RAWIO`, `NET_ADMIN`). Only escalate if the specific capabilities are genuinely insufficient.

Both `privileged: true` and `cap_add: [SYS_ADMIN]` are **equally dangerous** from a threat model perspective — both enable container escape and effectively grant root on the host. `CAP_SYS_ADMIN` is arguably worse because it creates a false sense of containment while providing nearly equivalent access.

If either `privileged: true` or `cap_add: [SYS_ADMIN]` is required, **always add a warning comment directly in the docker-compose.yml** explaining why, and get explicit user acknowledgment before adding it:

```yaml
services:
  my-service:
    build: my-service
    # WARNING: privileged mode required for [specific reason].
    # This disables ALL container security boundaries.
    # See: https://docs.balena.io/learn/develop/hardware/
    privileged: true
```

### Balena feature labels

> **Security warning:** Most feature labels grant broad access that breaks container isolation. Use them only when strictly necessary, with explicit user acknowledgment, and document why each label is needed.

Security impact levels: **Low** = scoped to specific hardware. **Moderate** = can affect device lifecycle. **High** = equivalent to or approaching root on host.

| Label | Grants access to | Security impact |
|-------|-----------------|-----------------|
| `io.balena.features.supervisor-api: 1` | Supervisor REST API | Moderate — can control device lifecycle |
| `io.balena.features.balena-api: 1` | balenaCloud API (device-scoped key) | Moderate — can read/modify own device data |
| `io.balena.features.gpu: 1` | GPU device access | Low — scoped to GPU hardware |
| `io.balena.features.dbus: 1` | Host D-Bus socket | **High — exposes host system bus** |
| `io.balena.features.balena-socket: 1` | balenaEngine socket | **High — equivalent to root on host** |
| `io.balena.features.kernel-modules: 1` | Kernel module loading | **High — can modify kernel behavior** |
| `io.balena.features.sysfs: 1` | Host `/sys` filesystem | **High — exposes hardware/kernel config** |
| `io.balena.features.procfs: 1` | Host `/proc` filesystem | **High — exposes all host process info** |

For specific device access (I2C, SPI, UART, USB), **prefer explicit `devices:` mappings** over feature labels — `devices:` gives you granular control over exactly which hardware paths are exposed, while labels grant access with zero granularity. The exception is `io.balena.features.gpu` which is the only practical way to grant GPU access.

## docker-compose.yml for Balena

Balena uses Docker Compose v2/2.1. Each service with a `build:` directive points to a directory containing a Dockerfile (or Dockerfile.template).

```yaml
version: "2.1"

services:
  my-service:
    build: my-service
    tmpfs:
      - /tmp
      - /var/run

  # Pre-built images use `image:` directly — no Dockerfile or template
  helper:
    image: bh.cr/some-org/some-block/1.0.0
    restart: "no"
```

> Only add `privileged`, `cap_add`, `network_mode: host`, or feature labels when the service genuinely requires them. See the Hardware Access and Networking sections for security guidance.

### Update Strategy Labels

Control how balena updates containers:

| Label | Values | Default |
|-------|--------|---------|
| `io.balena.update.strategy` | `download-then-kill`, `kill-then-download`, `delete-then-download`, `hand-over` | `download-then-kill` |
| `io.balena.update.handover-timeout` | Seconds (for `hand-over` strategy) | See [balena docs](https://docs.balena.io/learn/deploy/release-strategy/update-strategies/) |

- `download-then-kill` — download new image, then stop old container (minimal downtime, needs 2x storage)
- `kill-then-download` — stop old container first, then download (saves storage, longer downtime)
- `delete-then-download` — delete old image and container, then download (maximum storage savings)
- `hand-over` — old and new containers run simultaneously during handover period (zero-downtime deploys)

## balena.yml

A `balena.yml` is optional for basic fleet projects but required for balenaHub publishing. When present, `name` and `type` are the minimum useful fields:

```yaml
name: "My Fleet"
type: "sw.application"
version: 1.0.0
```

Optional fields for balenaHub publishing:

```yaml
description: "What this fleet does"
data:
  defaultDeviceType: "raspberrypi4-64"
  supportedDeviceTypes:
    - "raspberrypi3-64"
    - "raspberrypi4-64"
    - "genericx86-64-ext"
```

The `supportedDeviceTypes` list is used for display on balenaHub, not enforced by the build system. Recommended if publishing a public app, fleet, or block.

## Debugging Build Failures

| Symptom | Likely Cause |
|---------|-------------|
| `invalid reference format` | Template variable not substituted — check file is named `.template` |
| `manifest unknown` | Architecture not published for that image tag |
| `no match for platform` | Using a plain Dockerfile where a template is needed |
| Wrong arch binary runs | `%%BALENA_MACHINE_NAME%%` resolved to fleet default, not target device |

## Common Patterns Reference

Read `references/patterns.md` for:

- Legacy `%%RESIN_MACHINE_NAME%%` syntax (pre-2019 projects)
- Balenalib base images (`FROM balenalib/%%BALENA_MACHINE_NAME%%-alpine`)
- `install_packages` helper in balenalib images
- Multi-stage selection with `TARGETARCH` fallback
- The `INITSYSTEM on` ENV (legacy balenalib entrypoint integration)
- bh.cr (balena hub container registry) image references
- Renovate configuration patterns for auto-updating image tags

## Workflow

When converting a Docker project to balena:

1. Check `balena.yml` → what device types does this fleet target?
2. Map device types to architectures (see table above)
3. For each service: does its base image support multi-platform? If yes → plain Dockerfile. If no → Dockerfile.template with the simple `%%BALENA_ARCH%%` pattern.
4. Replace bind mounts with named volumes.
5. Remove `.env` files — migrate variables to balenaCloud dashboard or CLI.
6. Add balena-specific labels only where strictly necessary — warn the user about security implications of each (see feature labels table).
7. Use selective `devices:`/`cap_add:` for hardware access. If `privileged: true` or `cap_add: [SYS_ADMIN]` is unavoidable, add a `# WARNING:` comment in the docker-compose.yml explaining why — require explicit user acknowledgment.
8. Test with `balena build --deviceType <type>` for each target.
