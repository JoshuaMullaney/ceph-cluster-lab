# ceph-cluster-lab
Practice setting up a functional Ceph cluster on a local machine using virtual machines. Created a running cluster, block storage pools, simulated drive failures, to develop hands-on familiarity with the CLI 
# Ceph Home Lab — 3-Node Cluster on AlmaLinux 9.7

A self-directed home lab project to develop hands-on open-source Ceph administration skills, built to complement an enterprise storage background supporting Dell EMC DMX, VMAX, VMAX3, and PowerMax platforms.

---

## Cluster Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Host Machine (Windows)              │
│                                                      │
│   ┌──────────────┐  ┌──────────┐  ┌──────────┐      │
│   │  ceph-node1  │  │ceph-node2│  │ceph-node3│      │
│   │192.168.56.101│  │  .102    │  │  .103    │      │
│   │              │  │          │  │          │      │
│   │ MON, MGR,    │  │ OSD.1    │  │ OSD.0    │      │
│   │ OSD.2, OSD.5 │  │ OSD.4    │  │ OSD.3    │      │
│   └──────────────┘  └──────────┘  └──────────┘      │
│                                                      │
│         192.168.56.0/24 (VirtualBox Host-Only)       │
└─────────────────────────────────────────────────────┘
```

**Stack:**
- **Hypervisor:** Oracle VirtualBox
- **OS:** AlmaLinux 9.7 (RHEL-compatible)
- **Ceph Release:** Reef (via cephadm)
- **Container Engine:** Podman 5.6.0
- **Nodes:** 3 VMs × 2.5GB RAM, 2 vCPU
- **OSDs:** 6 total (2 per node × 10GB virtual disks)

---

## What This Lab Covers

| Exercise | EMC Parallel |
|---|---|
| cephadm cluster bootstrap | Initial array setup / Unisphere provisioning |
| OSD deployment across hosts | Disk Director / DAE drive assignment |
| RBD pool and image creation | Storage group / LUN provisioning |
| RBD block device host mapping | Host masking / LUN presentation |
| RBD pool mirroring | SRDF synchronous/asynchronous replication |
| OSD failure simulation & recovery | DAE drive failure / RAID rebuild |
| CRUSH map topology | SAN zoning / fabric layout |

---

## Cluster Setup

### Prerequisites

- VirtualBox with Host-Only network adapter (`192.168.56.0/24`)
- AlmaLinux 9.x Minimal ISO
- 3 VMs with 2 extra virtual disks each (10GB per disk)

### Node Configuration (all nodes)

Set static IP on host-only adapter:
```bash
nmcli con mod enp0s8 ipv4.addresses 192.168.56.10X/24
nmcli con mod enp0s8 ipv4.method manual
nmcli con up enp0s8
```

Add all nodes to `/etc/hosts`:
```bash
echo "192.168.56.101  ceph-node1.lab  ceph-node1" >> /etc/hosts
echo "192.168.56.102  ceph-node2.lab  ceph-node2" >> /etc/hosts
echo "192.168.56.103  ceph-node3.lab  ceph-node3" >> /etc/hosts
```

Install prerequisites:
```bash
dnf update -y
dnf install -y curl wget lvm2 python3 podman
systemctl disable --now firewalld
setenforce 0
```

### SSH Key Setup (node1 only)

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
ssh-copy-id root@ceph-node1
ssh-copy-id root@ceph-node2
ssh-copy-id root@ceph-node3
```

### Install cephadm (node1 only)

```bash
curl --silent --remote-name --location \
  https://github.com/ceph/ceph/raw/reef/src/cephadm/cephadm
chmod +x cephadm
mv cephadm /usr/local/bin/
cephadm add-repo --release reef
cephadm install
```

### Bootstrap the Cluster (node1 only)

```bash
hostname ceph-node1
cephadm bootstrap \
  --mon-ip 192.168.56.101 \
  --cluster-network 192.168.56.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password <your-password>
```

Dashboard available at: `https://192.168.56.101:8443`

### Add node2 and node3

```bash
ceph cephadm get-pub-key > ~/ceph.pub
ssh-copy-id -f -i ~/ceph.pub root@ceph-node2
ssh-copy-id -f -i ~/ceph.pub root@ceph-node3

ceph orch host add ceph-node2 192.168.56.102
ceph orch host add ceph-node3 192.168.56.103
```

