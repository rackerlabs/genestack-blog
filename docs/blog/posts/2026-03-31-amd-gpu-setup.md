---
date: 2026-03-31
title: "Getting Started with AMD GPU Compute on Rackspace OpenStack Flex"
authors:
  - cloudnull
description: >-
  From fresh Enterprise Linux instance to working ROCm environment on an AMD Radeon AI PRO R9700 the manual way, the cloud-init way, and everything in between.
categories:
  - OpenStack
  - GPU
  - AMD
  - Compute
  - Instance
  - Setup
---

# Getting Started with AMD GPU Compute on Rackspace OpenStack Flex

**Your instance is up, your AMD GPU is attached, and you're staring at a terminal with no `nvidia-smi` to lean on. Welcome to the other side.**

If you've read our [NVIDIA getting started guide](https://blog.rackspacecloud.com/blog/2025/12/01/getting_started_with_gpu_compute_on_rackspace_openstack_flex/), you know the drill: provision an instance, install drivers, verify the hardware, start computing. The AMD path follows the same logic but with different tooling. Instead of CUDA, you're working with ROCm. Instead of `nvidia-smi`, you've got `rocm-smi`. Instead of a driver ecosystem that's had two decades of cloud deployment polish, you've got one that's been moving fast and getting dramatically better, but still has some rough edges worth knowing about.

<!-- more -->

This guide walks you through setting up an AMD Radeon AI PRO R9700 on Rocky Linux 9, first step by step so you understand what each piece does, then as a cloud-init file you can fire and forget on your next instance launch. If you've worked through our [R9700 deployment post](https://blog.rackspacecloud.com/blog/2026/03/31/bringing_the_amd_radeon_ai_pro_r9700_online_in_openstack_flex/), you know the operator side; this is the tenant side of the same coin.

## Prerequisites

Before diving in, make sure you have the following in place.

An **AMD GPU-enabled instance** on Rackspace OpenStack Flex, currently the Radeon AI PRO R9700 with 32 GB GDDR6. Select one of the R9700 flavors from the catalog (such as `ao.9.24.96_R9700` for a single GPU or `ao.9.48.128_R9700-2` for dual GPUs).

A **Rocky Linux 9** base image. This guide is written for Rocky, but the process is nearly identical on AlmaLinux 9 or RHEL 9 since they share the same package ecosystem.

**SSH access** to your instance with root or sudo privileges.

A willingness to reboot once at the end, because GPU drivers and kernels are like that.

## Step 1: Update your system

Same as NVIDIA, same as any sane deployment. Get your system current before you start installing kernel modules. AMDGPU drivers compile via DKMS against your running kernel, so starting with a stale kernel is asking for a module mismatch that won't announce itself until you try to load the driver.

SSH into your instance and run:

```shell
sudo dnf update -y
```

On Rocky 9 this will pull in any pending kernel updates, security patches, and dependency refreshes. Don't skip this step. The ten minutes it takes now will save you an hour of debugging later.

## Step 2: Enable required repositories

AMD's driver and ROCm packages live in AMD's own repositories, not in the default Rocky repos. Before we can install anything, we need to get those repos configured and enable the CodeReady Builder (CRB) repository, which provides development headers and libraries that the AMDGPU DKMS module depends on.

First, install the AMDGPU installer package. This sets up the repository configuration for both the GPU driver and the ROCm stack:

```shell
sudo dnf install -y https://repo.radeon.com/amdgpu-install/7.2/el/9.7/amdgpu-install-7.2.70200-1.el9.noarch.rpm
```

Clean the DNF cache and enable CRB:

```shell
sudo dnf clean all
sudo dnf config-manager --set-enabled crb
```

Then install EPEL (Extra Packages for Enterprise Linux), which provides additional build dependencies:

```shell
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
```

At this point you have four active repository sources: the base Rocky repos, CRB, EPEL, and AMD's `repo.radeon.com`. That's the full dependency chain covered.

