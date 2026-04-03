# Fedora bootc with NVIDIA drivers and tools

**STATUS:** **Work In Progress** — This image is in active development and is not yet ready for production use.

We are deriving Fedora bootc images to integrate NVIDIA components (kernel modules, drivers, etc).
This specialized image is hosted on Quay.io at [quay.io/coreos-devel/fedora-bootc-nvidia](https://quay.io/repository/coreos-devel/fedora-bootc-nvidia?tab=tags).

The build process utilizes official NVIDIA DKMS packages from [NVIDIA repo](https://developer.download.nvidia.com/compute/cuda/repos/) to
compile the open kernel modules, a method usually reserved for the client system.
Despite requiring a tweak to target the immutable image's specific kernel (rather than the build host's), using DKMS simplifies integration significantly.
However, this introduces the downside of breaking Secure Boot as the kmods are not signed by the Fedora's key but a self-signed one, which is ok for a POC. For permanent deployment, we'd need to maintain a custom signing key, which adds the extra step of enrolling it with MOK to regain secure booting functionality.

## Building locally
```bash
source build-args.conf
podman build --build-arg-file build-args.conf -f Containerfile.builder -t $BUILDER_IMAGE
podman build --build-arg-file build-args.conf -f Containerfile -t localhost/fedora-bootc-nvidia
```

## To test it
```bash
[core@localhost ~]$ lsmod | grep nvidia
nvidia_drm            155648  0
nvidia_modeset       2248704  1 nvidia_drm
nvidia              15917056  1 nvidia_modeset
drm_ttm_helper         16384  1 nvidia_drm
video                  81920  1 nvidia_modeset

[core@localhost ~]$ lspci -nnk | grep -A2 NVIDIA
00:03.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [GeForce RTX 3080] [10de:2206] (rev a1)
	Subsystem: NVIDIA Corporation GA102 [GeForce RTX 3080] [10de:1467]
	Kernel driver in use: nvidia
	Kernel modules: nouveau, nvidia_drm, nvidia

# Checking the licenses
[core@localhost ~]$ modinfo -l nvidia nvidia_drm nvidia_modeset
Dual MIT/GPL
Dual MIT/GPL
Dual MIT/GPL
```

## Serve a LLM with RamaLama

Boot an existing Image Mode system, for example Fedora CoreOS:
Create a Containerfile to layer the the NVIDIA CUDA driver and libs:
```
cat > Containerfile << 'EOF'
ARG STREAM=43
ARG VERSION=595.58.03

FROM quay.io/coreos-devel/fedora-bootc-nvidia:${STREAM}-${VERSION}

RUN <<EORUN
set -xeuo pipefail
source /usr/lib/os-release
curl https://developer.download.nvidia.com/compute/cuda/repos/${ID}${VERSION_ID}/$(arch)/cuda-${ID}${VERSION_ID}.repo \
     -o /etc/yum.repos.d/cuda.repo
dnf install -y \
    nvidia-driver-cuda
dnf clean all
systemctl enable nvidia-persistenced nvidia-cdi-refresh
EORUN

RUN bootc container lint

EOF
```
Build that container locally

```
sudo podman build -t fedora-bootc-nvidia .
```

Then switch the system to that build

```
sudo bootc switch --transport containers-storage localhost/fedora-bootc-nvidia --apply
```

Check if everything is ok:
```
nvidia-smi
```

And finally, serve the requested LLM with RamaLama
```
ramalama --engine podman serve -p 8081 --oci-runtime crun --image quay.io/ramalama/cuda:0.12.4 mistral:7b-v3
```
