---
date: 2025-12-01
title: Getting Started with GPU Compute on Rackspace OpenStack Flex
authors:
  - cloudnull
description: >
  Getting Started with GPU Compute on Rackspace OpenStack Flex
categories:
  - OpenStack
  - GPU
  - NVIDIA
  - Compute
  - Instance
  - Setup

---

# Getting Started with GPU Compute on Rackspace OpenStack Flex

**Your instance is up, your GPU is attached, and now you're staring at a blank terminal wondering what's next. Time to get Clouding.**

Rackspace OpenStack Flex delivers GPU-enabled compute instances with real hardware acceleration, not the "GPU-adjacent" experience you might get elsewhere. Whether you're running inference workloads, training models, or just need raw parallel compute power, getting your instance properly configured is the difference between frustration and actual productivity.

<!-- more -->

This guide walks you through the complete setup process: from fresh Ubuntu 24.04 instance to a fully optimized NVIDIA GPU environment ready for production workloads.

## Prerequisites

Before diving in, make sure you have.

- A Rackspace OpenStack Flex instance with an attached GPU (H100, A30, or similar)
- Ubuntu 24.04 LTS as your base image
- SSH access to your instance
- Root or sudo privileges

If you're provisioning a new instance, select a GPU-enabled flavor from the OpenStack Flex catalog. The platform's SR-IOV architecture provides direct hardware access to your GPU resources, delivering near-native performance without the virtualization overhead that plagues other cloud GPU offerings.

## Step 1: Update Your System

First things first, get your system current. This isn't just good hygiene; NVIDIA drivers have specific kernel dependencies, and starting with an outdated kernel is a recipe for cryptic error messages at 2 AM.

SSH into your instance and run.

```bash
sudo apt update
sudo apt upgrade -y
```

## Step 2: Configure GRUB for PCI Passthrough

This is the step that separates the tutorials that work from the ones that leave you debugging kernel panics. When GPUs are passed through to VMs via SR-IOV or PCI passthrough, the kernel's default PCI resource allocation can conflict with how the hypervisor has already mapped the device.

The fix is telling the kernel to be more flexible about PCI resource assignment. Edit your GRUB configuration to add the following parameters.

```bash
sudo sed -i 's/^GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 pci=realloc,assign-busses,nocrs"/' /etc/default/grub
```

Let's break down what these parameters do.

- **pci=realloc** ,  Allows the kernel to reallocate PCI resources if the BIOS allocation doesn't match what's needed
- **assign-busses** ,  Forces the kernel to assign PCI bus numbers rather than trusting BIOS assignments
- **nocrs** ,  Ignores PCI _CRS (Current Resource Settings) from ACPI, preventing conflicts with hypervisor mappings

Now regenerate your GRUB configuration and initramfs.

```bash
sudo update-grub2
sudo update-initramfs -u
```

Reboot to load the new kernel parameters and drivers.

```bash
sudo reboot
```

## Step 3: Install NVIDIA Drivers

Wait a minute or two for the instance to come back up, then SSH back in. You can verify your kernel version with `uname -r` to confirm the upgrade took effect.

