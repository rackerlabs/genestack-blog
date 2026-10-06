---
date: 2026-10-06
title: "When Hardware Offload Goes Silent: The Case of the Orphaned TC Flowers"
authors:
  - dwith
description: >-
  What happens when NICs lack hardware offload support and disabling it leaves dormant Linux TC flower rules in the kernel? Here is how Genestack engineers tracked down silent packet drops, hung Cinder services, and audited 121 production nodes.
categories:
  - openstack
  - networking
  - ovn
  - kubernetes
  - troubleshooting
  - ansible
---

# When Hardware Offload Goes Silent: The Case of the Orphaned TC Flowers

In large-scale cloud infrastructure, few things sound as compelling on paper as hardware offloading. When you are running Genestack—powering enterprise OpenStack clouds orchestrated on top of Kubernetes with Kube-OVN and Open vSwitch (OVS)—encapsulating Geneve overlay traffic and tracking connection state can consume measurable CPU cycles across a busy fleet. Offloading those flow tables and match-action rules directly into the physical NIC silicon is the holy grail: line-rate throughput, near-zero CPU overhead, and microsecond packet handling.

Until your network cards decide they cannot actually handle it.

<!-- more -->

In our production environments, enabling OVS hardware offloading (`other_config:hw-offload=true`) did not yield frictionless line-rate acceleration. Instead, the physical NICs in our fleet encountered firmware locks, driver crashes, and unrecoverable server panics. 

Naturally, we took the responsible engineering decision: turn off hardware offloading fleet-wide (`other_config:hw-offload=false`) and revert cleanly to the tried-and-true Linux kernel software datapath (`openvswitch.ko`). Node stability immediately returned, server crashes vanished, and we closed the maintenance window.

What we did not realize at the time was that we had inadvertently primed a silent time bomb across dozens of hosts. Weeks later, services would begin mysteriously hanging, TCP handshakes would drop into one-way black holes, and the culprit would turn out to be a ghost in the Linux kernel: **orphaned Traffic Control (TC) flower filters**.

