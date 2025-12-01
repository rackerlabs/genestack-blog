---
title: Streamlining Node Access with K9s and kubectl-node-shell
date: 2025-11-19
authors:
  - japerezjr
slug: k9s-node-shell-integration
description: Learn how to integrate kubectl-node-shell with K9s for seamless node access and debugging in Kubernetes clusters
categories:
  - Kubernetes
  - DevOps
  - Tooling
---

# Streamlining Node Access with K9s and kubectl-node-shell

Debugging Kubernetes clusters often requires direct access to nodes. There are several ways to access your nodes... ssh, iLO/DRAC, `kubectl debug`, etc. I love shortcuts, aliases, functions, and scripts that can help me quickly gather data and help with my troubleshooting.  I have found K9s, a powerful terminal UI for Kubernetes, and how to enhance it with kubectl-node-shell for seamless node access.  This quick blog will hopefully give you another tool you can use with your kubernetes clusters.

<!-- more -->

## What is K9s?

[K9s](https://k9scli.io/) is a terminal-based UI for managing Kubernetes clusters. Quoting directly from their site:

> K9s is a terminal based UI to interact with your Kubernetes clusters. The aim of this project is to make it easier to navigate, observe and manage your deployed applications in the wild. K9s continually watches Kubernetes for changes and offers subsequent commands to interact with your observed resources.

K9s supports custom plugins, which makes it incredibly extensible.

## What is kubectl-node-shell?

[kubectl-node-shell](https://github.com/kvaps/kubectl-node-shell) is a kubectl plugin that opens an interactive shell on any node in your cluster. Unlike `kubectl debug`, it provides direct access to the node's root filesystem and processes, making it ideal for troubleshooting and system-level debugging.

### Installation

First, install kubectl-node-shell:

```bash
git clone https://github.com/kvaps/kubectl-node-shell.git
cd kubectl-node-shell
sudo cp kubectl-node_shell /usr/local/bin/
```

or using curl:

```bash
curl -LO https://github.com/kvaps/kubectl-node-shell/raw/master/kubectl-node_shell
chmod +x ./kubectl-node_shell
sudo mv ./kubectl-node_shell /usr/local/bin/kubectl-node_shell
```

Verify the installation:

```bash
kubectl node-shell --help
```

I encourage you to look at the other options possible with kubectl-node-shell.  There is even support for Talos

## Integrating with K9s

K9s allows you to define custom commands and plugins through a `plugins.yaml` configuration file. This enables you to launch kubectl-node-shell directly from K9s without leaving the UI.

### Step 1: Locate K9s Configuration

K9s stores its configuration in:

- **macOS/Linux**: `~/.config/k9s/`
- **Windows**: `%APPDATA%\k9s\`

### Step 2: Create or Edit plugins.yaml

Create or edit `~/.config/k9s/plugins.yaml`:

```yaml
plugins:
  node-shell:
    shortCut: Shift-S
    description: Node Shell
    scopes:
    - node
    command: kubectl
    background: false
    confirm: false
    args:
    - node-shell
    - $NAME
```

### Step 3: Restart K9s

Exit and restart K9s for the plugin to load.

## Using the Integration

1. Launch K9s: `k9s`
2. Navigate to the Nodes view (press `:nodes` or use the menu)
3. Select a node
4. Press `Shift-S` to open a shell on that node
5. You now have direct access to the node's filesystem and processes


## Common Use Cases

### Debugging Network Issues

```bash
# Inside the node shell
ip addr show
netstat -tulpn
iptables -L -n
```

### Checking Disk Space

```bash
df -h
du -sh /var/lib/kubelet/*
```

### Inspecting Container Runtime

```bash
# For containerd
ctr containers list
ctr images list

# For Docker
docker ps
docker images

# For crictl
crictl pods
crictl images
crictl ps
```

### Viewing System Logs

```bash
journalctl -u kubelet -n 50 -f
dmesg | tail -20
```

## Tips and Best Practices

- **Use read-only access when possible**: If you only need to view logs or check status, consider using `kubectl debug` with read-only access first
- **Document your findings**: When debugging issues, keep notes of what you discover for future reference
- **Clean up after yourself**: Exit node shells properly and don't leave processes running
- **Use K9s shortcuts**: Learn K9s keyboard shortcuts to navigate faster (`:help` shows all available commands)
- **Combine with other tools**: Use tools like `htop`, `iotop`, and `tcpdump` within the node shell for deeper analysis

## Troubleshooting

**Plugin not appearing in K9s:**
- Ensure `plugins.yaml` is in the correct directory
- Check the YAML syntax (use a YAML validator if needed)
- Restart K9s completely

**kubectl node-shell command not found:**
- Verify the plugin is in your PATH: `which kubectl-node_shell`
- Ensure it has execute permissions: `chmod +x /usr/local/bin/kubectl-node_shell`

**Permission denied errors:**
- You need cluster admin privileges to access nodes
- Check your kubeconfig and current context

## Comparing Your Options

When you need node access in Kubernetes, you have three main approaches other than physical access, iLO/DRAC, or ssh. Let's compare them:

### kubectl debug

**Pros:**
- Built into kubectl, no additional installation required
- Works on any cluster with the debug API enabled
- Creates temporary debug containers, leaving the node untouched
- Supports read-only access for safer exploration
- Good for RBAC-restricted environments (can be limited to specific namespaces)

**Cons:**
- Slower startup time (creates a pod, then a container)
- Limited to the container's filesystem and capabilities
- Requires the node to be running and schedulable
- Less direct access to node-level processes and system state
- More verbose command syntax

### K9s with K9S_FEATURE_GATE_NODE_SHELL=true

**Pros:**
- Built into K9s, no external plugins needed
- Integrated directly into the K9s UI
- Fast and convenient within the terminal UI
- No additional dependencies to manage

**Cons:**
- Requires setting an environment variable before launching K9s
- Less customizable than the plugin approach
- May have limitations depending on K9s version
- Still requires kubectl node-shell or similar backend to function
- Feature gate may change or be removed in future K9s versions

### kubectl-node-shell with K9s Plugin

**Pros:**
- Direct access to the node's root filesystem and all processes
- Fastest execution (direct shell, no container overhead)
- Full system-level debugging capabilities
- Customizable through plugins.yaml for different workflows
- Can chain multiple commands (journalctl, top, tcpdump, etc.)
- Persistent across K9s sessions once configured

**Cons:**
- Requires additional installation and setup
- Needs cluster admin privileges
- Direct node access is a security consideration
- Plugin configuration requires manual YAML editing
- Less portable if you frequently switch clusters

## Recommendation

- **Use `kubectl debug`** for general troubleshooting in shared clusters or when you need RBAC compliance
- **Use K9s Node Shell feature gate** if you want built-in node access without extra setup
- **Use kubectl-node-shell plugin** when you need full system access and are working in environments where you have admin privileges

For most users working with their own clusters or in DevOps roles, the kubectl-node-shell plugin integration offers the best balance of speed, functionality, and convenience.

## Conclusion

Hopefully you will find that combining K9s with kubectl-node-shell creates a powerful workflow for Kubernetes cluster management and debugging. Personally, I perfer to use k9s vs kubectl and this integration has helped me reduce context switching and makes node access as simple as pressing a keyboard shortcut. Whether you're troubleshooting network issues, checking disk space, or inspecting system logs, this setup will save you time and improve your debugging efficiency.

Choose the approach that best fits your security requirements and workflow. Start using K9s with node-shell integration today and streamline your Kubernetes operations.
