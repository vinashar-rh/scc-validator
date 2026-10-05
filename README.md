# SCC Validator

Client-side OpenShift **Security Context Constraint (SCC)** admission simulator.

Paste a workload YAML, set (or import) the project / ServiceAccount context, and see:

- whether admission would **allow** or **forbid** the pod
- **which SCC wins** (OpenShift order: higher priority → more restrictive → name)
- **why other granted SCCs failed**
- **mutations** the winning SCC would inject
- a ready-to-copy **RBAC grant** if you need a higher SCC

Live app: https://vinashar-rh.github.io/scc-validator/

## When to use it

Use this tool when you need to **reason about SCC admission without burning cluster iterations** — especially before you have access, or while you explain a failure to someone else.

**Good fits**

- A pod / Deployment is rejected with `unable to validate against any security context constraint` and you want a clear field-level why
- You are deciding whether to **fix the workload** for `restricted-v2` or ask for a higher SCC (`anyuid`, `hostnetwork-v2`, `privileged`)
- Training / demos: show how UID 0, HostPath, `hostPort`, and privileged DaemonSets fail under a normal project
- Reviewing a PR manifest offline (laptop, no `oc` login) before applying it
- Checking what OpenShift would **mutate** (injected UID, seccomp, `openshift.io/scc`, etc.) if admission succeeded
- Sharing a failing case with a teammate via **Copy share link**

**Not the right tool when**

- You need a definitive cluster answer → use `oc create --dry-run=server` / `oc adm policy scc-subject-review`
- The deny involves a **custom SCC**, image `USER`, or Pod Security Admission (PSA) on top of SCCs
- You must prove what is granted in a live project → export RoleBindings / run `oc adm policy who-can use sccs/...` (then import here if you want)

This is a **teaching / diagnostics workbench**, not a replacement for the API server. Always confirm on-cluster with:

```bash
oc create --dry-run=server -f workload.yaml
oc adm policy scc-subject-review -z <sa> -n <ns> -f workload.yaml
```

Nothing you paste or upload is sent to a server. Evaluation runs in the browser.

---

## Quick start

1. Open the app.
2. Paste a `Pod`, `Deployment`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob`, or similar into **Workload**, or click a **Sample**.
3. Open **Cluster context** and confirm:
   - namespace / ServiceAccount
   - UID range and SELinux MCS (or import them from cluster YAML)
   - which SCCs that ServiceAccount may use (default: `restricted-v2` only)
4. Read **Admission decision**. On deny, open **Violations** and click a finding to jump to the YAML line.
5. Optional: **Auto-fix for restricted-v2**, or use the **RBAC** tab if the workload truly needs a higher SCC.
6. Optional: **Copy share link** to send someone the same YAML + context.

---

## Cluster context

The app never talks to your cluster. You either edit the fields by hand or import exports.

### Default assumption

A new OpenShift project almost always has only **`restricted-v2`** (via `system:authenticated`).  
Tick `anyuid`, `privileged`, etc. only if a cluster-admin already granted them.

### Import from the cluster

In **Cluster context → Import cluster context**, the app shows commands like:

```bash
# Namespace UID range + MCS annotations
oc get ns <ns> -o yaml > ns.yaml

# Project RoleBindings (add-scc-to-user usually lands here)
oc get rolebinding -n <ns> -o yaml > rb.yaml