Here is the story of how we uncovered the problem, why standard troubleshooting tooling failed to see it, and how we engineered an automated audit and remediation pipeline across 121 production nodes in [PR #1843](https://github.com/rackerlabs/genestack/pull/1843/).

---

## The Mystery: The Case of the Hanging Cinder Nodes

The trouble surfaced weeks after disabling hardware offload during routine operations and certificate rotations. On storage nodes like `1116512-blockstorage-prod`, OpenStack block storage services (`cinder-volume`, `cinder-volume-netapp`, and `cinder-backup`) reported as `active (running)` in `systemctl status`, yet they were completely dead in the water.

They logged no application output, generated no heartbeats, and were conspicuously absent from `openstack volume service list`:

```shell
overseer$ openstack volume service list --service cinder-volume | grep 1116512-blockstorage-prod
# (crickets — no output)
```

Inspecting the service journal revealed nothing beyond the initial systemd invocation. Even restarting the units failed to produce logs:

```text
cinder-volume.service: State 'stop-sigterm' timed out. Killing.
cinder-volume.service: Killing process 1720535 (cinder-volume) with signal SIGKILL.
cinder-volume.service: Consumed 2.001s CPU time, 123.8M memory peak
```

A process consuming only 2.0 seconds of CPU across its entire lifetime is a classic signature: it wasn't crashing or churning; it was hanging synchronously on an external I/O call during early boot.

Attaching `py-spy` to the stuck process showed the main thread blocked in Eventlet's polling hub:

```text
MainThread idle in eventlet hub (do_poll / wait / run)
  cinder/service.py:create -> Service.create()
```

In OpenStack Cinder, the very first network operation executed inside `Service.create()` is querying the MariaDB database to register or update its host service record. RabbitMQ hasn't even been touched yet.

We ran a quick TCP check from the host to the MariaDB ClusterIP VIP:

```shell
root@1116512-blockstorage-prod:~# nc -zv 10.233.43.30 3306
Connection to 10.233.43.30 3306 port [tcp/mysql] succeeded!
```

Wait. `nc -zv` succeeded! If TCP port 3306 was reachable, why did the MySQL client and Cinder hang indefinitely?

---

## Down the Rabbit Hole: The One-Way TCP Black Hole

When `nc -zv` says "succeeded", it only proves one thing: the initial TCP SYN was acknowledged by a SYN-ACK. It does **not** prove that application data or subsequent ACKs can pass through the network path.

We fired up the real MariaDB client from the broken node:

```shell
root@1116512-blockstorage-prod:~# mariadb -h 10.233.43.30 -u cinder -p
# Hangs indefinitely...
```

Meanwhile, from a peer storage node (`1116514-blockstorage-prod`), the exact same command connected instantly.

We checked the low-level TCP socket statistics with `ss -tnpi`:

```text
ESTAB 10.233.43.30:35976 -> 10.233.43.30:3306
  bytes_acked:1 segs_out:7 segs_in:6 rcvmss:536
```

MariaDB is a server-speaks-first protocol: once the TCP 3-way handshake finishes, the database server emits an initial greeting packet (e.g. `11.8.5-MariaDB-ubu2404`).

Look at those segment counters:
- `segs_in: 6`: The server sent 1 SYN-ACK, followed by 5 duplicate SYN-ACK retransmissions.
- `segs_out: 7`: The client sent 1 SYN, 1 ACK, and 5 duplicate ACKs.

The server never received our client's ACK! The server thought the connection was still opening, while our client thought the handshake was complete and was waiting for the server's handshake greeting.

To see where the packets were vanishing, we ran simultaneous `tcpdump` traces during a connection attempt.

On the healthy node (`1116514`), every outbound packet appeared on the OVN host interface (`ovn0`) and then on the Geneve tunnel interface (`genev_sys_6081`):
```text
ovn0 Out: SYN -> genev_sys_6081 Out: SYN
genev_sys_6081 In: SYN-ACK -> ovn0 In: SYN-ACK
ovn0 Out: ACK -> genev_sys_6081 Out: ACK
# Greeting received! Handshake complete.
```

On the broken node (`1116512`), something bizarre happened:
```text
ovn0 Out: SYN -> genev_sys_6081 Out: SYN
genev_sys_6081 In: SYN-ACK -> ovn0 In: SYN-ACK
genev_sys_6081 Out: ACK          <-- NOTICE: Never appeared on ovn0!
genev_sys_6081 Out: ACK (re-tx)  <-- Never appeared on ovn0!
```

The SYN traversed `ovn0` normally. But the ACK and every subsequent packet **bypassed `ovn0` entirely** and showed up only on the Geneve tunnel interface—and yet the remote pod never saw them!

```mermaid
flowchart TD
    subgraph Client Node ["Host: 1116512-blockstorage-prod"]
        App["Cinder / MariaDB Client"] --> IPVS["IPVS (ClusterIP: 10.233.43.30:3306)"]
        IPVS --> DNAT["DNAT -> Pod IP: 10.236.24.130"]
        DNAT --> OVN0["dev ovn0 (Host Egress)"]
        
        OVN0 -->|"TCP SYN (ct_state: -trk-est)"| OVS["OVS Software Datapath (openvswitch.ko)"]
        OVS -->|"Geneve Encap to 172.26.64.13 (Current MariaDB Host)"| TunnelDev["genev_sys_6081"]
        
        OVN0 -.->|"TCP ACK (ct_state: +trk+est) Intercepted by Orphaned TC Flower!"| TC["Orphaned TC Filter (handle 0x2c)"]
        TC ==>|"mirred stolen (Frozen Destination: 172.26.64.11)"| DeadChassis["Old MariaDB Chassis (172.26.64.11) 💥 BLACK HOLE"]
    end
    
    TunnelDev -->|"Delivered"| LiveMariaDB["Node: 1335012 (MariaDB Pod)"]
```

---

## The Ghost in the Stack: Orphaned TC Flowers

How could packets skip the interface capture point and end up tunneled to oblivion? We inspected the Linux Traffic Control (`tc`) filters on `ovn0`:

```shell
root@1116512-blockstorage-prod:~# tc -s filter show dev ovn0 egress
```

Buried in the dump was this rule:

```text
filter protocol ip pref 2 flower chain 0 handle 0x2c
  eth_type ipv4
  ip_proto tcp
  src_ip 100.64.0.101
  dst_ip 10.236.24.130
  ct_state +trk+est-rel-rpl
  not_in_hw
  action order 1: tunnel_key set src_ip 172.26.64.191 dst_ip 172.26.64.11 key_id 3 ...
  action order 2: mirred (Egress Redirect to device genev_sys_6081) stolen
```

Let's dissect what this rule was doing:

1. **`src_ip 100.64.0.101`**: The Kube-OVN join subnet IP for this node.
2. **`dst_ip 10.236.24.130`**: The Kubernetes Pod IP of MariaDB.
3. **`ct_state +trk+est`**: It only matched **established** connections! The initial SYN had `ct_state -trk-est`, so it bypassed this filter and flowed through the normal OVS software pipeline.
4. **`tunnel_key set ... dst_ip 172.26.64.11`**: It encapsulated the packet directly into Geneve and addressed the tunnel to chassis **`172.26.64.11`**.
5. **`mirred ... stolen`**: It stole the packet immediately at the TC egress hook—before tcpdump on `ovn0` ever saw it!

And here is the kicker: **MariaDB was not at `172.26.64.11` anymore.**

Twelve days earlier, during an automated TLS certificate rotation orchestrated by `mariadb-operator`, the single-replica MariaDB pod had been deleted and rescheduled from node `1335010` (overlay IP `172.26.64.11`) to `1335012` (overlay IP `172.26.64.13`). 

The live OVS software datapath knew this and correctly routed to `.13`. But the TC flower filter was a frozen relic pointing at `.11`. Every ACK sent by Cinder was being stolen by the kernel TC subsystem and thrown into the void on a host that no longer ran the database!

### Why Were These Rules Still in the Kernel?

When hardware offload was enabled (`hw-offload=true`), OVS offloaded flows into the Linux kernel TC flower subsystem. 

When we flipped `other_config:hw-offload=false`, OVS gracefully switched back to its own kernel datapath module (`openvswitch.ko`). But **OVS does not delete or flush existing TC flower rules when hardware offloading is disabled.** It simply stops talking to the TC subsystem!

Because OVS stopped managing them, the revalidation thread stopped reaping them. They became permanent **orphans** in the Linux kernel. 

And it wasn't just on `ovn0`. When we checked the Geneve tunnel interface (`genev_sys_6081`), we found **94 orphaned ingress filters**. Many looked like this:

```text
action order 2: mirred (Egress Redirect to device *) stolen
```

In `tc` output, `device *` is what `iproute2` prints when an action's target interface index (`ifindex`) is 0. That happens when the target network device (such as a pod's veth pair `*_h`) has been deleted. When a packet hits a filter pointing to a deleted interface, the kernel logs `tc mirred: target device is gone` once and silently drops every packet thereafter.

