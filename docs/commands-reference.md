# 🧰 Command Reference — GCP VPC Service Controls Lab

Everything in this lab was built through the **GCP Console** (see `docs/snapshots/`). This file gives the `gcloud` CLI equivalent for every configuration step, plus every verification command actually run against the perimeters, organized by the same 7 parts as the README and Medium article.

> ⚠️ Replace `ORG_ID`, `POLICY_ID`, `PROJECT_ID` / `PROJECT_NUMBER`, and network/subnet names with your own values. Access-policy and perimeter IDs are org-specific — run `gcloud access-context-manager policies list --organization=ORG_ID` first to find yours. Flag names can shift slightly between `gcloud` versions — run `gcloud access-context-manager perimeters create --help` if a command below doesn't match your installed version.

---

## 0️⃣ Prerequisites

```bash
# Enable the required APIs
gcloud services enable accesscontextmanager.googleapis.com --project=PROJECT_ID

# Grant Access Context Manager Admin at the ORGANIZATION level (console-only
# in this lab, but this is the CLI equivalent)
gcloud organizations add-iam-policy-binding ORG_ID \
  --member="user:admin@example.com" \
  --role="roles/accesscontextmanager.policyAdmin"
```

---

## 1️⃣ First Perimeter & Access Policies

```bash
# Create the org-level access policy (only one per organization)
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --title="my-org-policy"

# Note the returned policy ID, then export it for convenience
export POLICY_ID=1234567890

# Create the perimeter — enforced mode, protecting a project
gcloud access-context-manager perimeters create secure_from_untrust_project \
  --title="secure-from-untrust-project" \
  --perimeter-type=regular \
  --policy=$POLICY_ID \
  --resources=projects/UNTRUST_PROJECT_NUMBER \
  --restricted-services=compute.googleapis.com,storage.googleapis.com

# Same perimeter in DRY RUN — the dry-run spec is managed separately
gcloud access-context-manager perimeters dry-run create dry_run_secure_untrust_project \
  --title="dry-run-secure-untrust-project" \
  --policy=$POLICY_ID \
  --resources=projects/UNTRUST_PROJECT_NUMBER \
  --restricted-services=storage.googleapis.com

# Delete a perimeter (used to restore access during testing)
gcloud access-context-manager perimeters delete secure_from_untrust_project \
  --policy=$POLICY_ID
```

**Verification commands (run from the test VMs):**

```bash
# From trust-vm, against the protected untrust-project-x bucket
gsutil ls gs://untrust-bucket/
# AccessDeniedException: 403 Request is prohibited by organization's policy.

gcloud storage buckets list --project=untrust-project-x
```

---

## 2️⃣ Project vs VPC-Network-Scoped Perimeters

```bash
# Protect a VPC NETWORK instead of a project — note the resource format
gcloud access-context-manager perimeters update secure_from_untrust_project \
  --policy=$POLICY_ID \
  --add-resources=//compute.googleapis.com/projects/UNTRUST_PROJECT_NUMBER/global/networks/untrust-custom-vpc
```

**Verification:**

```bash
# From INSIDE the protected VPC network (untrust-vm)
gsutil ls
# AccessDeniedException: 403 Request is prohibited by organization's policy.

# From OUTSIDE the perimeter (trust-vm), calling the same project — allowed
gcloud storage buckets list --project=untrust-project-x
```

---

## 3️⃣ Access Levels & Access Context Manager

Access levels are defined declaratively in a YAML "basic level spec" file, then created/attached with `gcloud`.

```yaml
# level-spec.yaml — IP-range + region condition
- ipSubnetworks:
    - "10.10.0.0/24"    # trust-custom-vpc subnet range
  regions:
    - "US"
```

```bash
gcloud access-context-manager levels create access_context_level \
  --title="access-context-level" \
  --basic-level-spec=level-spec.yaml \
  --combine-function=AND \
  --policy=$POLICY_ID

# Attach the access level to the perimeter
gcloud access-context-manager perimeters update untrustprojectx \
  --policy=$POLICY_ID \
  --add-access-levels=access_context_level

# List / describe access levels
gcloud access-context-manager levels list --policy=$POLICY_ID
gcloud access-context-manager levels describe access_context_level --policy=$POLICY_ID

# Remove an access level from a perimeter
gcloud access-context-manager perimeters update untrustprojectx \
  --policy=$POLICY_ID \
  --remove-access-levels=access_context_level
```

**Verification:**

```bash
# From a VM inside the allow-listed subnet, in us-central1
gcloud storage buckets list --project=untrust-project-x     # ✅ allowed

# From a VM outside the subnet, or in asia-south1
gcloud storage buckets list --project=untrust-project-x     # ❌ denied
```

---

## 4️⃣ Ingress & Egress Rules

Ingress and egress rules are also defined as YAML/JSON and applied to a perimeter.

```yaml
# ingress-policy.yaml
- ingressFrom:
    identityType: ANY_IDENTITY      # or ANY_USER_ACCOUNT / ANY_SERVICE_ACCOUNT
    sources:
      - accessLevel: "*"
  ingressTo:
    resources:
      - "projects/UNTRUST_PROJECT_NUMBER"
    operations:
      - serviceName: "storage.googleapis.com"
        methodSelectors:
          - method: "*"
```

