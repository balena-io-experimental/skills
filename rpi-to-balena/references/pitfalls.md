# Common Pitfalls

Hard-won lessons from porting RPi hardware projects to balenaOS.

## 1. The Vendor Repo Doesn't Contain the Driver You Need

Vendors often maintain multiple drivers across different repos, or a single repo for a product family where the specific driver for your hardware is in the mainline RPi kernel — not in the vendor's repo.

**Verify before using any vendor source:**

- Check the `obj-m :=` line in the Makefile — does it build the module you need, or something else?
- Check pre-compiled `.ko` filenames in release directories.
- Check the README — does it mention your specific hardware, or a different product?
- Search for your driver name with `grep -r "your_driver" src/` — if zero results, wrong repo.

If the driver is already in the Raspberry Pi kernel tree (and therefore in the balenaOS kernel), you don't need to compile anything. Just use `modprobe` or set the dtoverlay.

## 2. Kernel Version Mismatch

balenaOS runs its own kernel, not the Raspberry Pi Foundation kernel. The version numbers are different. A script that downloads pre-compiled `.ko` files based on `uname -r` will fail because there is no match.

```
RPi OS kernel: 6.6.51-v8+
balenaOS kernel: 6.6.52         ← different numbering, different build
```

Pre-compiled `.ko` files for one kernel cannot be loaded into another. Even if the version numbers happen to match, the module vermagic string must match exactly.

**Solution:** Compile from source against balenaOS kernel headers using the kmod sidecar pattern, or use the stock in-tree module.

## 3. Module Naming: Hyphens vs Underscores

Linux treats hyphens (`-`) and underscores (`_`) as interchangeable in module names, but filenames and tools differ:

| Context                | Convention                      |
| ---------------------- | ------------------------------- |
| `obj-m := my-driver.o` | Hyphens in Makefile             |
| `my-driver.ko`         | Hyphens in filename             |
| `lsmod` output         | Always underscores: `my_driver` |
| `modprobe my_driver`   | Underscores                     |
| `modprobe my-driver`   | Also works (auto-converted)     |

When writing scripts that search for `.ko` files or check `lsmod`, handle both:

```bash
# lsmod always uses underscores
lsmod | grep -q "my_driver"

# .ko file may use either
for f in "my-driver.ko" "my_driver.ko"; do
    [ -f "/path/$f" ] && MODULE_FILE="/path/$f" && break
done
```

## 4. Silent Failures with `|| true`

Many vendor scripts use `|| true` or `2>/dev/null` to suppress errors. When porting to a Dockerfile, these hide real failures:

```dockerfile
# DANGEROUS — if the copy fails (e.g. source path doesn't exist), you get
# an empty image with no error message
RUN cp -r vendor-repo/src/my_driver/* /usr/src/driver/ 2>/dev/null || true
```

**When adapting scripts:** temporarily remove all error suppression to see what actually fails. Then handle each failure explicitly.

## 5. Runtime-Only Tools Called at Build Time

These tools require a running system with hardware and **cannot work during `docker build`**:

- `dtparam` — modifies device tree at runtime
- `vcgencmd` — VideoCore GPU commands
- `raspi-config` — interactive or non-interactive system config
- `i2cdetect` — probes I2C bus for connected devices
- `modprobe` / `insmod` — loads kernel modules (no kernel in Docker build)

**Pattern:** Find the conditional checks that gate these calls and stub them:

```dockerfile
# The script checks for /dev/i2c-0 and calls dtparam+modprobe when missing
RUN mkdir -p /dev && touch /dev/i2c-0
# Now the script skips that branch
RUN ./vendor-install.sh -p library_package
```

## 6. Forgetting to Document Dashboard Variables

The most common user-facing failure: deploying a project and finding that hardware doesn't work because required `BALENA_HOST_CONFIG_*` variables were never set.

Unlike `config.txt` changes committed to a repo, dashboard variables must be set manually by each user. Always:

1. Document them in the README with a clear table.
2. Log a helpful message from `entry.sh` when the hardware isn't detected, telling the user what to set.
3. Include them in a Quick Start section, not buried at the bottom.

```bash
# In entry.sh — helpful error when config is missing
if ! lsmod | grep -q "my_driver"; then
    echo "ERROR: my_driver module not loaded."
    echo ""
    echo "Set BALENA_HOST_CONFIG_dtoverlay = 'my-device' in the dashboard"
    echo "then reboot the device."
fi
```

## 7. Platform Detection Returns Unexpected Values

| Source                          | On RPi OS                    | In container on balenaOS                |
| ------------------------------- | ---------------------------- | --------------------------------------- |
| `/proc/cpuinfo` Revision        | `c03115`                     | Same — passed from host                 |
| `uname -r`                      | `6.6.51-v8+`                 | `6.6.52` — balenaOS kernel              |
| `/etc/os-release`               | `Raspberry Pi OS (bookworm)` | Container base image (e.g. `Debian 12`) |
| `dpkg-query raspberrypi-kernel` | RPi kernel version           | **Fails** — not installed               |
| `BALENA_HOST_OS_VERSION`        | N/A                          | `balenaOS 2025.4.0`                     |

Scripts that branch on `uname -r` or `dpkg-query` will take wrong paths or fail. Understand which branches the script takes and set environment variables or stubs to steer it correctly.

## 8. Using Vendor Pre-compiled Binaries in the Wrong Context

Vendor repos often contain a `Release/bin/` directory with pre-compiled `.ko` files for various RPi kernel versions. These are:

- Compiled against the RPi Foundation kernel, not the balenaOS kernel.
- Often only for 32-bit ARM (`armv7l`), not `aarch64`.
- Version-locked to specific `uname -r` strings that don't exist on balenaOS.

**Never attempt to `insmod` a pre-compiled `.ko` from a vendor repo on balenaOS.** It will fail with `Invalid module format` or a version mismatch error. Always compile from source.