=== "Official NVIDIA Drivers"

    Here's where things get real. NVIDIA provides drivers specifically optimized for datacenter GPUs like the H100 and A30. We're using the 580.x branch, which is the current production release for these cards. However, always check [NVIDIA's official driver page](https://www.nvidia.com/Download/index.aspx) for the latest recommendations.

    Download the driver package.

    ```bash
    wget https://us.download.nvidia.com/tesla/580.105.08/nvidia-driver-local-repo-ubuntu2404-580.105.08_1.0-1_amd64.deb
    ```

    Install the local repository package.

    ```bash
    sudo dpkg -i nvidia-driver-local-repo-ubuntu2404-580.105.08_1.0-1_amd64.deb
    ```

    Copy the GPG keyring to your system's trusted keys.

    ```bash
    sudo cp /var/nvidia-driver-local-repo-ubuntu2404-580.105.08/nvidia-driver-local-207F658F-keyring.gpg /usr/share/keyrings/
    ```

    Update your package lists and install the CUDA drivers.

    ```bash
    sudo apt update
    sudo apt install -y cuda-drivers-580
    ```

    !!! note "Alternative: Open-Source NVIDIA Drivers"

        If you prefer the open-source kernel modules (which NVIDIA now provides for datacenter GPUs), you can install those instead:

        ```bash
        sudo apt install -y nvidia-open-580
        ```

        The open drivers provide identical functionality for compute workloads and are fully supported on H100/A30 hardware. The choice between proprietary and open modules is largely philosophical at this point, both work.

=== "Distro Drivers"

    Ubuntu includes NVIDIA drivers in its default repositories. While these may not always be the absolute latest, they are tested and integrated with the distro's kernel.

    Install the driver package common package.

    ```bash
    sudo apt install -y ubuntu-drivers-common
    ```

    Use the `ubuntu-drivers` tool to automatically identify and install the recommended NVIDIA driver for your GPU.

    ```bash
    sudo ubuntu-drivers autoinstall
    ```

    !!! note "Alternative: Open-Source NVIDIA Drivers"

        If you want to install a specific version of the open-source NVIDIA drivers, you can do so with:

        ```bash
        sudo ubuntu-drivers list --gpgpu
        ```

        Then install the desired version, for example:

        ```bash
        sudo ubuntu-drivers install --gpgpu nvidia:580-server
        ```

    Refer to the [Ubuntu documentation](https://documentation.ubuntu.com/server/how-to/graphics/install-nvidia-drivers/) for more details.

Once again, reboot to load the NVIDIA drivers.

```bash
sudo reboot
```

## Step 4: Verify Your Installation

After the reboot, SSH back in and run the moment of truth.

```bash
sudo nvidia-smi
```

You should see output showing your GPU(s), their temperature, memory usage, and driver version. Something like.

```shell
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.105.08             Driver Version: 580.105.08     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H100 NVL                On  |   00000000:06:00.0 Off |                  Off |
| N/A   33C    P0             64W /  400W |       0MiB /  95830MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA H100 NVL                On  |   00000000:07:00.0 Off |                  Off |
| N/A   32C    P0             62W /  400W |       0MiB /  95830MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA H100 NVL                On  |   00000000:08:00.0 Off |                  Off |
| N/A   35C    P0             65W /  400W |       0MiB /  95830MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA H100 NVL                On  |   00000000:09:00.0 Off |                  Off |
| N/A   32C    P0             61W /  400W |       0MiB /  95830MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

If you see your GPU listed with valid stats, congratulations, you have working GPU compute. If you see errors about the driver not being loaded, double-check that your GRUB parameters were applied correctly with `cat /proc/cmdline`.

## Step 5: Optimize for Production Workloads (Optional)

Now that your GPU is working, let's tune it for actual compute work. These optimizations reduce latency, improve throughput, and ensure consistent performance under load. All of the following steps are optional, so there's nothing that you need to do from this point forward, however, the following options can be used to enhance your GPU performance, depending on your workload.

### Enable Persistence Mode

By default, the NVIDIA driver unloads when no applications are using the GPU. This saves power but adds several seconds of initialization latency when you start a new workload. Persistence mode keeps the driver loaded.

```bash
sudo nvidia-smi -pm 1
```

### Set Compute-Exclusive Mode

If you're running a single application that needs full GPU access (most ML training and inference scenarios), exclusive process mode ensures no other processes can claim GPU resources:

```bash
sudo nvidia-smi -c EXCLUSIVE_PROCESS
```

### Lock GPU Clocks

NVIDIA GPUs dynamically scale their clock frequencies based on load, temperature, and power limits. For benchmarking or latency-sensitive workloads, you may want consistent performance. Lock the graphics clock to maximum:

!!! note

    The following values are specific for H100 GPUs. Check your specific GPU's capabilities with `nvidia-smi -q -d SUPPORTED_CLOCKS`.

    !!! example "The output will look similar to this"

        ```bash
        Timestamp                                 : Thu Dec  1 23:00:53 2025
        Driver Version                            : 580.105.08
        CUDA Version                              : 13.0

        Attached GPUs                             : 4
        GPU 00000000:06:00.0
            Supported Clocks
                Memory                            : 2619 MHz
                    Graphics                      : 1980 MHz
                    ...
        ```

With the memory and graphics clock values identified, lock the graphics clock.

```bash
sudo nvidia-smi -lgc 1980,1980  # H100 max graphics clock
```

Lock the memory clock.

```bash
sudo nvidia-smi -lmc 2619,2619  # H100 HBM3 max
```

### Disable ECC Memory

H100 and A30 GPUs have Error Correcting Code (ECC) memory enabled by default, which protects against bit flips but reduces available memory bandwidth by 5-10%. For workloads where you need maximum throughput and can tolerate occasional errors (training jobs with checkpointing, inference where you can retry failures), you can disable ECC.

```bash
sudo nvidia-smi -e 0
```

!!! Warning

    This requires a reboot to take effect, and you should carefully evaluate your data integrity requirements before disabling ECC. For financial, scientific, or medical applications, keep ECC enabled.

### Making Settings Persistent

The `nvidia-smi` settings above won't survive a reboot. To make them persistent, create a systemd service at `/etc/systemd/system/nvidia-persistenced.service`

```ini
[Unit]
Description=NVIDIA Persistence Daemon
After=network.target

[Service]
Type=forking
ExecStart=/usr/bin/nvidia-persistenced --user root
ExecStopPost=/bin/rm -rf /var/run/nvidia-persistenced

[Install]
WantedBy=multi-user.target
```

Enable and start the service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable nvidia-persistenced
sudo systemctl start nvidia-persistenced
```

For the clock settings and compute mode, create a systemd service that runs after the persistence daemon at `/etc/systemd/system/nvidia-gpu-optimize.service`.

```ini
[Unit]
Description=NVIDIA GPU Optimization Settings
After=nvidia-persistenced.service
Requires=nvidia-persistenced.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/nvidia-smi -c EXCLUSIVE_PROCESS
ExecStart=/usr/bin/nvidia-smi -lgc 1980,1980  # NOTE This value may differ based on GPU model
ExecStart=/usr/bin/nvidia-smi -lmc 2619,2619  # NOTE This value may differ based on GPU model

[Install]
WantedBy=multi-user.target
```

Enable the service.

```bash
sudo systemctl daemon-reload
sudo systemctl enable nvidia-gpu-optimize
```

The service will apply your optimization settings automatically on every boot, after the persistence daemon has initialized the GPU. If you've changed any be sure to once again reboot your instance for the changes to take effect, and validate your settings.

---

## What's Next?

With your GPU environment configured, you're ready to install your actual workload tooling.

- **CUDA Toolkit** ,  For compiling CUDA applications: `sudo apt install cuda-toolkit-12-8`
- **cuDNN** ,  For deep learning frameworks: available from NVIDIA's developer portal
- **PyTorch/TensorFlow** ,  Most ML frameworks auto-detect and use CUDA when available
- **Container runtimes** ,  NVIDIA Container Toolkit for Docker/Podman GPU passthrough

The OpenStack Flex platform's accelerator-optimized architecture means your GPU workloads get direct hardware access without the overhead you'd experience on hyperscaler clouds. Combined with Rackspace's flexible pricing, you can run GPU compute at a fraction of public cloud costs while maintaining the control and performance characteristics of dedicated infrastructure.

---

*Running into issues? The most common problems are GRUB parameters not being applied (check `/proc/cmdline`), kernel module conflicts with nouveau (blacklist it in `/etc/modprobe.d/`), or mismatched driver versions. The NVIDIA forums and Rackspace support are both excellent resources when you hit the weird edge cases.*

For more information about GPU-enabled compute on Rackspace OpenStack Flex, see the [Genestack documentation](https://docs.rackspacecloud.com/) or contact your Rackspace account team.

Are you ready to begin clouding? [Sign-up](https://cart.rackspace.com/en/cloud/openstack-flex/account) for Rackspace OpenStack today.
