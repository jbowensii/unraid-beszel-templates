# Beszel Unraid Templates

[Unraid Community Applications](https://unraid.net/community/apps) templates for [Beszel](https://github.com/henrygd/beszel) — a lightweight, self-hosted server monitoring solution.

## Templates Included

| Template | Use Case |
|----------|----------|
| **Beszel Hub** | Web UI and backend. Install once on one server. |
| **Beszel Agent (No GPU)** | Monitoring agent for servers without an Nvidia GPU. |
| **Beszel Agent (Nvidia GPU)** | Monitoring agent with full Nvidia GPU monitoring. Includes glibc bind-mounts for the distroless container image. |

## Installation

1. In Unraid, go to **Apps** (Community Applications plugin required).
2. Click the **Settings** icon (gear) at the top right.
3. Under **Template Repositories**, add this URL:

   ```
   https://github.com/jbowensii/unraid-beszel-templates
   ```

4. Click **Save**, then **Force Update** to pull the templates.
5. Search for "Beszel" in the Apps tab.

## Setup

### Hub

1. Install the **Beszel Hub** template on one server.
2. Open the web UI at `http://<server-ip>:8090`.
3. Create an account and go to **Settings → Keys** to get the SSH public key.

### Agents

1. On each server you want to monitor, install the appropriate agent template.
2. Paste the SSH public key from the hub into the **KEY** field.
3. Start the container.
4. In the hub UI, add the server using its IP address and port 45876 (the default agent port).

## Configurable Fields

All configuration is exposed through Unraid's Docker Manager UI when you install from the template. Key fields:

- **KEY** — SSH public key from the hub. Required on every agent.
- **SENSORS_EXCLUDE** — Comma-separated sensor names to suppress. Pre-filled with defaults that cover phantom sensors on Supermicro/ASUS server boards. Run `sensors -u` on the host to see your actual sensors and adjust.
- **Data / Agent Data paths** — Where persistent data is stored. Defaults to `/mnt/user/appdata/beszel-*`.
- **Web UI Port** (hub only) — Defaults to 8090. Change the host side if the port is taken.
- **GPU bind-mount paths** (GPU template only) — All 10 paths are individually editable. After a driver upgrade, update the `ld-linker` host path and the versioned `libnvidia-ml` paths. See [docs/gpu-setup.md](docs/gpu-setup.md).

## Nvidia GPU Monitoring

The `beszel-agent` image is distroless — no glibc, no dynamic linker. The GPU template injects these from the host via 10 read-only bind-mounts.

**Before deploying the GPU template**, run these on the host:

```bash
readlink -f /lib64/ld-linux-x86-64.so.2
nvidia-smi --query-gpu=driver_version --format=csv,noheader
```

Update the `ld-linker` and `libnvidia-ml (versioned)` paths in Docker Manager if they differ from the template defaults.

See [docs/gpu-setup.md](docs/gpu-setup.md) for the full error progression and troubleshooting guide.

## Phantom Sensors

Server motherboards commonly expose dozens of unwired sensor inputs that report garbage values (0°C, 127°C, or wildly fluctuating). The `SENSORS_EXCLUDE` default suppresses these:

- `AUXTIN0–5` — Unwired auxiliary thermal inputs
- `PCH_CHIP_CPU_MAX_TEMP`, `PCH_CPU_TEMP`, `PCH_MCH_TEMP` — Often absent or inaccurate
- `in0–in16` — Unconnected voltage rail inputs
- `intrusion0`, `intrusion1` — Chassis intrusion switches (stuck open on most homelab machines)

Adjust the list to match your hardware.

## Links

- [Beszel GitHub](https://github.com/henrygd/beszel)
- [Beszel Documentation](https://beszel.dev)
- [Unraid Community Applications](https://unraid.net/community/apps)