# ClusterRoleBindings that grant SCCs (optional)
oc get clusterrolebinding -o name | grep ':scc:' | while read -r r; do oc get "$r" -o yaml; echo '---'; done > crb-scc.yaml
```

Then **Upload YAML** or paste into the box and click **Apply pasted YAML**.

| Export | Used for |
|--------|----------|
| Namespace / Project | Name, `openshift.io/sa.scc.uid-range`, `openshift.io/sa.scc.mcs` |
| RoleBinding / ClusterRoleBinding (`system:openshift:scc:*`) | SCCs available to the ServiceAccount |
| ServiceAccount (optional) | SA name |
| SCC `users` / `groups` (optional) | Extra grants |

Custom SCCs in the export are reported but not simulated (the engine models stock SCCs only).

---

## Reading the result

### Admission allowed

- **Winning SCC** is the first granted SCC that accepts the pod under OpenShift sort order.
- **Mutations** lists fields the SCC would inject (UID, `seccomp`, `openshift.io/scc`, etc.).
- If both `restricted-v2` and `anyuid` are granted, a pod that could pass either often gets **`anyuid`** because it has priority **10**. Use the **anyuid priority** sample to see that.

### Admission forbidden

- **Violations** are grouped **per granted SCC**.
- Click a violation (or its `L12` badge) to jump to that line in the editor.
- **RBAC** suggests a least-privilege stock SCC grant (`oc adm policy add-scc-to-user` + RoleBinding). Prefer fixing the pod for `restricted-v2` when possible.

### Auto-fix

**Auto-fix for restricted-v2** rewrites the pasted YAML toward a restricted-friendly shape (drops host namespaces / hostPath-style volumes where needed, strips root UID / privileged, sets drop `ALL` + `NET_BIND_SERVICE`). Review the diff before using it in Git.

---

## Samples

| Sample | What it shows |
|--------|----------------|
| UID 0 root | Explicit root + extra capabilities → denied on `restricted-v2` |
| HostPath | Host filesystem mount → denied |
| hostPort | `hostPort` requires `hostnetwork-v2` / privileged |
| UID out of range | `runAsUser` outside namespace UID range |
| NFS volume | Non-trivial volume type vs restricted volumes |
| Privileged agent | DaemonSet needing `privileged` |
| anyuid priority | Clean pod with `restricted-v2` **and** `anyuid` checked → `anyuid` wins |
| Clean microservice | Passes `restricted-v2` |

---

## Shareable URLs

**Copy share link** (top bar) copies a URL whose hash (`#s=…`) embeds:

- workload YAML  
- namespace, ServiceAccount, UID range, MCS  
- which SCCs are checked  

Anyone opening that link restores the same case. The hash also updates quietly as you edit, so a bookmarked tab stays useful.

Large YAML is gzip-compressed in the hash when the browser supports `CompressionStream`.

---

## What the simulator covers

Stock SCCs: `restricted-v2`, `nonroot-v2`, `hostnetwork-v2`, `anyuid`, `privileged`.

Checks include (among others): hostNetwork / hostPID / hostIPC, hostPort, volume types, privileged, capabilities, `allowPrivilegeEscalation`, runAsUser / MustRunAsRange / MustRunAsNonRoot, fsGroup range, SELinux MCS, seccomp, and typical mutations.

It does **not** fully reimplement OpenShift admission (no live image USER inspection, no custom SCC catalog, no PSA layer, no sysctl matrix, etc.). Treat on-cluster dry-run as source of truth.

---

## Run locally

```bash
git clone https://github.com/vinashar-rh/scc-validator.git
cd scc-validator
# any static server, e.g.:
python3 -m http.server 8080
# open http://localhost:8080
```

Single file app: `index.html` (Tailwind + js-yaml from CDN).

---

## Visit statistics

GitHub Pages does not show hits on the app URL. The page sends an **anonymous pageview** to [GoatCounter](https://www.goatcounter.com/) (no cookies, no YAML, no cluster data). Share-link hashes are stripped so pasted manifests are not uploaded.

**Dashboard:** https://vinashar-scc.goatcounter.com

**One-time setup (required):** counts stay empty until you create that GoatCounter site.

1. Sign up at https://www.goatcounter.com/signup
2. When it asks for a **code**, use exactly `vinashar-scc` (must match the script in `index.html`)
3. Open https://vinashar-scc.goatcounter.com and log in
4. Load the live app once (ad blockers often hide GoatCounter — try a private window if the first hit is missing)

You should see pageviews within a few seconds. Ad blockers can under-count.

If you already used a different GoatCounter code, change `vinashar-scc` in `index.html` to match.

---

## License / trademark note

Personal diagnostic tool. Not an official Red Hat product. OpenShift® is a trademark of Red Hat, Inc.
