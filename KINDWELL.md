# Kindwell test source

This branch is a Terra source for validating the custom domain work on a live cluster before it goes back to the official repository for review. It is the official `main` with every Kindwell pull request branch merged on top, plus one commit that points the new plugins' icons at this branch so they render in the app store.

| Plugin | Change | Upstream PR |
|--------|--------|-------------|
| cert-issuer | new, creates the ACME ClusterIssuer | #698 |
| external-dns | new, DNS records from Ingress and HTTPRoute | #699 |
| domain-route | new, a hostname for any workload or in-session app | #700 |
| domain-manager | new, page showing hostnames, records and certificate state | #701 |
| n8n | `domain` field, served at `<name>.<domain>` | #703 |
| runtime-python, js, go, cpp | `domain` field under `ingress-noauth` | #705 |
| wetty | `published_ports` for in-session apps | fork only, #708 was closed |
| docs | custom domain convention | #706 |

Every chart change renders byte for byte what official `main` renders when its new fields are left empty.

---

## Adding the source

In **Terra**, open the source repo tab, click **NEW SOURCE**, and fill in:

| Field | Value |
|-------|-------|
| Name | `kindwell` |
| URL | `https://github.com/umer-jahangier/Terra-Official-Plugins` |
| Ref | `kindwell` |

No username or token is needed, the repository is public.

## Before installing anything

**n8n, wetty and the four runtimes exist in both sources. Do not uninstall the official copies.** These plugins only create ConfigMaps named after the Install Name, so a Kindwell copy installed under a different name, for example `n8n-kindwell`, sits alongside the official one without touching it, and running workloads and their templates stay exactly as they are. Author a new template from the Kindwell copy for testing. The new plugins, cert-issuer, external-dns, domain-route and domain-manager, have no official counterpart and install cleanly alongside everything else.

**Templates come from the install they were authored against.** A template authored from the official n8n keeps rendering the official chart. To test the new fields, create a new template in Genesis from the Kindwell copy's schema.

**Creating Secrets needs cluster access** in the `cert-manager` and `external-dns` namespaces, which a project-scoped k9s session does not have. Use Headlamp, or `sudo k3s kubectl` on a control plane node.

---

## Test order

Each step lists what passing looks like. Stop at the first failure.

### 1. Certificate Issuer

Install **Certificate Manager** from the official source if it is not already present. Then install **Certificate Issuer** from this source, and point `acme_server` at **staging** for the first run, so a misconfiguration cannot burn through Let's Encrypt production limits.

For HTTP-01 no Secret is needed. For DNS-01, create the Secret described in the plugin README first.

```bash
kubectl get clusterissuer
```

Passing: the issuer shows `READY True`.

### 2. ExternalDNS

Skip this step if records will be created by hand. Otherwise create the credential Secret from the plugin README, install **ExternalDNS** with `domain_filters` set to the test domain, and check the pod:

```bash
kubectl get pods -n external-dns
kubectl logs -n external-dns deploy/external-dns --tail=20
```

Passing: the pod is `Running` with no restarts, and the log reports its provider and sources without errors.

### 3. n8n on its own domain

Point a wildcard record for the test domain at the cluster ingress address, or rely on ExternalDNS from step 2. Launch n8n from the migrated template with `domain`, `tls_issuer` and, if using ExternalDNS, `publish_dns`.

```bash
kubectl get ingress,certificate -n <project>
curl -I https://<name>.<domain>/
curl -X POST https://<name>.<domain>/webhook/<id>
```

Passing: one Ingress for `<name>.<domain>`, the certificate `READY True`, the editor loads over HTTPS, and a webhook POST to an active workflow executes it.

### 4. Runtime template

Launch **runtime-python** with `network_mode` set to `ingress-noauth` and a `domain`.

Passing: the application answers at `https://<name>.<domain>/`, and the same launch without a domain still serves on the platform path.

### 5. An app inside a session

In a **Wetty** session launched with `published_ports` set to `8000`, start something listening on `0.0.0.0:8000`. Node is always present, since Wetty runs on it:

```bash
node -e "require('http').createServer((q, r) => r.end('ok')).listen(8000, '0.0.0.0')"
```

Install **Domain Route** into the same project with `target_name` set to the workload name, `target_port` to `8000`, and `selector_label` to `kuiper.juno-innovations.com/kuiper-instance`, which is the label Wetty pods carry.

Passing: the app answers at the Domain Route hostname, and the Wetty terminal still requires the platform login on its own path.

### 6. Domain Manager

Install **Domain Manager** with `host` set to the Genesis host.

Passing: the page opens in Genesis at `/domains` and lists every hostname from the steps above, with the record shown as `Pointing here` and certificates `Ready`.

---

## Rolling back

Delete the templates authored from Kindwell copies, uninstall the Kindwell plugins from Terra, then delete the `kindwell` source. Official installs were never touched, so nothing needs reinstalling. Certificates and DNS records created during testing are not removed automatically.

## Keeping this branch current

When a pull request branch changes or official `main` moves, rebuild the branch from `main` by merging the pull request branches again. The icon commit and this file are the only changes that exist here alone.
