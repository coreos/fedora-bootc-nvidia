---
name: update-fedora-bootc-nvidia
description: Guide to update the fedora-bootc-nvidia image to a new Fedora version and/or NVIDIA driver
license: MIT
compatibility: opencode
metadata:
  category: devops
  workflow: container-build
---

## Objective

Update the fedora-bootc-nvidia image by checking available versions on the NVIDIA repository and modifying the appropriate configuration files.

## Step 1: Detect available updates

### List Fedora versions supported by NVIDIA
```bash
curl -s "https://developer.download.nvidia.com/compute/cuda/repos/" | grep -oP 'fedora\d+' | sort -t 'a' -k2 -n | uniq
```

### Find the latest driver version for a given Fedora version
```bash
STREAM=43  # Replace with target version
curl -s "https://developer.download.nvidia.com/compute/cuda/repos/fedora${STREAM}/x86_64/" | grep -oP 'kmod-nvidia-open-dkms-\K[0-9.]+(?=-1)' | sort -V | tail -1
```

### Decision logic
1. If a new Fedora version appears in the NVIDIA repo → bump both STREAM and DRIVER_VERSION
2. If only a new driver is available for the current STREAM → bump only DRIVER_VERSION

## Step 2: Modify files

### 2.1 build-args.conf
Update `STREAM` and `DRIVER_VERSION`:
```
STREAM=<new_version>
DRIVER_VERSION=<new_driver>
```

### 2.2 .github/workflows/build-image.yml
Update the matrix (lines 32-33):
```yaml
matrix:
  STREAM: [<new_version>]
  NVIDIA_DRIVER_VERSION: [<new_driver>]
```

### 2.3 README.md
Update the ARG values in the Containerfile example (section "Serve a LLM with RamaLama"):
```dockerfile
ARG STREAM=<new_version>
ARG VERSION=<new_driver>
```

## Step 3: Verifications

### Validate YAML syntax
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/build-image.yml'))"
```

### Local test build
```bash
source build-args.conf
podman build --build-arg-file build-args.conf -f Containerfile.builder -t $BUILDER_IMAGE
podman build --build-arg-file build-args.conf -f Containerfile -t localhost/fedora-bootc-nvidia
```

## Step 4: Commit

Recommended commit message format:
```
Bump to Fedora <STREAM> with NVIDIA driver <DRIVER_VERSION>
```

Or if only the driver:
```
Update NVIDIA driver to <DRIVER_VERSION>
```
