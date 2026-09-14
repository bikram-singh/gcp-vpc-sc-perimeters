# 🛡️ Securing GCP with VPC Service Controls: IAM Isn't Enough

![GCP](https://img.shields.io/badge/Google%20Cloud-VPC%20Service%20Controls-4285F4?logo=googlecloud&logoColor=white)
![Status](https://img.shields.io/badge/status-complete-brightgreen)
![Type](https://img.shields.io/badge/type-hands--on%20lab-blueviolet)
![Parts](https://img.shields.io/badge/parts-7-orange)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

A seven-part, console-driven hands-on lab documenting how **VPC Service Controls (VPC SC)** actually behaves in a live GCP organization — perimeters, VPC-network-scoped perimeters, access levels, ingress/egress rules, scoped policies, perimeter bridges, and Shared VPC — with every test command, every denial, and every fix captured as a real screenshot.

- 📖 **Read the Full Article:** [Securing GCP with VPC Service Controls: IAM Isn't Enough](https://github.com/bikram-singh/gcp-vpc-sc-perimeters/blob/main/vpc-service-controls-medium-article.md)
- 🧰 **Command Reference:** [gcloud CLI equivalents for every part](https://github.com/bikram-singh/gcp-vpc-sc-perimeters/blob/main/docs/commands-reference.md)

---

## 🔐 What Is VPC Service Controls?

**VPC Service Controls (VPC SC)** is a GCP security feature that creates a logical perimeter around your resources to mitigate data exfiltration risk — even when IAM permissions are correctly scoped, it prevents access to protected resources from outside an approved context (network, identity, or project boundary).

IAM controls *who* can access a resource. VPC SC controls *from where* that access is allowed to happen — a second, independent layer. Even a user with valid IAM permissions can be blocked from copying data out via a compromised credential, a misconfigured public IP, or an insider using a personal account, because the perimeter itself refuses the request.

### Core building blocks

| Concept | What it does |
|---|---|
| **Service perimeter** | A boundary around a set of GCP projects (or a VPC network) that restricts communication with supported APIs (BigQuery, Cloud Storage, Pub/Sub, Vertex AI, etc.) to inside the perimeter only |
| **Access levels** | Conditions (IP range, device policy, identity) defined via Access Context Manager that allow specific access from *outside* the perimeter when needed |
| **Perimeter bridges** | Controlled communication *between* two perimeters (e.g., a shared services project talking to an app project) |
| **Ingress/egress rules** | Fine-grained rules specifying exactly which identities/services can cross the perimeter boundary, in which direction |
| **Dry-run mode** | Lets you test a perimeter's rules and see what *would* be blocked, via audit logs, before you enforce it |

### What it protects against

- Exfiltration via stolen or leaked credentials
- Unauthorized copying of data to a personal or external GCP project
- Access to protected APIs from unapproved networks
- Malicious insiders trying to route data outside a compliance boundary (PCI, HIPAA, etc.)

### What it does NOT do

- It doesn't replace IAM — it's an additive perimeter, not an access grant
- It only governs supported services — not every GCP API is perimeter-aware
- It doesn't encrypt data or manage keys — that's CMEK/KMS territory

This lab is the hands-on companion to those concepts — every claim above is tested against a real perimeter in the walkthrough below.

---

## 📌 Why This Repo Exists

IAM answers *who can do what*. VPC Service Controls answers *from where, and under what context, can that access happen at all* — an independent, additive perimeter around Google Cloud APIs that protects against stolen credentials, malicious insiders, and misconfigured IAM, even when every IAM binding involved is technically correct.

Instead of just summarizing the docs, this repo walks through **two deliberately over-permissioned projects** and proves, command by command, what VPC SC does and doesn't stop.

```
trust-project-ok      →  the "trusted" side, holds Editor access on itself
untrust-project-x     →  granted Editor on trust-project-ok's service account
                          (the exact misconfiguration VPC SC protects against)
```

---

## 🗺️ Architecture Overview

```
                         Organization: gcpcloudhub.in
                                    │
                    ┌───────────────┴────────────────┐
                    │        Access Context Manager    │
                    │   (org-level access policy: my-org-policy)
                    └───────────────┬────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
 ┌──────▼───────┐           ┌───────▼───────┐           ┌───────▼───────┐
 │ trust-project-│           │untrust-project-│           │ Shared VPC     │
 │      ok        │◄──bridge─►│      x        │           │ host-project-333│
 │ trust-custom- │  ingress/ │untrust-custom- │           │  ├─ dev-vpc    │
 │     vpc        │  egress  │     vpc        │           │  ├─ prod-vpc   │
 └───────────────┘   rules   └───────────────┘           │  └─ staging-vpc│
                                                            └───────┬───────┘
                                                                    │ attached
                                                        ┌───────────┴───────────┐
                                                        │ trust-project-ok       │
                                                        │ untrust-project-x      │
                                                        │ (service projects)     │
                                                        └───────────────────────┘

  Perimeter types used:      Regular  →  protects a project or a VPC network
                              Bridge   →  bidirectional link between 2 regular perimeters
  Access controls used:      Restricted services · VPC accessible services ·
                              Access levels (IP / geo) · Ingress rules · Egress rules
```

---

## 📚 What's Covered

| # | Topic | Core question it answers |
|---|-------|---------------------------|
| 1️⃣ | [Access Policies & Your First Perimeter](#1️⃣-access-policies--first-perimeter) | How do I stand up VPC SC from zero, and what's Dry Run vs Enforced? |
| 2️⃣ | [Project vs VPC-Network-Scoped Perimeters](#2️⃣-project-vs-vpc-network-scoped-perimeters) | Do I protect the project, or the network the traffic comes from? |
| 3️⃣ | [Access Levels & Access Context Manager](#3️⃣-access-levels--access-context-manager) | How do I allow trusted exceptions by IP range or geography? |
| 4️⃣ | [Ingress & Egress Rules](#4️⃣-ingress--egress-rules) | How do I open one narrow, identity-specific flow across the boundary? |
| 5️⃣ | [Scoped Policies](#5️⃣-scoped-policies) | How do I delegate perimeter admin to a folder/project owner? |
| 6️⃣ | [Perimeter Bridges](#6️⃣-perimeter-bridges) | How do two already-protected projects talk to each other? |
| 7️⃣ | [VPC Service Controls with Shared VPC](#7️⃣-vpc-service-controls-with-shared-vpc) | Can dev/prod/staging networks in one host project have different postures? |
| 8️⃣ | 📸 [Snapshots](https://github.com/bikram-singh/gcp-vpc-sc-perimeters/tree/main/docs/snapshots) | All GCP Console screenshots for every part |

Full narrative walkthrough with every test command and screenshot context lives in the [Medium article](https://github.com/bikram-singh/gcp-vpc-sc-perimeters/blob/main/vpc-service-controls-medium-article.md). This README summarizes the reference configuration and the gotchas.

---

## ✅ Prerequisites

| Requirement | Why |
|---|---|
| GCP organization (not just a standalone project) | Access Context Manager is an org-level resource |
| `Access Context Manager Admin` or `Organization Admin` role — granted **at the org level** | Required before the VPC Service Controls console page will even load resources |
| At least 2 test projects | To meaningfully test cross-project ingress/egress and bridges |
| `gcloud` / `gsutil` access via Cloud Shell or SSH-in-browser on a test VM | Used for every verification step in this lab |

---

## 1️⃣ Access Policies & First Perimeter

```
Access policy   : my-org-policy               (org-level, one per organization)
Perimeter       : secure-from-untrust-project  (Regular, Enforced)
Protects        : untrust-project-x
Restricts       : compute.googleapis.com, storage.googleapis.com
```

**Result:** `gsutil ls` and SSH-in-browser both denied from outside the perimeter with `403 Request is prohibited by organization's policy`. Rebuilt as `dry-run-secure-untrust-project` (Dry Run mode) — identical commands succeeded, violations logged instead of blocked.

> ⚠️ Restricting `compute.googleapis.com` also breaks IAP tunneling / SSH-in-browser for that project — restrict it deliberately.

---

## 2️⃣ Project vs VPC-Network-Scoped Perimeters

| Perimeter scope | What's protected | Effect |
|---|---|---|
| **Project** | The project's resources | Every caller, regardless of location, is governed |
| **VPC network** | Traffic originating from that network | Only calls *from inside* the network are governed — calls to the same project from elsewhere are untouched |

Tested with `untrust-custom-vpc` as the protected resource, restricting `storage.googleapis.com`:

```bash
# From inside untrust-custom-vpc
$ gsutil ls                                             # ❌ denied

# From trust-vm, outside the perimeter, same target project
$ gcloud storage buckets list --project=untrust-project-x  # ✅ succeeds
```

---

## 3️⃣ Access Levels & Access Context Manager

Access levels add contextual exceptions to a perimeter — evaluated at the boundary, on top of the restricted-services list.

```
access-context-level
 ├─ IP subnetworks   : trust-project-ok / trust-custom-vpc (all subnets)
 └─ Geographic loc.  : US (United States)
```

| Test | Result |
|---|---|
| VM inside allow-listed subnet, us-central1 | ✅ allowed |
| VM outside allow-listed subnet | ❌ denied |
| VM in asia-south1 (same project) | ❌ denied — geo condition |
| Condition polarity flipped `True → False` | Logic inverts entirely |

Access levels can reference other access levels as **dependencies**, letting you AND multiple conditions (e.g., public-IP range *and* region) without touching the perimeter's core config.

---

## 4️⃣ Ingress & Egress Rules

More surgical than access levels — each rule pairs a specific **source identity** with a specific **destination project** and a specific **set of API operations**.

```
Ingress rule 1
 From: Any identity, All sources
 To:   untrust-project-x  →  storage.googleapis.com (all methods)
```

Narrowing `Any identity` to `Any user account` (excluding service accounts) immediately denied the same call when run as a service account — user accounts and service accounts are distinct identity classes to VPC SC ingress rules.

Egress rules mirror this in the outbound direction (from inside the perimeter, out to another project or external resources).

---

## 5️⃣ Scoped Policies

An organization can have exactly **one** org-level access policy — but folders/projects can get their own **scoped policy** for delegated VPC SC administration.

```
untrust-project-policy   → scoped to project: untrust-project-x
test-folder-policy       → scoped to folder:  foldergcpcloudhub
```

### ⚠️ Key gotchas

| Gotcha | Detail |
|---|---|
| **One policy per project** | A project (or its folder) can belong to exactly one access-policy perimeter org-wide — org-level or scoped, never both, never two scoped policies. Violating this returns `Service perimeter creation failed: ... already contains resource`. |
| **Access levels don't cross policies** | An access level created under the org-level policy is invisible when editing a perimeter under a folder-scoped policy, and vice versa — each policy has its own namespace. |
| **Scoped-only enforcement can appear inert** | A folder-scoped perimeter didn't block bucket access until the org-level policy prerequisite was fully satisfied end-to-end for that resource's scope. |

---

## 6️⃣ Perimeter Bridges

Two regular perimeters, each protecting its own project and restricting `storage.googleapis.com`, deny **all** cross-project access by default — even between two protected projects. A **Bridge** perimeter opens bidirectional access between them without merging their access levels or restricted-service lists.

```
Perimeter type : Bridge
Resources      : trust-project-ok, untrust-project-x
```

### Rules for bridges

- Both projects must **already** belong to a regular perimeter first
- Bridges are **bidirectional only**
- Bridges **cannot cross organizations or scoped policies**
- A project in a bridge can't have its VPC networks added to a different perimeter

---

## 7️⃣ VPC Service Controls with Shared VPC

```
host-project-333
 ├── dev-vpc       (asia-south1)
 ├── prod-vpc      (europe-west1)
 └── staging-vpc   (us-central1)

Attached service projects: trust-project-ok, untrust-project-x
```

A perimeter can protect **one VPC network** inside a Shared VPC host project, leaving the other networks (and their environments) with a completely different security posture:

```
Perimeter : test-network
Resources : host-project-333 (project) + dev-vpc (network only)
Restricts : storage.googleapis.com
```

`dev-vpc` traffic was denied; `prod-vpc` and `staging-vpc` on the same host project were unaffected. Access was then restored surgically with an egress rule scoped to a specific service account, rather than removing the perimeter.

### Rules for VPC networks in a perimeter

- All VPC networks under the same host project must sit under the **same** access policy
- A VPC network **cannot** belong to more than one perimeter
- A VPC network **cannot** be used inside a perimeter **bridge**
- If a network's parent project is already in a bridge, that network can't be added to a regular perimeter

---

8️⃣📸 [Snapshots](https://github.com/bikram-singh/gcp-vpc-sc-perimeters/tree/main/docs/snapshots)

---

## 🧾 Full Gotcha Reference

```
1. Grant Access Context Manager Admin at the ORGANIZATION level, not the project.
2. Restricting compute.googleapis.com breaks IAP/SSH-in-browser too.
3. Project-scoped perimeters protect the resource; VPC-scoped perimeters protect
   calls originating FROM that network.
4. Name access levels so their True/False polarity is unambiguous.
5. A project/folder belongs to exactly ONE access-policy perimeter, org-wide.
6. Access levels are namespaced per access policy — no cross-policy visibility.
7. Ingress rules treat user accounts and service accounts as separate identity
   classes — "Any identity" quietly includes both.
8. Bridges need both projects already in a regular perimeter, are bidirectional-
   only, and cannot cross orgs or scoped policies.
9. A VPC network can't join a bridge and can't belong to more than one perimeter.
```

---

## 📁 Repository Structure

```
.
├── README.md
├── vpc-service-controls-medium-article.md
└── docs/
    ├── commands-reference.md               (gcloud CLI equivalent for every part)
    └── snapshots/
        ├── part1-first-perimeter/
        ├── part2-vpc-network-perimeter/
        ├── part3-access-levels/
        ├── part4-ingress-egress/
        ├── part5-scoped-policies/
        ├── part6-perimeter-bridge/
        └── part7-shared-vpc/
```

Each screenshot is a real GCP Console (or Cloud Shell / SSH-in-browser) capture from the lab — conceptual slide images from the source deck were excluded.

---

## 📋 Lab Topics Covered

| Part | Topic |
|---|---|
| 1 | First perimeter & access policies |
| 2 | VPC-network-scoped perimeters |
| 3 | Access levels |
| 4 | Ingress / egress rules |
| 5 | Scoped policies |
| 6 | Perimeter bridges |
| 7 | Shared VPC |
