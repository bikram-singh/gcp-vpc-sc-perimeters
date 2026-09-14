Securing GCP with VPC Service Controls: IAM Isn't Enough
==========================================================

*Perimeters, VPC networks, access levels, ingress/egress rules, scoped policies, perimeter bridges, and Shared VPC — tested end-to-end in a live GCP organization, with every "why did that just get blocked?" moment documented.*

---

If you've ever been told "IAM isn't enough for sensitive data," VPC Service Controls (VPC SC) is the answer GCP gives you. It's a second, independent security perimeter that sits *underneath* IAM — so even a principal with a valid `Editor` or `Storage Admin` role can be denied if the request comes from the wrong network, the wrong identity type, or the wrong region.

Most explanations of VPC SC stop at the conceptual slide deck. This one doesn't. I built two GCP projects (`trust-project-ok` and `untrust-project-x`), deliberately over-permissioned one against the other, and then spent seven rounds of hands-on testing breaking and fixing access with perimeters, access levels, ingress/egress rules, scoped policies, perimeter bridges, and a Shared VPC topology. Every command and every console click below is something I actually ran — including the failures, because the failures are where VPC SC actually teaches you something.

---

## 🔐 What Is VPC Service Controls?

**VPC Service Controls (VPC SC)** is a GCP security feature that creates a logical perimeter around your resources to mitigate data exfiltration risk — even when IAM permissions are correctly scoped, it prevents access to protected resources from outside an approved context (network, identity, or project boundary).

IAM controls **who** can access a resource. VPC SC controls **from where** that access is allowed to happen — a second, independent layer. Even a principal with valid IAM permissions (say, `roles/bigquery.admin`) can be blocked from copying data out via a compromised credential, a misconfigured public IP, or an insider using a personal account, because the perimeter itself refuses the request.

### Core building blocks

- **Service perimeter** — a boundary around a set of GCP projects (or a VPC network) that restricts communication with supported APIs (BigQuery, Cloud Storage, Pub/Sub, Vertex AI, etc.) to inside the perimeter only.
- **Access levels** — conditions (IP range, device policy, identity) defined via **Access Context Manager** that can allow specific access from *outside* the perimeter when needed.
- **Perimeter bridges** — allow controlled communication *between* two perimeters (e.g., a shared services project talking to an app project).
- **Ingress/egress rules** — fine-grained rules specifying exactly which identities/services can cross the perimeter boundary, in which direction.
- **Dry-run mode** — lets you test a perimeter's rules and see what *would* be blocked, via audit logs, before you enforce it.

### What it protects against

- Exfiltration via stolen or leaked credentials
- Unauthorized copying of data to a personal or external GCP project
- Access to protected APIs from unapproved networks (public internet without an access level)
- Malicious insiders trying to route data outside a compliance boundary (useful for PCI, HIPAA, etc.)

### What it does NOT do

- It doesn't replace IAM — it's an additive perimeter, not an access grant
- It only governs supported services — a defined list, not every GCP API is perimeter-aware
- It doesn't encrypt data or manage keys — that's CMEK/KMS territory

### Typical setup flow

1. Enable the Access Context Manager and VPC SC APIs at the org level
2. Define access levels (trusted IP ranges, corp devices, identity conditions)
3. Create a perimeter in **dry-run mode**, add your projects and restricted services
4. Review violation logs in Cloud Logging over some days/weeks
5. Add necessary ingress/egress rules or bridges for legitimate cross-perimeter flows
6. Switch the perimeter from dry-run to enforced

Everything below is that flow, run for real, against two live projects — including every place it broke.

---

## 🧭 What Problem Is VPC Service Controls Actually Solving?

IAM answers **"who can do what."** VPC SC answers **"from where, and under what context, can that access happen at all."** It's an additive perimeter around Google Cloud services (Cloud Storage, BigQuery, Compute Engine, and others), independent of IAM, designed to mitigate:

- Access from unauthorized networks using stolen credentials
- Data exfiltration by malicious insiders or compromised code
- Public exposure of private data caused by misconfigured IAM
- Unmonitored access to sensitive services

A perimeter can run in two modes:

```
Enforced mode   → requests that violate the perimeter policy are DENIED
Dry run mode    → requests that violate the perimeter policy are only LOGGED
```

Dry run is the safe way to roll out VPC SC in a live environment — you see exactly what *would* have broken before you flip the switch.

---

## 1️⃣ Getting Started: Access Policies & Your First Perimeter

### The setup

Two projects, two VMs, one deliberate problem:

```
trust-project-ok      → trust-vm     (trust-custom-vpc)
untrust-project-x     → untrust-vm   (untrust-custom-vpc)
```

`untrust-project-x`'s default Compute Engine service account had been granted `Editor` on `trust-project-ok`. That's the exact scenario VPC SC exists for: IAM is technically "correct," but the blast radius if that untrusted project is compromised is enormous.

### Gotcha #1 — you need org-level permission before you can even open the page

The very first click into **Security → VPC Service Controls** threw a permissions error:

```
Missing permissions:
  accesscontextmanager.accessLevels.list
  accesscontextmanager.policies.list
  accesscontextmanager.servicePerimeters.list
```

VPC Service Controls is managed through **Access Context Manager**, and Access Context Manager is an organization-level resource. Nothing works until an admin grants the `Access Context Manager Admin` role (or `Organization Admin`) at the **organization** level — not the project level. Once that role was granted, the console lit up with the **Enforced mode / Dry run mode** tabs.

### Create the access policy

Every VPC SC resource — perimeters and access levels — lives inside an **access policy**, which is a top-level container scoped to the whole organization (you can only have one org-level policy; more on scoped policies in Part 5).

```
Access policy title: my-org-policy
Scope: organization (gcpcloudhub.in)
```

### Create the perimeter

```
Title            : secure-from-untrust-project
Perimeter type   : Regular
Resources        : untrust-project-x
Restricted APIs  : compute.googleapis.com, storage.googleapis.com
VPC accessible   : All services
Access levels    : (none yet)
Ingress/Egress   : (none yet)
Enforcement mode : Enforced
```

### The test

From `trust-vm`, using the over-permissioned service account:

```bash
$ gsutil ls gs://untrust-bucket/
AccessDeniedException: 403 Request is prohibited by organization's policy.
vpcServiceControlsUniqueIdentifier: ...
```

SSH-in-browser into `untrust-vm` also failed outright ("Connection Failed — An error occurred while fetching instance or project metadata") because Compute Engine's own control-plane API was inside the restricted-services list.

> ⚠️ **Gotcha:** Restricting `compute.googleapis.com` doesn't just block *your* API calls to Compute Engine — it breaks IAP tunneling and SSH-in-browser too, since those rely on the same API surface. Restrict it deliberately, not by default.

After removing `compute.googleapis.com` from the restricted list (keeping only Cloud Storage), the Cloud Storage console itself still failed to load buckets ("Sorry, the server was not able to fulfil your request") — proof the perimeter was doing exactly what it was told.

Deleting the perimeter entirely restored full access instantly — no propagation delay, no cache to clear.

### Dry run mode

I rebuilt the same perimeter as `dry-run-secure-untrust-project` with `Enforcement mode: Dry run`. Same restricted-services list, same resources — but every one of the commands above succeeded normally. Dry run perimeters are a **decision-support tool**, not a soft version of enforcement: they log violations to Cloud Logging / the Violation Dashboard so you can validate a configuration against real traffic before it ever blocks anything.

✅ **Takeaway:** An access policy is the container. A perimeter is the fence. Enforced mode denies; dry run only logs. Org-level Access Context Manager permission is a hard prerequisite for any of it.

---

## 2️⃣ Protecting a VPC Network Instead of a Project

This is the distinction that trips almost everyone up the first time: a perimeter's **Resources to protect** step lets you choose **Projects** *or* **VPC networks** — and they behave completely differently.

```
Projects     → protects the PROJECT. All API calls that touch that project's
               resources are governed by the perimeter, regardless of where
               the caller sits.

VPC networks → protects the NETWORK. Only API calls that ORIGINATE FROM
               INSIDE that VPC are governed. Calls to the same project's
               resources from outside the VPC are untouched.
```

I rebuilt the perimeter, this time selecting the VPC network directly:

```
Resources to protect : projects/untrust-project-x/global/networks/untrust-custom-vpc
Restricted services  : storage.googleapis.com
```

### The test — same command, two different machines, two different outcomes

**From `untrust-vm` (inside the protected `untrust-custom-vpc`):**
```bash
$ gsutil ls
AccessDeniedException: 403 Request is prohibited by organization's policy.
```

**From `trust-vm` (outside the perimeter entirely, calling the same project):**
```bash
$ gcloud storage buckets list --project=untrust-project-x
creation_time: 2025-05-20T05:26:07+0000
name: untrust-bucket
storage_url: gs://untrust-bucket/
...
```