## Step 3: Install the AMDGPU driver and ROCm

This is where things diverge meaningfully from the NVIDIA workflow. AMD provides a unified installer, `amdgpu-install`, that handles both the kernel-level GPU driver and the ROCm userspace toolkit in a single command. No separate CUDA toolkit download, no GPG keyring dance, no choosing between proprietary and open-source modules (AMD's kernel driver is open source by default and has been upstream in the Linux kernel for years).

```shell
sudo amdgpu-install --usecase=rocm --no-32 -y
```

The `--usecase=rocm` flag tells the installer you want the full compute stack, not just the display driver. This pulls in the AMDGPU DKMS kernel module, the ROCm runtime, the HIP compiler infrastructure, and the OpenCL ICD. The `--no-32` flag skips 32-bit compatibility libraries, because if you're running AI inference on a cloud instance and need 32-bit GPU libraries, something has gone profoundly sideways in your architecture decisions.

This step takes a few minutes. The DKMS module compiles against your running kernel, which is why we updated the system first. When it finishes, the AMDGPU driver is installed but not yet loaded — we'll handle that with a reboot at the end.

## Step 4: Install container tooling

Most GPU workloads in 2026 run inside containers. Whether you're pulling a vLLM image, running a PyTorch training job, or deploying an inference API, you'll want a container runtime ready.

```shell
sudo dnf install -y podman podman-compose
```

We use Podman here rather than Docker because it's the default container runtime on Rocky Linux and doesn't require a daemon running as root. For GPU passthrough into containers, ROCm works with Podman via the `--device` flag — no separate "container toolkit" package required, unlike NVIDIA's approach. One less thing to install, one less thing to break.

## Step 5: Configure user permissions

By default, GPU device nodes (`/dev/kfd` and `/dev/dri/renderD*`) are owned by the `render` and `video` groups. Your user account needs membership in both groups to access the GPU without running everything as root.

```shell
sudo usermod -aG render,video rocky
```

!!! note

    If you're using a different base image, swap `rocky` for whatever your default user is. AlmaLinux uses `almalinux`, generic cloud images often use `cloud-user`:

    ```shell
    sudo usermod -aG render,video almalinux   # AlmaLinux
    sudo usermod -aG render,video cloud-user   # Generic cloud images
    ```

Group membership changes take effect on the next login, which the upcoming reboot will handle. If you're impatient, `newgrp render` will update your current session, but the reboot is coming anyway.

## Step 6: Reboot

The AMDGPU DKMS module needs a reboot to load into the running kernel. There's no way around this, the module was compiled during installation but isn't active until the next boot.

```shell
sudo reboot
```

Wait a minute, then SSH back in.

## Step 7: Verify your installation

The moment of truth. AMD's equivalent of `nvidia-smi` is `rocm-smi`:

```shell
rocm-smi
```

You should see output showing your R9700, its temperature, power draw, and memory usage. Something along the lines of:

```console
WARNING: AMD GPU device(s) is/are in a low-power state. Check power control/runtime_status

======================================= ROCm System Management Interface =======================================
================================================= Concise Info =================================================
Device  Node  IDs              Temp    Power  Partitions          SCLK  MCLK   Fan  Perf  PwrCap  VRAM%  GPU%
              (DID,     GUID)  (Edge)  (Avg)  (Mem, Compute, ID)
================================================================================================================
0       1     0x7551,   33194  27.0°C  1.0W   N/A, N/A, 0         0Mhz  96Mhz  0%   auto  300.0W  0%     0%
1       2     0x7551,   4459   32.0°C  1.0W   N/A, N/A, 0         0Mhz  96Mhz  0%   auto  300.0W  0%     0%
================================================================================================================
============================================= End of ROCm SMI Log ==============================================
```

!!! note

    There is also `amd-smi` which is a useful tool and provides output that is more similar to `nvidia-smi`, but it doesn't support all the features of the R9700 yet. For basic health checks and verifying the driver is talking to the hardware, `rocm-smi` is the more reliable choice.

    ```console
    +------------------------------------------------------------------------------+
    | AMD-SMI 26.2.1+fc0010cf6a    amdgpu version: 6.16.13  ROCm version: 7.2.0    |
    | VBIOS version: 00167619                                                      |
    | Platform: Linux Guest (Passthrough)                                          |
    |-------------------------------------+----------------------------------------|
    | BDF                        GPU-Name | Mem-Uti   Temp   UEC       Power-Usage |
    | GPU  HIP-ID  OAM-ID  Partition-Mode | GFX-Uti    Fan               Mem-Usage |
    |=====================================+========================================|
    | 0000:06:00.0 ...adeon AI PRO R9700S | 0 %      28 °C   0             1/300 W |
    |   0       0     N/A             N/A | 0 %        N/A             57/32624 MB |
    |-------------------------------------+----------------------------------------|
    | 0000:07:00.0 ...adeon AI PRO R9700S | 0 %      31 °C   0            13/300 W |
    |   1       1     N/A             N/A | 3 %        N/A             57/32624 MB |
    +-------------------------------------+----------------------------------------+
    +------------------------------------------------------------------------------+
    | Processes:                                                                   |
    |  GPU        PID  Process Name          GTT_MEM  VRAM_MEM  MEM_USAGE     CU % |
    |==============================================================================|
    |  No running processes found                                                  |
    +------------------------------------------------------------------------------+
    ```

If you see your GPU listed with valid statistics, congratulations, you have a working AMD compute environment on OpenStack Flex. If `rocm-smi` isn't found, check that `/opt/rocm/bin` is in your PATH (it should be after the reboot, via `/etc/profile.d/rocm.sh`).

For a more detailed hardware check.

```shell
rocminfo
```

This will dump the full agent information including the GPU's GFX target (`gfx1201` for the R9700), compute unit count, memory size, and supported ISA features. It's the "tell me everything about this GPU" command and it's useful for confirming that the ROCm runtime is talking to the hardware correctly.

You can also verify the kernel module is loaded:

```shell
lsmod | grep amdgpu
```

And check device permissions.

```shell
ls -la /dev/kfd /dev/dri/render*
```

Both should show `render` and `video` group ownership. If they don't, the DKMS module may not have loaded correctly — check `dmesg | grep amdgpu` for clues.

## The cloud-init way: skip all of the above

Everything in steps 1 through 6 can be automated with a cloud-init file passed at instance creation. If you've done this once manually and understand what each piece does, there's no reason to do it again. Save the following as `amdgpu-el97.yaml` and pass it as user data when creating your instance.

```yaml
#cloud-config
# AMDGPU + ROCm Cloud-Init for EL 9.7
#
# Provisions a GPU-enabled compute node with:
#   - AMDGPU driver with DKMS
#   - ROCm toolkit
#
# Usage:
#   openstack server create --user-data amdgpu-el97.yaml ...

# System Package Management
package_update: true
package_upgrade: true

# Install Base Dependencies
packages:
  - ca-certificates
  - curl
  - wget
  - dnf-utils

runcmd:
  # Install AMDGPU Installer and Clean
  - echo "[cloud-init] Installing AMDGPU driver installer..."
  - dnf install -y https://repo.radeon.com/amdgpu-install/7.2/el/9.7/amdgpu-install-7.2.70200-1.el9.noarch.rpm
  - dnf clean all
  - dnf config-manager --set-enabled crb

  # Install EPEL Repository
  - echo "[cloud-init] Installing EPEL repository..."
  - dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

  # Install AMDGPU Driver
  - echo "[cloud-init] Installing AMDGPU driver..."
  - amdgpu-install --usecase=rocm --no-32 -y

  # Configure Podman
  - echo "[cloud-init] Installing Podman..."
  - dnf install -y podman podman-compose

  # Configure User Permissions for GPU Access
  - echo "[cloud-init] Configuring user permissions..."
  - usermod -aG render,video rocky || true
  - usermod -aG render,video almalinux || true
  - usermod -aG render,video cloud-user || true

  # Log Completion
  - echo "[cloud-init] AMDGPU + ROCm node setup complete."

# Power State Configuration
power_state:
  delay: "+1"
  mode: reboot
  message: "Rebooting to apply AMDGPU driver and GRUB changes"
  timeout: 30
  condition: true
```

To launch an instance with this cloud-init configuration:

```shell
openstack server create \
          --flavor ao.9.24.96_R9700-2 \
          --image "Rocky-9" \
          --network your-network \
          --key-name your-keypair \
          --user-data amdgpu-el97.yaml \
          my-amd-gpu-instance
```

The instance will boot, update itself, install the AMDGPU driver and ROCm stack, configure permissions, and reboot, all unattended. Give it about ten to fifteen minutes for the full cycle (most of that is the `dnf update` and DKMS compilation). When you SSH in, `rocm-smi` should just work.

A few things worth noting about the cloud-init file. The `package_update: true` and `package_upgrade: true` directives handle our Step 1. The `packages` block installs base dependencies before the `runcmd` section fires. The `usermod` commands include `|| true` so they don't fail on images that use a different default username, whichever user exists gets the groups, and the rest silently no-op. The `power_state` block at the bottom triggers the mandatory reboot after everything is installed, with a one-minute delay to ensure all writes are flushed.

## What's next?

With your AMD GPU environment configured, you're ready to start deploying actual workloads. Here are the common next steps.

**ROCm-aware PyTorch** is probably the most common starting point. Install via pip with the ROCm index:

```shell
pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm7.2
```

**vLLM for inference serving** works on ROCm and treats it as a first-class platform. Pull the ROCm container image and you're running.

```shell
podman run --device /dev/kfd \
           --device /dev/dri \
           --group-add render \
           --group-add video \
           -v /path/to/models:/models \
           -p 8000:8000 \
           docker.io/rocm/vllm-dev:latest \
           --model /models/your-model \
           --host 0.0.0.0
```

!!! note

    At the time of this writing, on RDNA 4 hardware like the R9700, you'll want to set `VLLM_USE_TRITON_FLASH_ATTN=1` in the container environment because the Composable Kernel flash attention backend doesn't support RDNA architectures yet. The Triton backend works well for inference.

**llama.cpp** runs via the HIP backend for GGUF model inference. If you're doing local LLM work with quantized models, this is the lightweight path that doesn't require a full framework stack.

**TensorFlow** is available via `tensorflow-rocm` wheels for training and inference workloads.

Unlike the NVIDIA ecosystem, AMD's GPU driver is open source and upstream in the Linux kernel, which means you won't encounter licensing restrictions or proprietary container toolkits. Podman's `--device` flag is all you need to pass the GPU into a container. That simplicity is one of the genuine advantages of the AMD compute stack, fewer moving parts means fewer things to break at 3 AM.

!!! tip "Running into issues?"

    *The most common problems are: `rocm-smi` returning "No AMD GPUs" (check that the `amdgpu` kernel module loaded with `lsmod | grep amdgpu`), permission denied on `/dev/kfd` (your user isn't in the `render` group, log out and back in or `newgrp render`), or DKMS build failures during installation (usually a missing kernel headers package — `sudo dnf install kernel-devel-$(uname -r)` and re-run the installer). For RDNA 4 specific ROCm issues, the [AMD ROCm GitHub discussions](https://github.com/ROCm/ROCm/discussions) are an excellent resource.*

For more information about GPU-enabled compute on Rackspace OpenStack Flex, see the [Genestack documentation](https://docs.rackspacecloud.com/) or contact your Rackspace account team.

---

Are you ready to begin clouding? [Sign up](https://cart.rackspace.com/en/cloud/openstack-flex/account) for Rackspace OpenStack Flex today.