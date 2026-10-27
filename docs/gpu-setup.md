# Nvidia GPU Monitoring in Beszel Agent

## The Problem

The `henrygd/beszel-agent` Docker image is **distroless** — it contains no shell, no C runtime library (`glibc`), and no `/lib64` directory. This is intentional upstream (minimal attack surface), but it means Nvidia GPU monitoring requires injecting the host's dynamic linker and shared libraries into the container.

When Beszel calls `/usr/bin/nvidia-smi` to read GPU stats, the Linux kernel loads the ELF binary and looks for its dynamic linker at the path embedded in the binary (`/lib64/ld-linux-x86-64.so.2`). In a distroless container, that path doesn't exist, and the call fails — even if `nvidia-smi` itself was bind-mounted correctly.

## Error Progression

If GPU monitoring isn't working, check `docker logs beszel-agent` (or `beszel-agent-gpu`) and match the error:

| Error | Root Cause | Fix |
|-------|-----------|-----|
| `fork/exec /usr/bin/nvidia-smi: no such file or directory` | `--gpus all` missing from Extra Parameters, or nvidia-smi not bind-mounted | Use the GPU template; verify both `--gpus all` and the nvidia-smi path mount are present |
| `exec /usr/bin/nvidia-smi: no such file or directory` | nvidia-smi is mounted but the ELF dynamic linker (`/lib64/ld-linux-x86-64.so.2`) is missing | Add the `ld-linker` bind-mount — host path = real file from `readlink -f`, container path = `.so.2` |
| `nvidia-smi failed, restarting err="exit status 127"` | Linker is present but glibc runtime libraries are missing | Add all glibc lib mounts (libpthread, libm, libdl, libc, librt) and the three libnvidia-ml mounts |
| GPU visible in `nvidia-smi` on host but not in Beszel UI | Container was recreated from a stale template that lost the mounts | Open Docker Manager, verify all 11 bind-mounts are present, recreate the container if not |

## Discovery Commands

Run these **on the host** before deploying the GPU template, or after a driver upgrade:

```bash
# Get current driver version
nvidia-smi --query-gpu=driver_version --format=csv,noheader

# Get the REAL path of the ELF linker (it's a symlink on Unraid)
readlink -f /lib64/ld-linux-x86-64.so.2

# List all shared libraries nvidia-smi needs
ldd /usr/bin/nvidia-smi
```

## After a Driver Upgrade

When Unraid's Nvidia driver is updated, the versioned library path changes. Two template entries need updating:

1. **ld-linker** — the host path may change if glibc was also updated. Run `readlink -f /lib64/ld-linux-x86-64.so.2` and update the host path.
2. **libnvidia-ml (versioned)** — both the host path and container path contain the driver version (e.g. `libnvidia-ml.so.615.78.08`). Run `nvidia-smi --query-gpu=driver_version --format=csv,noheader`, then update both paths in Docker Manager.

After updating the paths, stop and recreate the container.

## Bind-Mount Reference

The GPU template includes these 10 bind-mounts (in addition to the standard agent mounts):

| Name | Host Path | Container Path | Notes |
|------|-----------|----------------|-------|
| nvidia-smi | `/usr/bin/nvidia-smi` | `/usr/bin/nvidia-smi` | The GPU query binary |
| ld-linker | `/lib64/ld-linux-x86-64-X.XX.so` | `/lib64/ld-linux-x86-64.so.2` | Host = real file, container = symlink path the ELF binary expects |
| libpthread | `/lib64/libpthread.so.0` | `/lib64/libpthread.so.0` | Standard on Unraid |
| libm | `/lib64/libm.so.6` | `/lib64/libm.so.6` | Standard on Unraid |
| libdl | `/lib64/libdl.so.2` | `/lib64/libdl.so.2` | Standard on Unraid |
| libc | `/lib64/libc.so.6` | `/lib64/libc.so.6` | Standard on Unraid |
| librt | `/lib64/librt.so.1` | `/lib64/librt.so.1` | Standard on Unraid |
| libnvidia-ml (versioned) | `/usr/lib/libnvidia-ml.so.<VER>` | `/usr/lib/libnvidia-ml.so.<VER>` | Changes with driver version |
| libnvidia-ml.so.1 | `/usr/lib/libnvidia-ml.so.1` | `/usr/lib/libnvidia-ml.so.1` | Soname symlink |
| libnvidia-ml.so | `/usr/lib/libnvidia-ml.so` | `/usr/lib/libnvidia-ml.so` | Dev symlink |