Any packet arriving on the tunnel whose tuple collided with one of these dormant filters was silently stolen and blackholed.

---

## Remediation 1.0 and the Hidden Blind Spots

Once we understood the problem, we set out to build automation. We created a read-only audit script (`scripts/tc-offload-audit.sh`) and an Ansible remediation playbook (`ansible/playbooks/clear-stale-tc-flower.yaml`) in [PR #1740](https://github.com/rackerlabs/genestack/pull/1740).

The strategy was straightforward:
- Confirm OVS reports zero live TC datapath flows (`ovs-appctl dpctl/dump-flows type=tc`).
- Delete stale flower filters and ingress qdiscs on affected interfaces.

It seemed to work. But as we observed the cluster over the subsequent weeks, intermittent network timeouts continued to pop up on isolated nodes. When we dug deeper, we realized our first-generation tooling was riddled with critical architectural blind spots.

### Blind Spot 1: The "Chain 0" Fallacy

In the initial audit script and playbook, the commands were explicitly scoped to **`chain 0`**:

```bash
# Old playbook snippet
pre=$(tc filter show dev $d ingress chain 0 2>/dev/null | grep -c "^filter.*flower")
if [ "$pre" -gt 0 ]; then
  tc filter del dev $d ingress chain 0 2>/dev/null
fi
```

And in `tc-offload-audit.sh`:

```bash
# Old audit verdict logic
elif [ "$hw" = "false" ] && [ "$live_flows" = "0" ] && [ "$chain0_max" -eq 0 ]; then
  verdict="RESIDUAL-ONLY (Acceptable)"
```

This was a massive mistake. 

OVS flower offloading uses **goto-chains** extensively. Complex flows match on chain 0, perform an action (like `ct` or `pedit`), and branch into `chain 1`, `chain 2`, or arbitrary chain IDs like `chain 234159283`.

When `chain 0` was cleared, the higher-numbered chains remained lodged in the kernel on `genev_sys_6081` and shared ingress blocks (`block 10`, `block 11`). The audit script saw `chain 0` was empty, cheerfully declared the node `RESIDUAL-ONLY (Acceptable)`, exited with code `0`, and allowed our deployment pipelines to proceed!

Meanwhile, dozens of dormant filters containing `mirred (Egress Redirect to device *) stolen` remained alive on higher chains, ready to swallow packets whenever an ephemeral port or tunnel ID collided with them.

### Blind Spot 2: Hostname Resolution and the Skipped Node

In our Ansible playbook, the task needed to execute `ovs-appctl` inside the local `ovs-ovn` pod to check flow status. It built a dictionary mapping node names:

```yaml
# Old playbook lookup
ovs_pod: "{{ ovs_pod_by_node[node_name] | default('') }}"
```

In production, inventory hostnames often vary between short names (`1116481-blockstorage-prod`) and fully qualified domain names (`1116481-blockstorage-prod.example.com`), whereas Kubernetes `spec.nodeName` uses the FQDN.

When an inventory host failed an exact string match, `ovs_pod` resolved to an empty string. The playbook encountered:

```yaml
- name: Skip hosts that do not run an ovs-ovn pod
  when: ovs_pod | length == 0
  block:
    - name: Mark NO-OVS-POD
```

The node was silently marked `NO-OVS-POD` and skipped! The playbook reported a clean run, while the node sat completely un-remediated in production with all its stranded flower rules intact.

### Blind Spot 3: Dynamic Device Discovery Gaps

The original cleanup loop only inspected devices discovered via:

```bash
tc qdisc show | grep -E "^qdisc (ingress|clsact)" | awk '{print $5}'
```

In certain kernel states, interfaces like `genev_sys_6081`, `ovn0`, or `br-int` had ingress filters attached without reporting an explicit `ingress_block` or standard qdisc annotation in the summary output. They slipped past the cleanup undetected.

---

## Remediation 2.0: PR #1843 & The Fleet-Wide Sweep

We went back to the drawing board and overhauled both tools in [PR #1843](https://github.com/rackerlabs/genestack/pull/1843/) (commit `19f81a16`).

### 1. Rewriting `scripts/tc-offload-audit.sh`

We turned the audit script into an uncompromising gate:

- **Auditing All Chains (`chain0/total`):** Instead of only checking chain 0, the script now tallies both chain 0 and total flower filters across all chains on shared ingress blocks and interfaces:
  ```text
  BLOCKS(chain0/total)     DEVS(chain0/total)
  none                     genev_sys_6081=0/73
  ```
- **Strict Verdicts (`ORPHANED-RESIDUAL`):** We eliminated `RESIDUAL-ONLY (Acceptable)`. Any node with non-zero flower filters on any chain is marked `ORPHANED-RESIDUAL` and triggers an exit code of `2`.
- **Explicit Device Auditing:** We added explicit checks for `genev_sys_6081`, `ovn0`, `mirror0`, and `br-int`, ensuring no critical device escapes scrutiny.

### 2. Overhauling `ansible/playbooks/clear-stale-tc-flower.yaml`

We re-engineered the playbook for comprehensive, atomic cleanup:

- **Multi-Chain Deletion:** The playbook dynamically queries every active chain ID and deletes them one by one before deleting the base filter:
  ```bash
  for c in $(tc filter show dev $d ingress 2>/dev/null | grep -oP "chain \K\d+" | sort -un); do
    tc filter del dev $d ingress chain $c 2>/dev/null || true
  done
  tc filter del dev $d ingress 2>/dev/null || true
  ```
- **Targeted Qdisc Removal:** If flower rules are detected, we delete the `clsact` and `ingress` qdiscs completely (`tc qdisc del dev $d clsact`), guaranteeing clean packet fall-through directly to `openvswitch.ko`.
- **Preserving Pod QoS:** We specifically target `flower` classifiers, leaving pod bandwidth rate-limiting filters (`matchall` / `u32` with action `police`) untouched.
- **OVS Software Datapath Flush:** Immediately after removing the kernel filters, the playbook triggers:
  ```bash
  ovs-appctl dpctl/del-flows
  ```
  This forces OVS to flush its software flow cache and cleanly re-instantiate active flows from OpenFlow rules.
- **Hostname Normalization:** We fixed the hostname mapping dictionary to index both FQDNs and short hostnames:
  ```yaml
  ovs_pod_by_node: >-
    {{ dict(tc_ovs_pods | map(attribute='spec.nodeName')
            | zip(tc_ovs_pods | map(attribute='metadata.name')))
       | combine(dict(tc_ovs_pods | map(attribute='spec.nodeName')
                      | map('regex_replace', '\..*$', '')
                      | zip(tc_ovs_pods | map(attribute='metadata.name')))) }}
  ```
  No more `NO-OVS-POD` false skips.

---

## Validation & Results

Before rolling across our production fleet, we performed a dry-run and canary execution on `1116474-blockstorage-prod`:

```shell
ansible-playbook -i /etc/genestack/inventory/inventory.yaml \
  /opt/genestack/ansible/playbooks/clear-stale-tc-flower.yaml \
  --limit 1116474-blockstorage-prod.example.com \
  -e tc_cleanup_dry_run=true
```

Output:
```text
TASK [Report (dry run) orphaned flower filters on shared blocks and per-device qdiscs]
ok: [1116474-blockstorage-prod.example.com] => {
    "msg": "1116474-blockstorage-prod.example.com [DRY-RUN]: genev_sys_6081:73->73"
}
```

The playbook correctly found all 73 stranded rules on `genev_sys_6081`. 

We executed the remediation live (`-e tc_cleanup_dry_run=false`). The cleanup completed in seconds without a single packet drop, interface flap, or socket disconnect. A direct verification check with `tc -s filter show dev genev_sys_6081 ingress` returned completely clean.

With confidence restored, we executed the playbook across all 121 production nodes (compute, control, storage, and network). 

We ran the post-remediation audit:

```text
((genestack) ) [PROD] ubuntu@1335008-overseer01:/opt/genestack/scripts$ ./tc-offload-audit.sh
=== per-node (full TSV: /home/ubuntu/maint/tc-offload-audit-2026-10-06-1821.tsv) ===
...
=== summary ===
ovs-ovn pods audited : 121
  CLEAN           121
by role label       :
  NOLABEL/CLEAN                       8
  compute,network/CLEAN               64
  control/CLEAN                       6
  network/CLEAN                       4
  storage/CLEAN                       39
nodes needing attention: none
```

**100% of nodes transitioned to `CLEAN`.** 

MariaDB ClusterIP connectivity immediately stabilized across all block storage nodes. Cinder services started cleanly, established their database connections, and registered their heartbeats without delay.

---

## Lessons Learned

This incident was one of the most subtle networking bugs we have encountered, and it left us with several key takeaways for operating large-scale software-defined networks:

1. **Turning off a userspace feature does not clean up the kernel.** Setting `other_config:hw-offload=false` in OVS stopped it from managing TC rules, but it did not remove existing rules. The kernel faithfully executed those abandoned rules until we explicitly purged them.
2. **TC hooks precede software datapath hooks.** When debugging packet loss in OVS/OVN environments, never assume `ovs-appctl dpctl/dump-flows` or interface tcpdump tells the full story. If a packet is stolen by a TC filter at the ingress/egress hook, it never enters the OVS pipeline at all.
3. **Beware the "Chain 0" assumption.** Modern kernel networking uses multi-chain classification. When auditing or pruning TC flower rules, always inspect all chains. A clean `chain 0` can easily conceal active packet-dropping filters on higher-numbered chains.
4. **Normalize your inventory keys.** In cross-system automation bridging Ansible and Kubernetes, never rely on a single hostname format. Normalize both FQDNs and short hostnames to prevent critical nodes from being silently skipped.
5. **Canary with dry-runs and automated gates.** By building an audit script with strict exit codes and pairing it with a dry-run enabled playbook, we were able to safely remediate 121 production hypervisors and storage appliances during live customer operations without downtime.

---

*Have you encountered orphaned TC filters or unexpected hardware offload behavior in your OVS or Kube-OVN clusters? Join us in the [Rackspace Discord](https://discord.gg/2mN5yZvV3a) or check out our open-source repositories on [GitHub](https://github.com/rackerlabs/genestack).*