### Deploy OSDs

```bash
ceph orch apply osd --all-available-devices
```

Verify:
```bash
ceph osd tree
```

---

## RBD Block Storage

Create a pool and provision a block device (equivalent to VMAX LUN provisioning):

```bash
ceph osd pool create rbd-pool 32
rbd pool init rbd-pool
ceph osd pool set rbd-pool size 3
ceph osd pool set rbd-pool min_size 2

rbd create --size 5120 rbd-pool/test-lun-01
rbd map rbd-pool/test-lun-01
mkfs.xfs /dev/rbd0
mount /dev/rbd0 /mnt/ceph-block
```

---

## RBD Mirroring (SRDF Equivalent)

Enable pool-level mirroring and configure an image for replication:

```bash
rbd mirror pool enable rbd-pool image
rbd mirror image enable rbd-pool/test-lun-01 snapshot
rbd mirror image status rbd-pool/test-lun-01
```

**EMC parallel:** RBD snapshot mirroring replicates block device state between pools/clusters — the same concept as SRDF/A (asynchronous) replication between VMAX arrays across sites.

---

## OSD Failure Simulation

Simulate a drive failure and observe cluster self-healing:

```bash
# Take down an OSD
ceph orch daemon stop osd.2

# Watch cluster react in real time
watch ceph -s
```

**Expected behavior:**
1. `HEALTH_WARN` — OSD marked down
2. After ~5 minutes — OSD marked `out`, rebalancing begins
3. Data redistributes across remaining OSDs
4. Restore the OSD:

```bash
ceph orch daemon start osd.2
```

Cluster returns to `HEALTH_OK` as data rebalances back.

**EMC parallel:** Equivalent to a physical drive failure in a VMAX DAE — the difference is Ceph handles rebuild in software across commodity hardware rather than a proprietary RAID controller.

---

## Key Operational Commands

```bash
# Cluster health overview
ceph -s
ceph health detail
```
<img width="601" height="314" alt="image" src="https://github.com/user-attachments/assets/1ee1e00b-d24a-4f2f-b21a-4d7a1dab2741" />
```bash
# OSD layout and status
ceph osd tree
ceph osd stat
ceph osd df
```
<img width="886" height="434" alt="image" src="https://github.com/user-attachments/assets/ed239bc1-1abe-4132-a69e-fc0d2a31e332" />
```bash
# Pool and capacity
ceph df
ceph osd pool ls detail
```
<img width="1276" height="280" alt="image" src="https://github.com/user-attachments/assets/5dd7f546-7c18-4fab-8b30-3d3ad336a01c" />
```bash
# Live activity stream
ceph -w
```
<img width="624" height="293" alt="image" src="https://github.com/user-attachments/assets/eb62997e-9033-4491-bd2d-c42f6ee3b71d" />
```bash
# Performance
ceph osd perf
```
<img width="354" height="147" alt="image" src="https://github.com/user-attachments/assets/86a356ec-02e5-4684-8f84-1a3818c90f5f" />


---

## Troubleshooting Encountered

Real issues hit during deployment and how they were resolved:

| Issue | Resolution |
|---|---|
| `cephadm add-repo --release quincy` returning 404 | Switched to `--release reef` (current stable) |
| AlmaLinux 10 repo incompatibility | Rebuilt VMs on AlmaLinux 9.7 (el9 required for Ceph packages) |
| Bootstrap failing on FQDN hostname | Ran `hostname ceph-node1` to strip the `.lab` suffix before bootstrapping |
| node2/node3 failing host-add check | Installed podman manually on worker nodes (`dnf install -y podman`) |
| SSH root login refused | Enabled `PermitRootLogin yes` in `/etc/ssh/sshd_config` |
| Windows SSH known_hosts conflict after VM rebuild | Cleared with `ssh-keygen -R 192.168.56.10X` |

---

## Related Projects

- [Kubernetes SRE Lab](https://github.com/JoshuaMullaney/kubernetes-sre-project) — Minikube cluster with Prometheus/Grafana monitoring stack

---

## Background

Built as part of preparation for roles focused on Ceph storage, drawing on 8+ years of enterprise storage experience supporting Dell EMC DMX, VMAX, VMAX3, and PowerMax platforms at Dell Technologies.
