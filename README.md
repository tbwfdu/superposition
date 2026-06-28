# Superposition Instructions

Superposition is a console for moving and managing Workspace ONE UEM configuration. It runs three jobs from one UI:

- **Migration**: Intune to Workspace ONE UEM (one-time import)
- **Sync**: Workspace ONE to Workspace ONE (keep two tenants aligned)
- **GitOps**: Workspace ONE to GitHub (configuration as code, with an Actions-driven apply)

---

## Table of Contents

1. [Installation](#installation)
2. [Configuration](#configuration)
3. [Credential Storage](#credential-storage)
4. [Database Storage](#database-storage)
5. [Policies](#policies)
6. [Migration (Intune to Workspace ONE)](#migration-intune-to-workspace-one)
7. [Sync (Workspace ONE to Workspace ONE)](#sync-workspace-one-to-workspace-one)
8. [GitHub Sync (Workspace ONE to GitHub)](#github-sync-workspace-one-to-github)
9. [Schedules](#schedules)
10. [Troubleshooting](#troubleshooting)

---

## Installation

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- An Intune tenant (source) and/or a Workspace ONE UEM tenant (destination)

### Pull and run the container

The image is published publicly on the GitHub Container Registry. No login is required.

**1. Pull the image**

```bash
docker pull ghcr.io/tbwfdu/superposition:latest
```

**2. Run the container**

```bash
docker run -d \
  --name superposition \
  -p 3000:3000 \
  -v superposition-output:/app/output \
  --restart unless-stopped \
  ghcr.io/tbwfdu/superposition:latest
```

Open [http://localhost:3000](http://localhost:3000). Allow 10-15 seconds for the container to start.

The named volume `superposition-output` persists everything that matters across restarts and image upgrades: your databases, credentials vault, and any binary application artifacts from sync runs. Deleting the volume is the equivalent of a factory reset; do not remove it unless you intend to start fresh.

<img src="docs/screenshots/001-dashboard.png" width="700">

---

## Configuration

All credentials are entered on the **Config** page. Secrets are encrypted at rest using AES-256-GCM (see [Credential Storage](#credential-storage) below). A live connection test runs against the external API when you save each environment, so a failed test means the credentials or permissions are wrong, not that the save itself failed.

<img src="docs/screenshots/002-config-empty.png" width="700">

### Connect an Intune tenant (source)

Superposition reads Intune via the Microsoft Graph API using an Entra ID app registration with application (not delegated) permissions. Application permissions mean no user login is required; the app authenticates directly with its client secret.

**Create the app registration**

1. In the [Azure portal](https://portal.azure.com), open **Entra ID > App registrations > New registration**.
2. Give it a name (e.g. `Superposition`), leave the redirect URI blank, and click **Register**.
3. Note the **Application (client) ID** and **Directory (tenant) ID** from the Overview page.
4. Go to **Certificates and secrets > Client secrets > New client secret**. Set an expiry that suits your operational schedule. Copy the **Value** immediately; it is not shown again.
5. Go to **API permissions > Add a permission > Microsoft Graph > Application permissions** and add:

   | Permission | Purpose |
   |---|---|
   | `DeviceManagementConfiguration.Read.All` | Read device configuration profiles and Settings Catalog policies |
   | `DeviceManagementApps.Read.All` | Read application inventory and assignments |
   | `Group.Read.All` | Resolve Entra group display names on exported assignments (recommended; falls back to raw GUIDs without it) |

6. Click **Grant admin consent**. All three permissions require admin consent; the export will fail with 403 until consent is granted.

**Add to Superposition**

In the web console, go to **Config > Add environment > Intune**. Enter the Tenant ID, Client ID, and Client Secret from the steps above. Superposition stores only the non-secret metadata (Tenant ID, Client ID) in the environment database; the Client Secret is encrypted and stored separately in the credentials vault.

<img src="docs/screenshots/003-config-intune-saved.png" width="700">

### Connect a Workspace ONE UEM tenant

Superposition reads from and writes to WS1 UEM using OAuth 2.0 client credentials. The OAuth client authenticates with its own client ID and secret; no user account is required.

**Create the OAuth client**

1. In the UEM console, go to **Groups and Settings > Configurations > OAuth Client Management**.
2. Click **Add** and create a new client. Set the description to something recognisable (e.g. `Superposition Sync`).
3. Copy the **Client ID** and **Client Secret** when they are shown. The secret is not retrievable later.
4. Grant the client sufficient role permissions to read profiles, scripts, sensors, smart groups, and applications in the source Organization Group; and to create/update the same resource types in the target Organization Group. A role with full API access to the relevant OGs is the safest starting point.

**Find your token URL**

The token endpoint is regional. Use the one that matches where your UEM tenant is hosted:

| Region | Token URL |
|---|---|
| North America | `https://na.uemauth.workspaceone.com/connect/token` |
| Europe | `https://eu.uemauth.workspaceone.com/connect/token` |
| Asia Pacific | `https://apac.uemauth.workspaceone.com/connect/token` |

**Find your Organization Group ID**

Navigate to the target Organization Group in the UEM console. The **Group ID** is the short human-readable string shown in the OG's settings, for example `contoso-prod`. This is not the numeric internal ID that appears in the URL; it is the text identifier used when you created the OG.

**Add to Superposition**

In the web console, go to **Config > Add environment > Workspace ONE**. Enter the Server URL (e.g. `https://as1831.awmdm.com`), Client ID, Client Secret, Token URL, and Group ID. The Server URL must not have a trailing slash.

<img src="docs/screenshots/004-config-workspace-one-prod-saved.png" width="700">

### Connect a GitHub App (for GitOps syncs)

A GitHub App is a machine identity that Superposition uses to authenticate to GitHub on behalf of your organisation. It is scoped to the specific repository you install it on, and uses a short-lived installation token (not a personal access token) for all API calls.

**Create the app registration**

1. Go to **GitHub > Settings > Developer settings > GitHub Apps > New GitHub App**.
2. Fill in:
   - **GitHub App name**: e.g. `Superposition Sync`
   - **Homepage URL**: any URL (required by GitHub but not used)
   - **Webhook**: untick Active (Superposition does not need to receive webhook events)
3. Set these **Repository permissions** (all other permissions can remain at "No access"):

   | Permission | Access | Why |
   |---|---|---|
   | Contents | Read and write | Create commits and branches |
   | Workflows | Read and write | Scaffold the apply workflow on first push |
   | Pull requests | Read and write | Required for PR-gated mode |
   | Metadata | Read-only | Granted automatically |

4. Click **Create GitHub App**.
5. Note the **App ID** shown at the top of the App's settings page.
6. Scroll to **Private keys > Generate a private key**. GitHub downloads a `.pem` file. Keep the entire file including the `-----BEGIN RSA PRIVATE KEY-----` header and `-----END RSA PRIVATE KEY-----` footer lines. Superposition uses this key to sign RS256 JWTs to obtain installation tokens at runtime.

**Install the App**

On the App's settings page, click **Install App**, choose your organisation, select **Only select repositories**, and pick the config repo. The app must be installed before any push can succeed.

**Add to Superposition**

Go to **Config > Add environment > GitHub App**. Enter the App ID and paste the full contents of the `.pem` file into the Private Key field.

<img src="docs/screenshots/005-config-github-app-saved.png" width="700">

---

## Credential Storage

Superposition separates environment metadata from secrets at the storage layer.

### What goes where

**Non-secret metadata** (Tenant IDs, Client IDs, Server URLs, Group IDs, App IDs) is stored in `environments.db` in the output volume. This data is not sensitive on its own; it identifies the environment but cannot authenticate to it.

**Secrets** (Client Secrets, Private Keys) are stored in a separate encrypted vault file at `output/secrets.enc.json` inside the output volume. The vault file is keyed by environment ID, so metadata and secrets are never combined into a single record.

### Encryption

Secrets are encrypted with **AES-256-GCM** before being written to disk. Each secret bag is encrypted independently with a fresh random 96-bit IV (nonce), producing three hex-encoded fields stored in the vault:

- `iv`: the 12-byte initialisation vector used for this entry
- `tag`: the 16-byte GCM authentication tag (detects tampering)
- `ct`: the ciphertext

The GCM authentication tag means any modification to the ciphertext is detected and rejected at decrypt time. The IV is unique per write, so two identical secrets produce different ciphertext.

### Master key

The 256-bit master key is sourced in this order of precedence:

1. The `SUPERPOSITION_MASTER_KEY` environment variable, if set. This is the expected path when deploying with an external secret manager.
2. The value stored in `output/.env` inside the persistent volume. On first run, the dashboard generates a random key and writes it here. Subsequent container restarts reload it from the volume so the vault remains readable without any manual key management.

The key is accepted as either 64 hex characters or base64. It is cached in memory for the lifetime of the server process and never written to logs.

### Vault file permissions

The vault file is written with mode `0600` (owner read/write only). Even though the content is encrypted, the restrictive permissions reduce the chance of an accidental world-read by a co-located process.

---

## Database Storage

Superposition uses two SQLite databases, both stored in the output volume. The Drizzle ORM manages schema migrations automatically on startup.

### `environments.db`

Stores saved environment configurations: one row per connected Intune tenant, Workspace ONE UEM tenant, or GitHub App. The `config` column holds a JSON blob with non-secret fields only (Tenant ID, Client ID, Server URL, Group ID, etc.). Secrets are never written to this database; they are looked up from the vault by environment ID at runtime.

### `migrations.db`

The main operational database. It contains:

| Table | What it stores |
|---|---|
| `migrations` | Saved migration configs: source/destination environments, selected resource categories, platform scope, and mode flags. Each row is re-runnable; re-opening a migration creates a new run, not a new migration. |
| `syncs` | Saved sync configs: source and destination environments and OGs, selected categories, attached policy IDs, GitHub destination settings (repo, branch, apply mode toggles), and schedule state. |
| `policies` | Reusable policy definitions: change detection patterns, conflict resolution strategy, and resource definition include/exclude lists. Four built-in policies are seeded on startup. |
| `runs` | One row per pipeline stage invocation. Records the start/end time, exit code, stage ID, trigger (manual or schedule), and chain ID linking stages of one end-to-end execution. |
| `run_artifacts` | One row per output file produced by a run. Text-based artifacts (JSON, Markdown, HTML, SyncML) are stored inline in the `content` column. Binary files (app installers) are kept on disk under `output/migrations/{ownerId}/runs/{runId}/_artifacts/` and referenced by path. |
| `sync_schedules` | One row per sync that has a schedule configured. Stores cadence (hourly/daily/weekly), time, timezone, and the last/next fire times. The in-process scheduler reads this table on boot and on each 30-second tick. |

### Run artifact lifecycle

When a pipeline stage completes, the runner reads every file from the stage's scratch directory and ingests them into `run_artifacts`. Text files go directly into the `content` column; binary files stay on disk and only their path is recorded. The scratch directory is then deleted. Subsequent stages (and the UI) read artifacts from the database, not from disk. This means the database is the durable source of truth; there is no separate filesystem sync to worry about.

---

## Policies

Policies control how syncs detect changes and handle conflicts. They are defined once and can be attached to any sync at compose time. Four built-in policies are seeded on startup and cannot be edited or deleted; you can clone a built-in to create an editable variant.

<img src="docs/screenshots/020-policies-page.png" width="700">

<img src="docs/screenshots/021-policies-all-saved.png" width="700">

### Change detection

Change detection policies determine what counts as a meaningful difference between a source resource and the corresponding destination resource. Two modes exist, and a sync can have one of each attached simultaneously.

**Ignore mode**

An ignore-mode policy defines fields that are excluded from the comparison. Before the engine diffs a source and destination resource pair, it strips any field path that matches a pattern in the attached ignore policy. A change to an ignored field is never treated as drift and will not trigger an update.

This is useful for fields that change automatically and carry no configuration intent, such as timestamps, internal version counters, blob checksums, or install counts. The built-in "Default ignore patterns" policy covers the most common volatile fields across profiles, scripts, sensors, and apps. Attaching it prevents every sync from proposing updates purely because UEM incremented an internal counter.

<img src="docs/screenshots/022-policies-ignored-keys-editor.png" width="700">

Each pattern in an ignore policy targets a specific resource type (e.g. `profiles`, `scripts`, `apps`) and a glob expression matching the field path to ignore (e.g. `modifiedAt`, `version`, `blobId`). Patterns are evaluated against the dot-notation path of each field in the resource body.

You can also add **update patterns** to an ignore policy. An update pattern is a path-prefix rule that further restricts when an "ignored key changed" fires. This lets you suppress changes to a broad key (e.g. `metadata`) while still catching changes to specific sub-keys within it that do matter. Without update patterns, the glob applies to the whole key unconditionally.

<img src="docs/screenshots/023-policies-ignored-keys-update-patterns.png" width="700">

**Match mode**

A match-mode policy is the inverse of ignore: instead of saying "these fields don't matter", it says "only these fields matter". When a match policy is attached to a sync, the engine is in a restrictive posture: only field paths matching one of the policy's patterns count as a real update. Changes to any other field are silently discarded.

Match policies are useful when you want a sync to be extremely selective, for example to watch only for changes to a `payloadContent` field and ignore everything else. No built-in match policy is shipped because match-only mode is an advanced opt-in; create one from the Policies page when needed.

### Exclusions

Exclusion policies define which resources are skipped entirely. A resource that matches any attached exclusion policy is not compared, not imported, and not counted in conflict statistics. It appears in run results with status `excluded` so you can see what was dropped.

Multiple exclusion policies can be attached to a single sync. The engine unions all attached exclusion lists into one combined exclude set before processing begins.

**Exclude Smart Groups**

Drops resources whose assignment targets reference one or more specific smart groups. Enter the display names of the smart groups to exclude. Any source resource assigned to one of the named groups is excluded from the sync run.

<img src="docs/screenshots/024-policies-exclude-smart-groups-editor.png" width="700">

**Exclude Certificate Profiles**

Drops certificate-type profiles from the sync scope regardless of name or assignment. This is a type-based exclusion: any resource that Superposition identifies as a certificate profile (based on its payload type) is skipped.

<img src="docs/screenshots/025-policies-exclude-certificate-profiles-editor.png" width="700">

### Conflict resolution

A conflict resolution policy determines what happens when a source resource and a destination resource share the same name. Every sync must have exactly one conflict resolution policy attached.

| Policy | Behaviour | When to use |
|---|---|---|
| **Skip** | Leave the existing destination resource completely untouched. The sync creates new resources but never modifies existing ones. | First-time syncs where you want to discover what already exists before overwriting anything. Safe re-run default. |
| **Overwrite** | Update the destination resource in place with the source body. The destination UUID is preserved where the UEM API supports in-place updates, so device profile assignments remain intact. | Ongoing syncs where the source is authoritative and you want changes propagated automatically. |
| **Source wins** | Update matching destination resources AND delete any destination resource whose name does not appear in the source. | Syncs where the source OG is the single source of truth and the destination should mirror it exactly, including deletions. Use with caution: any resource created directly in the destination will be deleted. |

The built-in Skip, Overwrite, and Source wins policies cover the common cases. Clone one to create a named variant with a custom description if you want to document the intent per sync.

---

## Migration (Intune to Workspace ONE)

A migration moves configuration from an Intune tenant into a Workspace ONE UEM Organization Group. It runs once per migration object; re-running is safe because the validation step surfaces conflicts before any write occurs.

### Step 1: Compose

Open **Migrate > Compose** and work through the steps.

<img src="docs/screenshots/040-compose-migration-page.png" width="700">

**Step 2: Select source and destination**

Drag your Intune environment into the Source slot and your Workspace ONE environment into the Destination slot. The environments listed are the ones you saved on the Config page.

<img src="docs/screenshots/041-compose-migration-source.png" width="700">

<img src="docs/screenshots/042-compose-migration-destination.png" width="700">

**Step 3: Pick resources**

Choose the resource types to include. Available categories include device configuration profiles, Settings Catalog policies, Group Policy (ADMX) templates, compliance policies, device management scripts, proactive remediations, application inventory, certificates, network profiles, update management artifacts, enrollment and provisioning inventory, and RBAC inventory. You can scope the migration to Windows only, macOS only, or both.

Not every category produces importable WS1 profiles. Non-importable categories (RBAC, enrollment, update management) export their data and generate a migration report with manual guidance instead.

<img src="docs/screenshots/043-compose-migration-resources.png" width="700">

**Step 4: Review the canvas and save**

The canvas shows the full pipeline chain from source to destination, with each stage represented as a node. Hit **Save and continue** to store the migration and proceed to the Export stage.

<img src="docs/screenshots/044-compose-migration-canvas.png" width="700">

### Step 2: Export

The Export stage authenticates to the Microsoft Graph API and pulls the full breadth of your Intune tenant across all selected categories: device configurations, Settings Catalog policies, Group Policy ADMX templates, compliance policies, device management scripts, proactive remediations, application inventory, assignments, certificates, network profiles, update management, enrollment, provisioning, and RBAC inventory.

The export fetches full detail for each resource. The Graph list endpoints omit some fields; Superposition follows up with per-resource detail calls to fill the gaps. Assignments are normalised and Entra group GUIDs are resolved to display names (if `Group.Read.All` is granted).

<img src="docs/screenshots/045-migrate-export-page.png" width="700">

Hover over a stream node to expand the resource list within that category before starting:

<img src="docs/screenshots/046-migrate-export-expanded.png" width="700">

Hit **Start Export**. Stream nodes flip from grey to colour as each category completes. The right panel shows live per-category progress including counts and any errors.

<img src="docs/screenshots/047-migrate-export-in-progress.png" width="700">

### Step 3: Translate

The Translate stage converts exported Intune profiles into Workspace ONE UEM JSON using Claude AI via Vertex AI. Each setting within a profile is translated individually: a failure in one setting does not block the rest of the profile. Profiles with any failed setting are marked "partial" and their migration notes identify which keys could not be mapped and why.

All Windows payload types (Custom OMA-URI, BitLocker, Defender, Windows Update, Firewall, and ADMX/Group Policy) are consolidated into a single `CustomSettingsPayload` in the translated output. This is the format the Workspace ONE WinRT V2 REST API accepts; it converts the OMA-URI / CSP paths to SyncML at upload time. The WS1 V2 API does not expose dedicated Encryption, Security, or Update payload sections, so the translator does not emit them. The migration report's "native candidates" section identifies profiles that have an equivalent WS1 V1 native payload (e.g. the console Encryption payload) and need a manual rebuild.

<img src="docs/screenshots/048-migrate-translate-page.png" width="700">

Hit **Start Translate** to run the conversion. Translation time depends on the number of profiles and the number of individual settings within each profile.

### Step 4: Validation

Validation posts translated profiles to the live Workspace ONE tenant in preview mode. Nothing is written to UEM. All conflicts, unsupported keys, name collisions, and payload errors surface here before a single byte is committed.

When validation completes, the results modal shows a per-profile breakdown with status, conflict classification, and any notes:

<img src="docs/screenshots/049-migrate-validation-results.png" width="700">

A 100% pass means all profiles are structurally valid and ready for import. Partial passes identify specific profiles that have settings the validator flagged; review the migration notes for those profiles before proceeding.

### Step 5: Import

The Import stage uploads validated profiles to the target Workspace ONE Organization Group via the WinRT V2 REST API. Before writing anything, the resolution dialog lists every resource and lets you confirm or deselect individual items:

<img src="docs/screenshots/050-migrate-import-page.png" width="700">

<img src="docs/screenshots/051-migrate-import-resolve.png" width="700">

Click **Confirm and run import**. When the run completes, the results modal shows the outcome per resource type including counts of created, skipped, and failed items:

<img src="docs/screenshots/052-migrate-import-results.png" width="700">

### Step 6: Manual steps

Some profile types cannot be migrated automatically. The Manual Steps stage generates a report of every profile that requires a human to recreate it in the Workspace ONE console. The report includes the settings to apply and a reference to the source Intune policy.

<img src="docs/screenshots/053-migrate-manual-steps.png" width="700">

Hit **Generate report** after each import run to refresh the list with the latest outcomes. The report accounts for what was successfully imported so only the genuinely manual items remain.

---

## Sync (Workspace ONE to Workspace ONE)

A sync keeps two Workspace ONE tenants aligned on an ongoing basis. Common use cases include Production-to-UAT promotion, Pre-prod-to-Production rollout, and tenant migration during a tenancy change.

### Step 1: Compose

Open **Sync > Compose** and work through the steps.

<img src="docs/screenshots/070-compose-sync-page.png" width="700">

**Step 1: Choose the flow type**

Select **Sync** (Workspace ONE to Workspace ONE or GitHub).

<img src="docs/screenshots/071-compose-sync-step1.png" width="700">

**Step 2: Choose the source kind**

Select **Workspace ONE** as the source.

<img src="docs/screenshots/072-compose-sync-step2-source-kind.png" width="700">

**Step 3: Select environments and organization groups**

Drag your source Workspace ONE environment into the Source slot and your destination environment into the Destination slot. Then select the source and destination Organization Groups from the dropdowns. The OG dropdown reflects the full hierarchy of the connected tenant, so you can pick an exact level.

<img src="docs/screenshots/073-compose-sync-step3-environments.png" width="700">

<img src="docs/screenshots/074-compose-sync-step3-source-og.png" width="700">

**Step 4: Attach policies**

Policies govern change detection and conflict resolution for this sync. Select the policies you configured on the Policies page:

- **Change Detection (Ignore)**: attach an ignore-mode policy to filter out volatile fields that should not count as drift.
- **Change Detection (Match)**: optionally attach a match-mode policy if you only want a subset of fields to trigger updates.
- **Conflict Resolution**: pick Skip, Overwrite, or Source wins. This is required.
- **Exclusions**: attach one or more exclusion policies to filter out specific resource types or assignment-scoped resources.

You can also enable **Mirror organization group structure** to replicate the source OG's child hierarchy under the destination OG. When enabled, the export walks the source OG tree and the importer creates matching child OGs in the destination before placing each resource into its mirrored OG.

<img src="docs/screenshots/075-compose-sync-step4-policies.png" width="700">

Save and continue to the Export stage.

### Step 2: Export

The Export stage reads profiles, scripts, sensors, applications, and smart groups from the source Workspace ONE OG. Only resources within the selected OG scope are exported; child OG resources are included only if Mirror OG structure is enabled.

<img src="docs/screenshots/076-sync-export-page.png" width="700">

Hit **Start Export** and monitor the canvas as each resource type streams in.

### Step 3: Validation

Validation compares the source export against the destination OG without writing anything. It applies the attached change detection and conflict policies to identify what would be created, updated, or skipped. The results give you a preview of the full import plan before you commit.

<img src="docs/screenshots/077-sync-validation-page.png" width="700">

When complete, the validation results modal gives a per-category breakdown of the planned actions:

<img src="docs/screenshots/078-sync-validation-results.png" width="700">

### Step 4: Import

The import review dialog shows the full execution plan: how many resources will be created, how many updated, how many skipped, and whether any deletes are pending (Source wins mode only). You can deselect individual resources before confirming.

<img src="docs/screenshots/079-sync-import-review.png" width="700">

Click **Confirm and run import**. Results are shown per resource type on completion:

<img src="docs/screenshots/080-sync-import-results.png" width="700">

---

## GitHub Sync (Workspace ONE to GitHub)

The GitHub sync commits live Workspace ONE configuration to a GitHub repository as versioned JSON. A bundled GitHub Actions workflow ships alongside the export and applies the configuration to a target UEM when triggered. This is the recommended path for production tenants: every configuration change has a git history, and a pull request provides a human review gate before anything reaches the live environment.

```
Superposition sync --> GitHub repo --> Actions workflow --> target UEM
```

### Prerequisites

Complete the [Connect a GitHub App](#connect-a-github-app-for-gitops-syncs) setup in Config, then add the following Actions secrets to the config repository under **Settings > Secrets and variables > Actions**:

| Secret | Value |
|---|---|
| `WS1_SERVER` | UEM API host, e.g. `https://as1831.awmdm.com` |
| `WS1_CLIENT_ID` | OAuth client ID |
| `WS1_CLIENT_SECRET` | OAuth client secret |
| `WS1_TOKEN_URL` | Region OAuth token endpoint |
| `WS1_GROUP_ID` | Target Organization Group string ID |

The config repository must have at least one existing commit before the first sync. Create a README when initialising the repo.

### Step 1: Compose

Follow the same Compose flow as a WS1-to-WS1 sync (Steps 1-2 above). At Step 3, drag your GitHub App environment into the Destination slot and configure the target repository (`owner/name`) and branch:

<img src="docs/screenshots/090-compose-sync-github-step3.png" width="700">

**Apply modes**

| Mode | Behaviour | When to use |
|---|---|---|
| Direct push | Commits straight to the branch; the Actions apply workflow fires immediately on every sync | Dev and scratch tenants where you want instant propagation |
| PR-gated | Pushes to a fresh `superposition-sync/<timestamp>` branch and opens a pull request; the apply workflow fires only after a human merges the PR | Production tenants where changes need review before they reach UEM |
| Export only | Commits with `[skip ci]`; no apply workflow is scaffolded or triggered | Config backups, audit trails, or snapshot archives with no automated apply |

Export-only and PR-gated modes compose independently: Export only controls whether the apply pipeline runs at all; PR-gated controls how the snapshot lands in the repo (direct commit vs PR). An export-only PR carries a note in the PR description that merging it triggers no apply.

**Additional toggles**

- **Mirror organization group structure**: replicates the source OG hierarchy into the repo under `slingshot/export/og_tree.json` and bakes `mirrorOgStructure: true` into the manifest, so the apply workflow forwards `--mirror-ogs` to the importer on the target UEM.
- **Git LFS**: routes binary application files (`.msi`, `.exe`, `.pkg`, `.dmg`, `.msix`) through GitHub LFS. Without this, non-JSON files are skipped. Enable it only if you need application binaries in the repo; LFS storage and bandwidth are metered on most GitHub plans.

### Step 2: Export

Run the Export stage the same way as a WS1-to-WS1 sync. The export feeds directly into the push stage.

### Step 3: Push to GitHub

Navigate to the **Push to GitHub** stage. The destination summary shows the target repository, apply branch, and the active toggles (apply mode, LFS, OG mirroring).

<img src="docs/screenshots/091-sync-github-push-page.png" width="700">

Click **Push and open PR** (or **Push to GitHub** for direct push). A confirmation dialog describes exactly what will happen before any write occurs:

<img src="docs/screenshots/092-sync-github-push-confirm.png" width="700">

The push log shows each step: authenticating with the GitHub App (minting a short-lived installation token from the RS256-signed App JWT), resolving the target branch HEAD, uploading any LFS objects, committing the export files and bundled pipeline under `slingshot/`, and opening the pull request.

<img src="docs/screenshots/093-sync-github-push-progress.png" width="700">

On the first push, Superposition also scaffolds `.github/workflows/superposition-apply.yml` and `slingshot/manifest.json`. The manifest records the categories, conflict policy, OG mirroring flag, and the apply command so the workflow needs no manual configuration. Unchanged files reuse their existing blob SHAs, so subsequent pushes commit only the files that actually changed.

### Step 4: Review and merge the pull request

Open the pull request on GitHub. Superposition populates the PR description with the conflict policy, categories in scope, file count, and apply branch. The Actions workflow check runs immediately to validate the commit structure.

<img src="docs/screenshots/094-sync-github-pr-details.png" width="700">

### Step 5: The Actions workflow applies to the target UEM

Merging the PR triggers the bundled `superposition-apply.yml` workflow. It installs the pipeline dependencies from `slingshot/pipeline/requirements.txt`, reads `slingshot/manifest.json`, stages the export for the importer, and applies the configuration to the target UEM using the `WS1_*` Actions secrets.

<img src="docs/screenshots/095-sync-github-runner.png" width="700">

The **Apply to target UEM** step shows the full command including categories, conflict policy, and OG mirroring flag:

<img src="docs/screenshots/096-sync-github-runner-apply.png" width="700">

The importer logs every OG it creates, every resource it processes, and a final tally of created, updated, skipped, and failed items:

<img src="docs/screenshots/097-sync-github-runner-uem-progress.png" width="700">

---

## Schedules

A saved sync can run end-to-end on a schedule without any manual intervention. The schedule fires the export, validation, and import stages as one unattended chain. For GitHub-destination syncs in PR-gated mode, the schedule still opens a PR; a human must still merge it before the apply workflow fires.

To attach a schedule, open a saved sync and go to the **Schedule** step at the bottom of the compose flow. Choose a cadence:

| Cadence | What it means |
|---|---|
| Hourly | Fires once per hour at the configured minute |
| Daily | Fires once per day at the configured hour and minute |
| Weekly | Fires once per week on the configured weekday, hour, and minute |

Set the time and time zone (IANA timezone names, e.g. `America/Los_Angeles`). The scheduler evaluates due syncs on a 30-second tick, so actual fire times may be up to 30 seconds after the configured time.

Schedules are stored in the database and rehydrate on server boot. Scheduled runs that were missed while the server was down are not caught up automatically. Each scheduled run is tagged `trigger: schedule` in the run history, distinct from runs started manually. To run a sync immediately without waiting for the next scheduled time, use the **Run now** button on the sync's detail page.

Schedules only fire while the Superposition server process is running. If the container is stopped, no runs execute. Plan your container restart windows accordingly if you rely on scheduled syncs for production automation.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Container exits immediately | Run `docker logs superposition` to see the startup error |
| Graph API returns 403 on export | Verify admin consent was granted for all three app registration permissions in the Azure portal |
| WS1 import fails with "No organization group found" | Confirm the Group ID in Config matches the OG's Group ID string in the UEM console. It is the human-readable short identifier, not the numeric internal ID in the URL |
| GitHub environment save fails with "rejected App ID / private key pair" | Re-paste the full `.pem` file including the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines |
| Sync fails with "the GitHub App is not installed on owner/name" | Install the GitHub App on the config repository: App settings page > Install App > select the repo |
| First push fails with "repo has no commits" | Add an initial commit to the config repository. Tick "Add a README" when creating it in GitHub |
| Push fails with 403 on `.github/workflows/` | Add Workflows: Read and write to the GitHub App permissions, then accept the pending permission update on the App's installation page |
| Apply workflow fails authenticating to UEM | Check that all five `WS1_*` Actions secrets are set correctly under the config repository's Settings > Secrets and variables > Actions |
| Validation shows 100% but import fails | Check that the UEM OAuth client has write permissions on the target Organization Group, not just read permissions |
| Translated profiles are "partial" | Open the migration report for that profile. Migration notes identify which specific settings could not be mapped and whether a manual V1 payload rebuild in the WS1 console is needed |
| Scheduled runs are not firing | Confirm the container is running. Schedules only fire while the server process is up. Check `docker logs superposition` for scheduler startup messages |
| Change detection is flagging everything as updated on every run | Attach the built-in "Default ignore patterns" change detection policy to the sync. Volatile fields like timestamps and version counters are excluded from the diff when this policy is attached |
