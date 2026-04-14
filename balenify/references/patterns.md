# Balena Dockerfile Template Patterns Reference

## Table of Contents

- [bh.cr Registry References](#bhcr-registry-references)
- [Deprecated Balenalib Base Images](#deprecated-balenalib-base-images)
- [Legacy Resin Syntax](#legacy-resin-syntax)
- [Multi-Stage Selection Pattern](#multi-stage-selection-pattern)
- [INITSYSTEM ENV](#initsystem-env)
- [install_packages Helper](#install_packages-helper)
- [Renovate Configuration](#renovate-configuration)
- [Device Type to Architecture Mapping](#device-type-to-architecture-mapping)

---

## bh.cr Registry References

balena hub container registry (`bh.cr`) hosts blocks and fleet images. Images are published per-architecture using balena's arch naming convention.

**URL format:** `bh.cr/<org>/<block-name>[-<arch>]/<version-or-hash>`

The version or content hash is always a **path segment**, not a Docker tag. The tag after `:` is not used for image resolution by bh.cr — it serves as a visual reference for humans and tooling (e.g., version tracking, automation).

Real examples:

```dockerfile
# Version as path segment
FROM bh.cr/balenalabs/fbcp/1.0.4

# With arch in path
FROM bh.cr/g_tomas_migone1/hostname-%%BALENA_ARCH%%/0.2.1

# Content hash as path segment, version as tag (for reference only)
FROM bh.cr/gh_klutchell/tailscale-amd64/ebf61ab1515195f9df43aa64baf15c39:1.88.1
```

When using `%%BALENA_ARCH%%` in bh.cr references, the arch is part of the image path, not a tag:

```dockerfile
# Correct — arch in path
FROM bh.cr/some-org/my-block-%%BALENA_ARCH%%/1.0.0

# Wrong — arch as tag
FROM bh.cr/some-org/my-block:%%BALENA_ARCH%%-1.0.0
```

---

## Deprecated Balenalib Base Images

Balenalib base images are **officially deprecated** and no longer maintained. They are still present in many existing projects but should be migrated away from.

```dockerfile
# Alpine-based
FROM balenalib/%%BALENA_MACHINE_NAME%%-alpine:3.18

# With runtime
FROM balenalib/%%BALENA_MACHINE_NAME%%-node:18-bookworm

# Architecture-based (less common)
FROM balenalib/%%BALENA_ARCH%%-alpine
```

Balenalib images include the `install_packages` helper (see below) and were designed for balena's init system.

**For new projects**, use standard images (Alpine, Debian, etc.) or bh.cr blocks instead. Note: the `%%BALENA_MACHINE_NAME%%` and `%%BALENA_ARCH%%` template variables are **not** deprecated — they are core to balena's Dockerfile template system and are used for many purposes beyond balenalib images (blocks, arch-specific build steps, etc.).

---

## Legacy Resin Syntax

Pre-2019 projects used `%%RESIN_MACHINE_NAME%%` and `%%RESIN_ARCH%%` (before the Resin → Balena rename):

```dockerfile
FROM resin/%%RESIN_MACHINE_NAME%%-alpine-golang

ENV INITSYSTEM on

WORKDIR /go/src/app
COPY . ./
RUN go build
CMD ./myapp
```

Both `RESIN_*` and `BALENA_*` variables still work, but use `BALENA_*` for anything new.

---

## Multi-Stage Selection Pattern

The balena builder does not set BuildKit variables like `TARGETARCH`. Use `%%BALENA_ARCH%%` directly:

```dockerfile
ARG BALENA_ARCH=%%BALENA_ARCH%%

FROM certbot/dns-cloudflare:amd64-v1.30.0 AS certbot-amd64
FROM certbot/dns-cloudflare:arm64v8-v1.30.0 AS certbot-aarch64

FROM certbot-${BALENA_ARCH}
```

Note the aliasing: `certbot-aarch64` points to `arm64v8` because Docker uses `arm64` while balena uses `aarch64`. Add stages for each target architecture (e.g., `certbot-armv7hf` for armv7hf) or the final FROM will fail to resolve.

---

## INITSYSTEM ENV

Legacy balenalib images included a systemd-based init system in the container entrypoint. `ENV INITSYSTEM on` activated it:

```dockerfile
ENV INITSYSTEM on
```

This is a legacy pattern — modern balena projects don't need it. It enabled PID 1 process management (zombie reaping, signal forwarding) via the balenalib entrypoint, not the balena supervisor.

---

## install_packages Helper

Balenalib base images include `install_packages`, a cross-distro package installer:

```dockerfile
FROM balenalib/%%BALENA_MACHINE_NAME%%-alpine:3.18
RUN install_packages curl jq
```

It abstracts over `apt-get`, `apk`, `dnf`, etc. based on the base distro. Only available in balenalib images — for standard base images, use the native package manager directly.

---

## Renovate Configuration

For projects using the multi-stage pattern (separate FROM per arch), Renovate can pin and auto-update each image independently.

**renovate.json example** (from a real balena-pihole project):

```json
{
  "extends": ["github>klutchell/renovate-config"],
  "packageRules": [
    {
      "matchManagers": ["dockerfile"],
      "matchPackageNames": ["pihole/pihole"],
      "matchUpdateTypes": ["major", "minor", "patch"],
      "postUpgradeTasks": {
        "commands": [
          "sed -e \"s|^version: .*$|version: {{{newVersion}}}|\" -e \"s|\\b0\\+\\([0-9]\\)|\\1|g\" -i balena.yml"
        ],
        "fileFilters": ["balena.yml"],
        "executionMode": "update"
      }
    }
  ]
}
```

This syncs the `version:` field in `balena.yml` when Renovate bumps the pihole image tag.

**Why multi-stage helps Renovate:** The simple `%%BALENA_ARCH%%` pattern puts a template variable inside the image reference, which Renovate can't parse. The multi-stage pattern puts each arch on its own clean FROM line that Renovate recognizes as a standard Docker image reference.

---

## Device Type to Architecture Mapping

Common device types and their `%%BALENA_ARCH%%` values:

### aarch64

`raspberrypi3-64`, `raspberrypi4-64`, `raspberrypi5`, `jetson-nano`, `jetson-xavier-nx-devkit`, `genericaarch64`, `fincm3`, `nanopi-neo-air`, `coral-dev`, `imx8m-var-dart`, ...

### amd64

`genericx86-64-ext`, `intel-nuc`, `generic-amd64`, `surface-go`, `surface-pro-6`, `up-board`, ...

### armv7hf

`raspberry-pi2`, `raspberrypi3`, `beaglebone-black`, `asus-tinker-board`, `ts4900`, `apalis-imx6q`, ...

### rpi

`raspberry-pi` (Pi 1 / Zero, armv6)

### i386

`intel-edison`, `qemux86`

Full list available via the balena API (note: API version may change):

```text
https://api.balena-cloud.com/v7/device_type?$select=slug&$expand=is_of__cpu_architecture($select=slug)
```
