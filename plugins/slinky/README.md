# Slinky

![Slinky](https://raw.githubusercontent.com/juno-fx/Terra-Official-Plugins/refs/heads/main/plugins/slinky/assets/logo.png)

**Category:** Compute
**Type:** Cluster Service
**Tags:** `slurm` `hpc` `scheduler` `batch` `operator` `cluster-level`

---

## Overview

[Slinky](https://slurm.schedmd.com/slinky.html) is SchedMD's toolkit for running [Slurm](https://slurm.schedmd.com/) and
Kubernetes together. This plugin installs the **slurm-operator** half of Slinky: a Kubernetes operator that deploys and
manages a real Slurm cluster — `slurmctld`, `slurmd` compute nodes, login nodes, and `slurmrestd` — as pods on your
existing Kubernetes nodes.

The result is a fully functional Slurm cluster that users interact with exactly as they would on bare metal: `sbatch`,
`srun`, `squeue`, `sinfo`, `sacct`, MPI, job dependencies, fairshare, preemption and QoS all work normally. Kubernetes
supplies the hardware and the lifecycle management; Slurm supplies the batch scheduler.

---

## How It Works

**Slinky does not translate Slurm jobs into Kubernetes Jobs.** It is not a dashboard and it has no web UI. It runs
Slurm itself, containerized, under an operator.

The operator introduces Custom Resources that describe the pieces of a Slurm cluster, and reconciles them into pods:

| Custom Resource | What it becomes |
|-----------------|-----------------|
| `Controller` | The `slurmctld` control plane pod, with a PVC for state-save data |
| `NodeSet` | A set of homogeneous `slurmd` compute node pods (scaled like a StatefulSet, or one-per-node like a DaemonSet) |
| `LoginSet` | Login / submit node pods running `sshd` + `sackd` + SSSD — the front door for users |
| `RestApi` | A `slurmrestd` pod exposing the Slurm REST API |
| `Accounting` | A `slurmdbd` pod for job accounting (optional, needs a database) |

A job submitted with `sbatch` from a login pod is scheduled by `slurmctld` onto a `slurmd` pod and runs there. The
Kubernetes scheduler is not involved in placing that job — it only placed the `slurmd` pods themselves. The operator
watches in the other direction too: before it scales in or upgrades a `NodeSet`, it drains the corresponding Slurm
nodes so running jobs are not killed.

**Cluster Service** — Installed once per cluster by an administrator. The operator is a cluster singleton; the Slurm
cluster it manages lives in its own namespace and can be re-installed for additional clusters (see
[Multiple Slurm Clusters](#multiple-slurm-clusters)).

### What this plugin installs

Three ArgoCD Applications plus the accounting database, ordered by sync wave:

| Wave | Resource | Chart / Kind | Namespace |
|------|----------|--------------|-----------|
| 0 | `<release>-slinky-crds` | `slurm-operator-crds` | `slinky` |
| 0 | `cluster_namespace` | `Namespace` | — |
| 1 | `<release>-slinky-operator` | `slurm-operator` | `slinky` |
| 1 | `mariadb` | `StatefulSet` + `Service` + `Secret` (only when `accounting_enabled`) | `cluster_namespace` |
| 2 | `<release>-slurm` | `slurm` | `cluster_namespace` (default `slurm`) |

The three charts come from SchedMD's OCI registry at `ghcr.io/slinkyproject/charts`. The wave ordering matters: the
CRDs must be established before the operator, and both before any Slurm custom resource; the database must be up
before `slurmdbd` tries to connect.

### What this plugin does *not* install

Slinky's other half, **slurm-bridge**, does the reverse of the operator: it registers Slurm as a Kubernetes scheduler
so that Kubernetes `Pods`, `Jobs` and `JobSets` are translated into Slurm jobs for scheduling, with `slurm-bridge`
binding the resulting pods to whichever nodes Slurm allocated. That is the piece people are usually thinking of when
they ask about "converting" between the two systems.

It is deliberately deferred rather than ruled out. `slurm-bridge` needs **Kubernetes v1.35 or newer**, an
already-running Slurm cluster, a manually created JWT secret, and specific
[DRA](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) drivers for CPU and GPU.
It also allocates whole nodes exclusively per pod, and native `cpu` requests do not activate CPU DRA — pods must
explicitly request `deviceclass.resource.kubernetes.io/dra.cpu` instead, which changes how workloads must be written.

The version floor is met on current deployments, so adding it later is a matter of appetite for those constraints, not
of cluster capability. It builds on this plugin: install Slinky first, then `slurm-bridge` against the resulting
cluster.

---

## Prerequisites

- **Kubernetes v1.29 or newer** (Slinky v1.2 compatibility matrix)
- **A default StorageClass**, or a StorageClass named in `storage_class` — the controller persists `slurmctld`
  state-save data to a PVC
- **NVIDIA GPU Operator plugin** if `worker_gpus` is greater than `0`
- **MetalLB plugin** (or another load balancer controller) *only* if you change `login_service_type` to
  `LoadBalancer` — the default `ClusterIP` needs nothing
- **Certificate Manager plugin** *only* if you set `cert_manager` to `true` — off by default

Nothing else is required. The accounting database is deployed by this plugin, and the webhook certificate is
self-signed in-cluster, so a default install has no external dependencies.

---

## Installation

1. Open **Terra** and navigate to the **Plugin Marketplace**
2. Search for **"Slinky"**
3. Click **Install**
4. Fill in the configuration fields below
5. Click **Confirm** to deploy

---

## Configuration

### Install-Time Fields

Only `chart_version` is genuinely required. Everything else has a working default, and the fields below the
`deploy_cluster` line describe the Slurm cluster — they are ignored entirely when `deploy_cluster` is off, which is
why none of them are marked required. (Terra has no conditional fields, so they stay visible on the form regardless.)

| Field | Details |
|-------|---------|
| `chart_version` | **select** · Required · Default: `1.2.1`<br>Version applied to all three Slinky charts |
| `install_crds` | **boolean** · Optional · Default: `true`<br>Install the Slinky CRDs. Set `false` on any additional install so it does not contend with the first one over cluster-scoped CRDs |
| `install_operator` | **boolean** · Optional · Default: `true`<br>Install the operator and webhook into the `slinky` namespace. The operator is a cluster singleton |
| `cert_manager` | **boolean** · Optional · Default: `false`<br>Issue the webhook certificate via cert-manager instead of the chart's own self-signed CA. Off by default so the install stays self-contained |
| `deploy_cluster` | **boolean** · Optional · Default: `true`<br>Deploy a Slurm cluster. `false` installs only the operator, leaving you to manage Slinky CRs yourself |
| `cluster_namespace` | **string** · Optional · Default: `slurm`<br>Namespace for the Slurm cluster. Use a distinct value per cluster. Ignored when `deploy_cluster` is off |
| `worker_replicas` | **int** · Optional · Default: `2`<br>Number of `slurmd` compute node pods. Supports scale-to-zero. Ignored when `deploy_cluster` is off |
| `worker_cpu` | **string** · Optional · Default: `2`<br>CPU request and limit per compute node. Becomes the node's CPU count in Slurm. Ignored when `deploy_cluster` is off |
| `worker_memory` | **string** · Optional · Default: `4Gi`<br>Memory request and limit per compute node. Becomes the node's `RealMemory` in Slurm. Ignored when `deploy_cluster` is off |
| `worker_gpus` | **int** · Optional · Default: `0`<br>NVIDIA GPUs per compute node. Above `0` this also sets `GresTypes=gpu`, `Gres=gpu:<n>` and `gres.conf` `AutoDetect=nvidia` |
| `storage_class` | **string** · Optional · Default: *(empty)*<br>StorageClass for the `slurmctld` state-save PVC. Empty uses the cluster default |
| `state_size` | **string** · Optional · Default: `4Gi`<br>Size of the `slurmctld` state-save PVC. Ignored when `deploy_cluster` is off |
| `login_enabled` | **boolean** · Optional · Default: `true`<br>Deploy a login node users SSH into to submit jobs |
| `login_service_type` | **select** · Optional · Default: `ClusterIP`<br>How the login node's SSH port is exposed. `ClusterIP` suffices when users connect via the Slurm Login workload; use `LoadBalancer` or `NodePort` only for SSH from outside the cluster |
| `login_ssh_public_key` | **string** · Optional · Default: *(empty)*<br>An SSH public key granted root access to the login node. Works even when SSSD is misconfigured |
| `accounting_enabled` | **boolean** · Optional · Default: `false`<br>Deploy `slurmdbd` plus a bundled MariaDB — see [Accounting](#accounting) |
| `accounting_db_password` | **string** (secret) · Optional · Default: *(empty)*<br>Password for the `slurm` accounting database user. Required when accounting is enabled |
| `accounting_db_size` | **string** · Optional · Default: `8Gi`<br>Size of the accounting database volume |
| `ldap_uri` | **string** · Optional · Default: *(empty)*<br>LDAP URI for login node user identity, e.g. `ldap://simple-ldap.argocd.svc.cluster.local:389`. Empty means local accounts only |
| `ldap_search_base` | **string** · Optional · Default: *(empty)*<br>LDAP search base DN, e.g. `dc=example,dc=org`. Required when `ldap_uri` is set |
| `ldap_bind_dn` | **string** · Optional · Default: *(empty)*<br>Bind DN for directory queries. Empty means anonymous bind |
| `ldap_bind_password` | **string** (secret) · Optional · Default: *(empty)*<br>Password for `ldap_bind_dn` |

---

## Using the Cluster

### From the Slurm Login workload (how end users get in)

Install the companion **[Slurm Login](../slinky-login/README.md)** plugin. It adds a workload template that users
launch from Hubble: a browser terminal that SSHes into this cluster's login node and drops them at a shell where
`sbatch` and friends work.

That workload holds no Slurm credentials of its own — it borrows the login node deployed here, which already has
`slurm.conf` and the auth key. Users sign in as themselves against the directory configured in
[User Identity on the Login Node](#user-identity-on-the-login-node), so job ownership and fairshare attribute
correctly. This is why `login_service_type` defaults to `ClusterIP`: nothing outside the cluster needs to reach it.

### Over SSH from outside the cluster

Only if you set `login_service_type` to `LoadBalancer` (or `NodePort`) and provided `login_ssh_public_key`:

```sh
SLURM_LOGIN_IP="$(kubectl get svc -n slurm <release>-slurm-login-slinky -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
SLURM_LOGIN_PORT="$(kubectl get svc -n slurm <release>-slurm-login-slinky -o jsonpath='{.status.loadBalancer.ingress[0].ports[0].port}')"
ssh -p "${SLURM_LOGIN_PORT:-22}" "root@${SLURM_LOGIN_IP}"
```

### From the REST API

`slurmrestd` is deployed with the cluster and is a genuine HTTP API — the right choice for scripted submission, since
it needs only a JWT and `curl`, with no Slurm binaries anywhere.

### From the controller pod

No SSH or load balancer required — useful for a quick smoke test:

```sh
kubectl exec -n slurm <release>-slurm-controller-0 -- sinfo
kubectl exec -n slurm <release>-slurm-controller-0 -- srun hostname
```

### Standard Slurm commands

```sh
sinfo                      # partition and node state
srun hostname              # run an interactive job
sbatch --wrap="sleep 60"   # submit a batch job
squeue                     # queued and running jobs
sacct                      # accounting records (requires accounting_enabled)
```

All compute nodes land in a single partition named `all`, which is the default partition with no time limit.

### Scaling compute nodes

Edit `worker_replicas` in Terra, or scale the `NodeSet` directly. The operator drains Slurm nodes before terminating
their pods, so running jobs are not lost:

```sh
kubectl scale nodeset -n slurm <release>-slurm-worker-slinky --replicas=8
```

---

## Multiple Slurm Clusters

The CRDs and the operator are cluster-wide singletons; a Slurm cluster is not. To run a second, independent Slurm
cluster, install this plugin again with:

- `install_crds` → `false`
- `install_operator` → `false`
- `cluster_namespace` → a namespace not already in use

The single operator installed by the first release reconciles every Slurm cluster in the Kubernetes cluster.

---

## Accounting

Setting `accounting_enabled` to `true` deploys `slurmdbd`, which unlocks `sacct`, fairshare and QoS — **and a MariaDB
database alongside it**. No external database, and no additional plugin, is required.

A StorageClass on its own is not sufficient here: `slurmdbd` needs something speaking the MySQL wire protocol, not
just a volume. The bundled database is a single-replica `StatefulSet` whose PVC uses the same `storage_class` field as
the controller, sized by `accounting_db_size`.

The names it uses are deliberately chosen to match the Slurm chart's default `accounting.storageConfig`, so the two
sides wire themselves together with no further configuration:

| Setting | Value |
|---------|-------|
| Host | `mariadb` (Service in `cluster_namespace`) |
| Port | `3306` |
| Database | `slurm_acct_db` |
| User | `slurm` |
| Password | Secret `mariadb-password`, key `password` |

Set `accounting_db_password` when enabling accounting; install fails fast with a clear message if it is left empty.

> **Choose the password once.** It is applied when the database initialises. Changing it later updates the Secret but
> not the credential already stored inside MariaDB, and `slurmdbd` will then fail to connect.

The database is intended for Slurm's accounting records only — the root account is randomised and unused, and the
Service is reachable only from within the namespace.

---

## User Identity on the Login Node

By default the login node knows only local accounts, so `login_ssh_public_key` is the only way in. To let ordinary
users sign in, point SSSD at a directory with the `ldap_*` fields:

| Field | Example |
|-------|---------|
| `ldap_uri` | `ldap://simple-ldap.argocd.svc.cluster.local:389` |
| `ldap_search_base` | `dc=example,dc=org` |
| `ldap_bind_dn` | `cn=admin,dc=example,dc=org` (omit for anonymous bind) |
| `ldap_bind_password` | the bind password |

This generates the login node's `sssd.conf` for you. Using the same directory the rest of the platform authenticates
against means Slurm job ownership, fairshare and accounting all attribute to the right person.

> TLS verification is disabled on the LDAP connection (`ldap_tls_reqcert = never`), which is appropriate for an
> in-cluster directory reached over the pod network. Front the directory with TLS you actually validate before
> pointing this at anything outside the cluster.

---

## Notes

- Slinky v1.2 requires Kubernetes v1.29+ and ships Slurm 26.05 container images
- With `cert_manager` off (the default) the operator chart mints its own 10-year webhook CA. Because Helm regenerates
  that keypair on every render, the plugin sets `ignoreDifferences` on the generated Secret and webhook `caBundle`
  fields — without it ArgoCD would report permanent drift and restart the webhook on every refresh
- The operator drains Slurm nodes before scale-in and rolling upgrades, so in-flight jobs survive `NodeSet` changes
- `slurmctld` HA is achieved by Kubernetes restarting the controller pod, which is typically faster than a backup
  controller taking over — no shared filesystem is required
- Slurm configuration changes are detected and applied without restarting the control plane; propagation is bounded by
  the kubelet's ConfigMap sync frequency (60s by default)
- Compute node pods carry Slurm node state (Idle, Allocated, Mixed, Drain, Down …) as pod conditions, so
  `kubectl get pods` reflects what `sinfo` reports
- Uninstalling the plugin does not remove the Slinky CRDs — they are deliberately left un-pruned so a second Slurm
  cluster is never torn out from under a running job
- For deeper configuration (topology, Pyxis, prolog/epilog scripts, IMEX, SR-IOV, autoscaling), see the
  [Slinky documentation](https://slinky.schedmd.com/) and the
  [slurm-operator docs](https://github.com/SlinkyProject/slurm-operator/tree/main/docs)