```bash
gcloud access-context-manager perimeters update untrustprojectx \
  --policy=$POLICY_ID \
  --set-ingress-policies=ingress-policy.yaml
```

```yaml
# egress-policy.yaml
- egressFrom:
    identityType: ANY_IDENTITY
  egressTo:
    resources:
      - "projects/TRUST_PROJECT_NUMBER"
    operations:
      - serviceName: "storage.googleapis.com"
        methodSelectors:
          - method: "*"
```

```bash
gcloud access-context-manager perimeters update untrustprojectx \
  --policy=$POLICY_ID \
  --set-egress-policies=egress-policy.yaml
```

**Verification:**

```bash
# Before the ingress rule
gcloud storage buckets list --project=untrust-project-x     # ❌ denied

# After the ingress rule (identityType: ANY_IDENTITY)
gcloud storage buckets list --project=untrust-project-x     # ✅ allowed

# After narrowing to ANY_USER_ACCOUNT, run from the VM's service account
gcloud storage buckets list --project=untrust-project-x     # ❌ denied again
```

---

## 5️⃣ Scoped Policies

```bash
# Create a policy scoped to a single PROJECT
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --title="untrust-project-policy" \
  --scopes=projects/UNTRUST_PROJECT_NUMBER

# Create a policy scoped to a single FOLDER
gcloud access-context-manager policies create \
  --organization=ORG_ID \
  --title="test-folder-policy" \
  --scopes=folders/FOLDER_ID

# Build a perimeter under the scoped policy exactly like a normal one —
# just point --policy at the scoped policy's ID
export SCOPED_POLICY_ID=9876543210
gcloud access-context-manager perimeters create folder_test \
  --title="folder-test" \
  --policy=$SCOPED_POLICY_ID \
  --resources=projects/FIRST_PROJECT_NUMBER \
  --restricted-services=storage.googleapis.com
```

> ⚠️ Attempting to add a project that already belongs to another policy (org-level or scoped) fails with `PERMISSION_DENIED: ... already contains resource`. A project/folder can belong to exactly **one** access-policy perimeter, org-wide.

---

## 6️⃣ Perimeter Bridges

```bash
# Both projects must already be in a regular perimeter first, e.g.:
gcloud access-context-manager perimeters create protect_trust_project \
  --title="protect-trust-project" --policy=$POLICY_ID \
  --resources=projects/TRUST_PROJECT_NUMBER \
  --restricted-services=storage.googleapis.com

gcloud access-context-manager perimeters create protect_untrust_project \
  --title="protect-untrust-project" --policy=$POLICY_ID \
  --resources=projects/UNTRUST_PROJECT_NUMBER \
  --restricted-services=storage.googleapis.com

# Create the bridge connecting both
gcloud access-context-manager perimeters create bridge \
  --title="bridge" \
  --perimeter-type=bridge \
  --policy=$POLICY_ID \
  --resources=projects/TRUST_PROJECT_NUMBER,projects/UNTRUST_PROJECT_NUMBER
```

**Verification:**

```bash
# Cross-project access, both directions, before the bridge → denied
# After the bridge is created → both succeed
gcloud storage buckets list --project=untrust-project-x   # from trust-vm
gcloud storage buckets list --project=trust-project-ok    # from untrust-vm
```

---

## 7️⃣ VPC Service Controls with Shared VPC

```bash
# Protect ONE network inside a Shared VPC host project (host + dev-vpc only)
gcloud access-context-manager perimeters create test_network \
  --title="test-network" \
  --policy=$POLICY_ID \
  --resources=projects/HOST_PROJECT_NUMBER,//compute.googleapis.com/projects/HOST_PROJECT_NUMBER/global/networks/dev-vpc \
  --restricted-services=storage.googleapis.com

# Restore access surgically with an egress rule scoped to a service account
# (egress-policy.yaml as in Part 4, scoped to the specific service account
# and trust-project-ok's storage/bigquery APIs)
gcloud access-context-manager perimeters update test_network \
  --policy=$POLICY_ID \
  --set-egress-policies=egress-policy.yaml
```

**Verification:**

```bash
# From a VM on dev-vpc (protected)
gsutil ls gs://trust-dev-bucket-333/     # ❌ denied once the perimeter is live

# From a VM on prod-vpc / staging-vpc (same host project, NOT protected)
gsutil ls gs://<their-bucket>/           # ✅ unaffected

# After the egress rule is applied
gsutil ls gs://trust-dev-bucket-333/     # ✅ restored
```

---

## 🔎 Useful Read-Only Commands

```bash
# List every perimeter under a policy
gcloud access-context-manager perimeters list --policy=$POLICY_ID

# Describe a specific perimeter's full config
gcloud access-context-manager perimeters describe PERIMETER_ID --policy=$POLICY_ID

# List every access policy visible to you (org + scoped)
gcloud access-context-manager policies list --organization=ORG_ID

# Check violations while a perimeter runs in dry-run mode
# (Console: VPC Service Controls → Violation Dashboard / Audit Logs)
gcloud logging read \
  'protoPayload.metadata."@type"="type.googleapis.com/google.cloud.audit.VpcServiceControlAuditMetadata"' \
  --project=PROJECT_ID --limit=20
```