Identical bucket, identical project — one call denied, one call allowed, purely because of where the request physically originated. This is the scenario the concept slides describe as: *"you have 5 VPCs across dev/prod, and you want no one from development to reach a production Cloud Storage bucket"* — VPC-network-scoped perimeters are exactly the tool for that, and they don't require touching IAM at all.

Adding `compute.googleapis.com` back to the restricted list then blocked Compute Engine API calls originating from inside `untrust-custom-vpc` the same way.

✅ **Takeaway:** Project-scoped perimeters protect the resource. VPC-scoped perimeters protect the network boundary. Pick based on whether the risk is "who can reach this project" or "what can this network reach out to."

---

## 3️⃣ Access Levels & Access Context Manager

A perimeter by itself is binary: in the perimeter or not. **Access levels** add context — IP range, geography, device policy (Chrome Enterprise Premium) — as a condition a request must satisfy to be treated as trusted, even from outside the perimeter.

### IP-subnet-based access level

```
Access level title : access-context-level
Mode                : Basic
Condition           : IP subnetworks → Private IP
                       VPC network: trust-project-ok / trust-custom-vpc (all subnets)
When condition met  : return True
```

Attached to the `untrust-project-x` perimeter (restricted services: `bigquery.googleapis.com`, `storage.googleapis.com`), the access level allowed exactly one VM's subnet range through:

```bash
# From a VM inside trust-custom-vpc's allowed subnet
$ gcloud storage buckets list --project=untrust-project-x   # ✅ succeeds

# From a second VM NOT in the allow-listed subnet
$ gcloud storage buckets list --project=untrust-project-x   # ❌ denied
```

Removing the access level from the perimeter immediately reverted the first VM back to denied — access levels are purely additive; there's no fallback.

### Geography-based access level

Same access level, new condition:

```
+ Geographic locations: US (United States)
```

```bash
# VM in us-central1                → ✅ "Allow from US"
# VM in asia-south1 (same project)  → ❌ "Not allowed from other region"
```

Flipping **"When condition is met, return"** from `True` to `False` inverts the logic entirely — instead of *allow only from US*, it becomes *allow everything except US*. It's easy to misread this toggle in the console, so I'd recommend naming your access levels with the polarity baked in (`allow-us-only` vs `block-us`) rather than relying on remembering which way you set it.

### Combining conditions and access levels

I also built a second access level, `allow-public-ip`, scoped to a specific public CIDR block, and then referenced the region-based access level as an **access level dependency** inside it — meaning a request now has to satisfy *both* conditions (public IP range **and** region) to be treated as trusted. Access levels can be composed this way to build genuinely fine-grained, defense-in-depth conditions without touching the perimeter's core resource/service configuration at all.

✅ **Takeaway:** Access levels are reusable, composable trust conditions (IP, geography, device policy, or other access levels) that you attach to a perimeter's ingress path. They don't replace the perimeter — they're the exception mechanism layered on top of it.

---

## 4️⃣ Ingress and Egress Rules

Access levels answer *"is this request trustworthy enough to get in?"* — but they're evaluated at the perimeter boundary as a whole. **Ingress and egress rules** are more surgical: each rule pairs a **source** (identity + network/project/access-level) with a **destination** (specific project) and a **specific set of API operations**, so you can allow exactly one narrow flow across the boundary instead of opening the whole perimeter.

### Building an ingress rule

```
Rule name  : Ingress rule 1
From
  Identities : Any identity
  Sources    : All sources
To
  Resources  : untrust-project-x
  Operations : storage.googleapis.com (all methods)
```

Before this rule existed, a call from `trust-vm` to `untrust-project-x`'s bucket was denied outright. After adding it:

```bash
$ gcloud storage buckets list --project=untrust-project-x
name: untrust-bucket
...                                                          # ✅ now succeeds
```

### Narrowing the identity type

I then edited the same rule and changed **Identities** from `Any identity` to `Any user account` (excluding service accounts). Immediately, the exact same call — now running as the VM's default *service account* — was denied again:

```
AccessDeniedException: 403 ... not allowed to access the resource because
it's not permitted by the currently active service perimeter
```

> ⚠️ **Gotcha:** Ingress rules distinguish between **user accounts** and **service accounts** as separate identity classes. A rule written for "any identity" silently includes both — if you only mean human operators, be explicit, or your service accounts will fail in ways that look like a completely unrelated bug.

### Egress rules

Egress works the same way in reverse — governing calls that originate *inside* a perimeter and reach *outside* it (or out to the public internet):

```
Rule name  : egress
From
  Identities : Any identity
  Sources    : trust-project-ok
To
  Resources  : All projects (Google Cloud resources)
  Operations : storage.googleapis.com
```

