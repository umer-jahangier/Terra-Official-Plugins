# Slurm Login

![Slurm Login](https://raw.githubusercontent.com/juno-fx/Terra-Official-Plugins/refs/heads/main/plugins/slinky-login/scripts/assets/logo.png)

**Category:** Compute
**Type:** Workload Template
**Tags:** `slurm` `hpc` `terminal` `batch` `cluster-level` `workload`

---

## Overview

A browser terminal onto a Slurm cluster deployed by the [Slinky](../slinky/README.md) plugin. Each user launches
their own session from Hubble, signs in as themselves, and gets an ordinary shell on the Slurm login node — where
`sbatch`, `srun`, `squeue`, `sinfo` and `sacct` all work normally.

Administrators install this plugin and configure the template in Genesis; end users launch it from Hubble.

---

## How It Works

**The terminal pod contains no Slurm software and no Slurm credentials.** It runs
[wetty](https://github.com/butlerx/wetty) in SSH mode and connects to the Slurm login node (a `LoginSet`, deployed by
the Slinky plugin) over SSH:

```
namespace: <user's project>              namespace: slurm

  browser ──▶ Ingress ──▶ [wetty pod] ──ssh──▶ [LoginSet pod]
              (Hubble auth)   thin client       sackd + sshd + sssd
                                                slurm.conf + slurm.key
                                                        │
                                                        ▼
                                                  [slurmctld] ──▶ [slurmd × N]
```

This matters because Slurm's CLI tools are not HTTP clients — `sbatch` and `srun` speak Slurm's own binary RPC
protocol, authenticate with a shared cluster key, and in `srun`'s case the client process itself becomes part of the
running job. A pod can only do that with the Slurm binaries, `slurm.conf`, and the auth key present locally.

Rather than replicate all three into every user's namespace, this template borrows the login node that already has
them. The consequences:

- **No `slurm.key` outside the `slurm` namespace.** That key is a cluster-wide credential — anyone holding it can
  impersonate any Slurm user. It never leaves the namespace it was created in.
- **Users authenticate as themselves.** SSSD on the login node decides who they are, so job ownership, fairshare and
  accounting all attribute correctly. Configure the directory with the Slinky plugin's `ldap_*` fields.
- **No version coupling.** The terminal image does not have to track the Slurm release, because it contains no Slurm.
- **Egress is locked down.** A NetworkPolicy permits exactly two things out of the pod: cluster DNS, and TCP to the
  login node. Nothing else.

### One template, many users

This is a single template that serves everyone. Kuiper renders it per launch with the launching user's identity
injected (`.Values.user`, `.Values.name`, `.Values.puid`, `.Values.guid`), so each session SSHes out as its own user
and lands in its own pod. Nothing about a specific user is baked into the template.

---

## Prerequisites

- **Slinky plugin** installed, with `login_enabled` set to `true`
- **A directory for user identity** — set the Slinky plugin's `ldap_uri` / `ldap_search_base` fields, e.g. pointing at
  the Simple LDAP plugin. Without it the login node only knows local accounts and ordinary users cannot sign in
- Network connectivity from the user's project namespace to the Slurm namespace on the login port (the bundled
  NetworkPolicy allows this; a stricter cluster-wide policy could still block it)

---

## Installation

Administrators install the plugin once:

1. Open **Terra** and navigate to the **Plugin Marketplace**
2. Search for **"Slurm Login"**
3. Click **Install**

The template then appears in **Genesis** for configuration, and users launch sessions from **Hubble**.

---

## Configuration

### Install-Time Fields

No install-time configuration is required for this plugin.

### Workload Launch Fields

| Field | Details |
|-------|---------|
| `slurmNamespace` | **string** · Required · Default: `slurm`<br>Namespace the Slurm cluster runs in — must match the Slinky plugin's `cluster_namespace` |
| `loginService` | **string** · Required · Default: `slinky-slurm-login-slinky`<br>Name of the login node Service. For a Slinky release named `<name>` this is `<name>-slurm-login-slinky` |
| `loginPort` | **int** · Required · Default: `22`<br>SSH port on the login node Service |
| `registry` | **string** · Required · Default: `wettyoss`<br>Registry for the terminal image |
| `repo` | **string** · Required · Default: `wetty`<br>Repository for the terminal image |
| `tag` | **string** · Required · Default: `2.5`<br>Tag for the terminal image |
| `nginx_registry` | **string** · Required · Default: `docker.io`<br>Registry for the nginx sidecar image |
| `nginx_repo` | **string** · Required · Default: `nginx`<br>Repository for the nginx sidecar image |
| `nginx_tag` | **string** · Required · Default: `1.29.3`<br>Tag for the nginx sidecar image |
| `publicAccess` | **boolean** · Required · Default: `false`<br>Allow public access, disabling Hubble authentication on the ingress |
| `ingressNamespace` | **string** · Required · Default: `ingress-nginx`<br>Namespace of the ingress controller, used by the NetworkPolicy |

### Custom Environment Variables

None. This pod is a thin SSH client — the shell a user actually works in runs on the Slurm login node, so anything
affecting the session belongs in that node's profile rather than in workload environment variables.

---

## Finding the Login Service Name

The default assumes a Slinky release named `slinky` (Terra's default). To check:

```sh
kubectl get svc -n slurm -l app.kubernetes.io/part-of=slurm | grep login
```

Use that Service name for the `loginService` field.

---

## Using the Session

Launch the workload from Hubble and open it. You are prompted for your directory password, then land on the login
node:

```sh
sinfo                      # partition and node state
srun hostname              # run an interactive job
sbatch --wrap="sleep 60"   # submit a batch job
squeue                     # queued and running jobs
sacct                      # accounting records (needs accounting enabled in Slinky)
```

Jobs run under your own Slurm identity, so `squeue --me` and fairshare behave as expected.

---

## Notes

- The session is not persistent — closing the browser drops the SSH connection. Run long work under `tmux` or
  `screen` on the login node, or submit it with `sbatch` so it survives independently of the terminal
- Home directories live on the login node. To share files between Slurm jobs and other Juno workloads, mount the same
  storage into both the LoginSet and the NodeSets via the Slinky plugin
- For scripted or automated submission, use `slurmrestd` — the Slinky plugin deploys it, and it is a genuine HTTP API
  needing only a JWT and `curl`, with no Slurm binaries and no terminal
- `publicAccess` disables Hubble authentication on the ingress. The login node still demands valid credentials, but
  the terminal becomes reachable by anyone who can reach the ingress — leave it `false` unless you have a reason