✅ **Takeaway:** Access levels gate *who gets past the fence*; ingress/egress rules gate *which specific API calls, for which specific identities, are allowed to cross it in each direction*. Use rules when a bridge or a blanket access level would be too coarse.

---

## 5️⃣ Scoped Policies — Delegating VPC SC to a Folder or Project

An organization can only have **one** org-level access policy. But large organizations don't want every perimeter change to require org-admin approval — so GCP offers **scoped policies**: access policies whose scope is a single folder or project, used to delegate perimeter/access-level administration downward.

```
untrust-project-policy  → scope: untrust-project-x (project-scoped)
test-folder-policy      → scope: foldergcpcloudhub (folder-scoped)
```

### Gotcha #2 — a project can only belong to one policy, period

Trying to add a project that was already governed by another access policy failed immediately:

```
Service perimeter creation failed
Policy: accessPolicies/244775374082 already contains resource:
projects/300702408885 or its children, thus the resource or its children
cannot be added to policy: accessPolicies/106482310921.
```

This happened twice in the lab — once between two scoped policies, once between a folder-scoped policy and the org-level policy. **A project (or its parent folder) can be a member of exactly one service-perimeter policy across the entire organization**, whether that policy is org-level or scoped. Plan your folder/policy hierarchy before you start creating perimeters, because you can't have overlapping coverage.

### Gotcha #3 — access levels don't cross policy boundaries

An access level created under the **org-level** policy (`my-access-level`) was completely invisible when editing a perimeter that belonged to the **folder-scoped** policy — the "Add access levels" list showed zero rows. Each access policy (org or scoped) has its own separate namespace of access levels; there's no inheritance. The fix was simply to recreate the equivalent access level directly under the folder-scoped policy.

### Gotcha #4 — enforcement can lag without a real org-level policy backing the scope

Early in this test, a folder-scoped perimeter (`folder-test`, protecting `first-project`, restricting `storage.googleapis.com`, Enforced) was in place — and yet the target bucket was still fully browsable. It only started returning **"Denied by VPC security control policy"** once the organization's policy hierarchy was fully in place end-to-end. If a scoped-policy perimeter appears to be doing nothing, don't assume the perimeter config is wrong first — check that the org-level policy prerequisite documented in Part 1 is actually satisfied for that resource's scope.

✅ **Takeaway:** Scoped policies are how you delegate VPC SC administration without granting org-wide Access Context Manager permissions. The tradeoffs: one policy per project, and zero automatic sharing of access levels across policy boundaries.

---

## 6️⃣ Perimeter Bridges — Letting Two Perimeters Talk to Each Other

With `trust-project-ok` and `untrust-project-x` each sitting inside their **own** regular perimeter (`protect-trust-project` and `protect-untrust-project`, both restricting `storage.googleapis.com`), cross-project access was denied **in both directions** — each perimeter protects its own project, full stop, even against another protected project.

```bash
# From trust-vm  → untrust-project-x's bucket   → ❌ denied
# From untrust-vm → trust-project-ok's bucket    → ❌ denied
```

A **perimeter bridge** is a third, special perimeter type that creates bidirectional access between two *existing regular* perimeters, without merging their access levels or restricted-service lists:

```
Title             : bridge
Perimeter type    : Bridge
Resources         : trust-project-ok, untrust-project-x
Enforcement mode  : Enforced
```

```bash
# Same commands as above, run again after the bridge was created → ✅ both succeed
```

### Rules worth internalizing before you reach for a bridge

- A project **must already belong to a regular perimeter** before it can be added to a bridge — bridges connect perimeters, not bare projects.
- Bridges are **bidirectional only** — you can't build a one-way bridge (use ingress/egress rules for asymmetric access instead).
- Bridges **cannot cross organizations**, and **cannot cross scoped policies** — for that, ingress/egress rules are the only option.
- Once a project is in a bridge, you can no longer add its VPC networks to a *different* perimeter.
- A project can have multiple bridges to multiple other projects, but access levels and restricted services always remain governed solely by the project's own regular perimeter — a bridge only opens the door, it doesn't relax what's on the other side of it.

✅ **Takeaway:** Reach for a bridge when two already-protected projects need routine bidirectional access to each other's restricted services. Reach for ingress/egress rules when the access needs to be asymmetric, narrower, or cross a scoped-policy boundary.

---

## 7️⃣ VPC Service Controls with Shared VPC

The final scenario: a **Shared VPC** host project (`host-project-333`) exposing three environment-specific networks — `dev-vpc`, `prod-vpc`, `staging-vpc` — to two service projects, `trust-project-ok` and `untrust-project-x`, each running VMs on all three networks.

```
host-project-333
 ├── dev-vpc      (subnet-asia, asia-south1)
 ├── prod-vpc     (subnet-eu,   europe-west1)
 └── staging-vpc  (subnet-us,   us-central1)

Attached service projects: trust-project-ok, untrust-project-x
```

The key capability this part demonstrates: you can build **one perimeter per VPC network inside a Shared VPC host project**, instead of one perimeter for the whole host project. That means dev, prod, and staging can each carry a completely different VPC SC posture even though they share a host.

```
Perimeter        : test-network
Resources        : host-project-333 (project) + dev-vpc (network only)
Restricted APIs  : storage.googleapis.com
Enforcement mode : Enforced
```

### The test

```bash
# From dev-vm (on the now-protected dev-vpc)
$ gsutil ls gs://trust-dev-bucket-333/       # ✅ before the perimeter existed
$ gsutil ls gs://trust-dev-bucket-333/       # ❌ after: AccessDeniedException 403
```

VMs on `prod-vpc` and `staging-vpc` — same host project, same organization, same Shared VPC — were completely unaffected, because the perimeter's **Resources to protect** explicitly scoped it to `dev-vpc` and nothing else.

### Restoring access surgically with an egress rule

Rather than removing the perimeter, I added an egress rule scoped to the service account making the call:

```
Rule name  : allow-storage-bucket-rule
From
  Identities : Any service account
  Sources    : trust-project-ok
To
  Resources  : trust-project-ok
  Operations : bigquery.googleapis.com, storage.googleapis.com
```

```bash
$ gsutil ls gs://trust-dev-bucket-333/
gs://trust-dev-bucket-333/                                  # ✅ restored
```

### Rules for VPC networks inside a perimeter

- If the host project itself is **not** already protected by a perimeter, its individual VPC networks can be split across separate perimeters under the same access policy.
- All VPC networks belonging to the same host project must sit under the **same** access policy.
- A host project and its VPC networks must not be split across **different** perimeters simultaneously in a conflicting way — a network and its parent project can't end up in two places.
- A VPC network **cannot** belong to more than one perimeter.
- A VPC network **cannot** be used inside a perimeter **bridge** — bridges only operate at the project level.
- If a VPC network's parent project is part of a bridge, that network can no longer be added to a regular perimeter.

✅ **Takeaway:** Shared VPC and VPC SC compose cleanly down to the individual network level — you're not forced to give every environment sharing a host project the same security posture. Combine network-scoped perimeters with narrow egress rules to keep least-privilege intact without duplicating host projects per environment.

---

## 🧾 The Gotchas, All in One Place

```
1. Access Context Manager is an ORG-level resource — grant the role at the
   organization, not the project, or the console won't even load.

2. Restricting compute.googleapis.com also breaks IAP/SSH-in-browser —
   restrict it deliberately.

3. Project-scoped perimeters ≠ VPC-network-scoped perimeters. One protects
   the resource regardless of caller location; the other protects calls
   made FROM inside that network.

4. Access levels don't stack automatically across True/False polarity —
   name them so the direction of the condition is obvious.

5. A project (or folder) can belong to exactly ONE access-policy perimeter
   across the whole org — org-level or scoped, never both, never two scoped
   policies at once.

6. Access levels are scoped to the access policy they were created under —
   an org-level access level is invisible to a folder-scoped perimeter's
   configuration UI, and vice versa.

7. Ingress rules treat "user account" and "service account" as distinct
   identity classes — "any identity" quietly includes both.

8. Perimeter bridges require both projects to already sit inside a regular
   perimeter, are bidirectional-only, and can't cross orgs or scoped
   policies.

9. A VPC network can't be added to a bridge, and can't belong to more than
   one perimeter — plan Shared VPC network-level perimeters carefully.
```

---

## Closing Thoughts

VPC Service Controls rewards hands-on testing more than almost any other GCP security feature, precisely because so much of its behavior is *contextual* — the same `gsutil ls` command succeeds or fails depending on which VM ran it, which network it's attached to, which policy governs the target project, and which rule (if any) opened a narrow exception. None of that is visible from IAM alone.

If you're rolling this out on a production organization, my one hard recommendation: **build everything in Dry Run mode first**, review the Violation Dashboard against real traffic for a few days, and only then flip to Enforced. The gotchas above are exactly the kind of thing that show up as a 3 a.m. page if you enforce blind.

---

*If this was useful, the companion GitHub repository has the full README with an architecture diagram and a condensed reference table of every command shown above.*
