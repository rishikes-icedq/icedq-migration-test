# iceDQ GitHub Actions Guide

This guide will help you get started with iceDQ's GitHub Actions to automate your tasks.

## Available GitHub Actions

| Action | What it does |
|---|---|
| [`icedq-tools/export-action`](https://github.com/marketplace/actions/icedq-export) | Initiates an export, polls until complete, downloads the bundle ZIP, optionally uploads it as a workflow artifact. |
| [`icedq-tools/generate-mapping-action`](https://github.com/marketplace/actions/icedq-generate-mapping) | Analyses the export bundle, queries the target workspace to auto-match connections, parameters, and custom fields by name and type, and produces a ready-to-use mapping JSON for the import action. |
| [`icedq-tools/import-action`](https://github.com/marketplace/actions/icedq-import) | Submits a bundle to a target workspace, polls until complete, parses the import log for skipped rules, optionally fails the workflow on any skip (strict: true). |

All three Actions handle OAuth token acquisition/refresh, async job polling with backoff, and step-summary output for you — you only need to supply the inputs shown below.

---

## Prerequisites

### 1. Create a Service Account

For promoting your resources from lower environment to higher, you will need to create a **service account** in both environments. Follow the steps in [How to Create a Service Account](Service%20Account.md#how-to-create-a-service-account).

> **Note:** Service accounts are available from **iceDQ version 7.8.0** and above.

### 2. Assign Roles to the Service Account

You are required to provide appropriate role to each service account to Export or Import resources.

| Action | Minimum required Role |
|---|---|
| Export | Reader |
| Generate Mapping | Contributor |
| Import | Contributor |

Follow the steps in [How to Assign Role to the Service Account](Service%20Account.md#how-to-assign-role-to-the-service-account).

### 3. Collect your iceDQ identifiers

You'll need to configure the following as GitHub Environment variables for each environment. If you don't have access to find these values, ask your iceDQ administrator.

| Variable | What it is | Where to get it | Required for |
|---|---|---|---|
| `ICEDQ_URL` | Base URL of your iceDQ instance | Your browser's address bar when logged in to iceDQ | All actions |
| `ICEDQ_KEYCLOAK_URL` | Keycloak authentication realm URL | Your iceDQ administrator — format: `https://<host>/auth/realms/<realm>` | All actions |
| `ICEDQ_ORG_ID` | Your iceDQ organization ID | Open any rule → view rule metadata → copy `orgId` | All actions |
| `ICEDQ_ACCOUNT_ID` | Your iceDQ account ID | Open any rule → view rule metadata → copy `accountId` | All actions |
| `ICEDQ_WORKSPACE_ID` | Your iceDQ workspace ID | Open any rule → view rule metadata → copy `workspaceId` | All actions |
| `ICEDQ_CLIENT_ID` | Service account client identifier | iceDQ UI → Administration → Identity and Access → Service Accounts | All actions |
| `ICEDQ_CLIENT_SECRET` | Service account client secret | Shown once when the service account is created or credentials rotated — download or copy immediately | All actions |

#### a. iceDQ Base URL

Open iceDQ in your browser. The base URL is everything before the first path segment.

There is no single correct value — every organization has its own iceDQ instance, so use your instance's address:

- iceDQ Cloud customers: typically https://app.icedq.net (shown as the example throughout these guides).

- On-premise / private-cloud installs: whatever URL your team uses (e.g., https://icedq.mycompany.com).

Whenever you see https://app.icedq.net in a config example, treat it as a placeholder and replace it with your own instance URL. Don't add a trailing slash.

#### b. iceDQ Keycloak URL

The Keycloak URL includes the realm name and follows this format:

```
https://<host>/auth/realms/<realm>
```

The realm is usually either `icedq` or `iam.icedq`. Which one applies depends on how your iceDQ instance was set up, **so confirm the exact value with your iceDQ administrator**. 

Example: `https://app.icedq.net/auth/realms/iam.icedq`

#### c. iceDQ Org ID, Account ID and Workspace ID

All three IDs are attached to every rule in your iceDQ instance. The quickest way to find them:

1. Open the **Data Testing** module.
2. Open any existing rule.
3. View the rule's metadata (the JSON definition).
4. Copy the values of `orgId`, `accountId`, and `workspaceId`.

Example metadata:

```json
{
    "orgId": "org-icedq",
    "accountId": "acct-c97e76f5-d51a-560e-a322-c46f76a2455d",
    "workspaceId": "wksc-fdd3fa7d-07ab-5b7a-b039-4f8330bab135",
    "rule": {
        "id": "rule-c2c3652a-38e6-5f26-abec-8571f09c6ad3",
        "folderId": "fldr-ec5b7669-9070-5e61-8dd0-50ab97486bdc",
        "sourceConnectionId": "conn-b21e00c4-621a-5b60-9cea-aa45c218e248"
    }
}
```

> If your workspace has no rules yet, ask your iceDQ administrator — every organization in iceDQ has exactly one Org ID, and the admin can read all three values directly from the platform.

#### d. iceDQ Client ID and Secret

These are the credentials of the service account you created in [Prerequisite 1](#1-create-a-service-account). The Client Secret is shown only once at creation time — if you no longer have it, rotate the credentials following the steps in [How To: Rotate Service Account Credentials](Service%20Account.md#how-to-rotate-service-account-credentials).

---

## Quick Start

### Promote a Workflow on every push

```yaml
# .github/workflows/promote-workflow.yml
name: Promote Workflow from DEV to UAT

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}
  cancel-in-progress: false

jobs:
  export-workflow:
    runs-on: ubuntu-latest
    environment: DEV
    timeout-minutes: 45
    steps:
      - name: Export workflow from DEV environment
        uses: icedq-tools/export-action@v1
        with:
          icedq-url:       ${{ vars.ICEDQ_URL }}
          keycloak-url:    ${{ vars.ICEDQ_KEYCLOAK_URL }}
          org-id:          ${{ vars.ICEDQ_ORG_ID }}
          client-id:       ${{ secrets.ICEDQ_CLIENT_ID }}
          client-secret:   ${{ secrets.ICEDQ_CLIENT_SECRET }}
          account-id:      ${{ vars.ICEDQ_ACCOUNT_ID }}
          workspace-id:    ${{ vars.ICEDQ_WORKSPACE_ID }}
          resource:        workflow
          id:              ${{ vars.FINANCE_WORKFLOW_ID }}
          include-child:   true
          output-file:     ./exports/finance.zip
          artifact-name:   icedq-finance-workflow-bundle

  generate-workflow-mapping:
    runs-on: ubuntu-latest
    needs: export-workflow
    environment: UAT
    timeout-minutes: 30
    steps:
      - name: Download bundle from export job
        uses: actions/download-artifact@v4
        with:
          name: icedq-finance-workflow-bundle
          path: ./exports

      - name: Generate mapping for UAT environment
        uses: icedq-tools/generate-mapping-action@v1
        with:
          icedq-url:      ${{ vars.ICEDQ_URL }}
          keycloak-url:   ${{ vars.ICEDQ_KEYCLOAK_URL }}
          org-id:         ${{ vars.ICEDQ_ORG_ID }}
          client-id:      ${{ secrets.ICEDQ_CLIENT_ID }}
          client-secret:  ${{ secrets.ICEDQ_CLIENT_SECRET }}
          account-id:     ${{ vars.ICEDQ_ACCOUNT_ID }}
          workspace-id:   ${{ vars.ICEDQ_WORKSPACE_ID }}
          bundle:         ./exports/finance.zip
          output-file:    ./mapping/finance-workflow-mapping.json
          artifact-name:  icedq-finance-workflow-mapping

  import-workflow:
    runs-on: ubuntu-latest
    needs: [export-workflow, generate-workflow-mapping]
    environment: UAT
    timeout-minutes: 30
    steps:
      - name: Download bundle from export job
        uses: actions/download-artifact@v4
        with:
          name: icedq-finance-workflow-bundle
          path: ./exports

      - name: Download generated mapping
        uses: actions/download-artifact@v4
        with:
          name: icedq-finance-workflow-mapping
          path: ./mapping

      - name: Import workflow into UAT environment
        uses: icedq-tools/import-action@v1
        with:
          icedq-url:             ${{ vars.ICEDQ_URL }}
          keycloak-url:          ${{ vars.ICEDQ_KEYCLOAK_URL }}
          org-id:                ${{ vars.ICEDQ_ORG_ID }}
          client-id:             ${{ secrets.ICEDQ_CLIENT_ID }}
          client-secret:         ${{ secrets.ICEDQ_CLIENT_SECRET }}
          account-id:            ${{ vars.ICEDQ_ACCOUNT_ID }}
          workspace-id:          ${{ vars.ICEDQ_WORKSPACE_ID }}
          bundle:                ./exports/finance.zip
          kind:                  workflows
          mapping-file:          ./mapping/finance-workflow-mapping.json
          strict:                false
          terminate-on-conflict: true
```

**Notes on this pipeline:**
- The `generate-workflow-mapping` job runs against the **UAT environment** because it needs to query TARGET workspace (UAT in this example) to auto-match connections by name and type.
- You can further extend this pipeline to promote the same artifact bundle through all your environments — no re-export per environment, ensuring identical bytes are imported everywhere.
- `strict: 'true'` fails the job if any rule is skipped (e.g., a missing target connection). The next environment is gated on `needs:` so failures stop the chain.
- To go beyond DEV → UAT, add further `generate-*-mapping` + `import-*` job pairs chained with `needs:`, each targeting the next environment (e.g. QA, then Prod). Configure [required reviewers](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) on the later GitHub Environments to gate promotion behind manual approval.

### Create Mapping Files

The `import-action` requires a `mapping-file` that tells it how to re-link source connections, parameters, and custom fields to their counterparts in the target workspace. There are two ways to produce this file.

#### Option 1 (recommended) — Auto-generate with `generate-mapping-action`

Use `icedq-tools/generate-mapping-action` as a middle job between export and import. It queries the target workspace, matches resources by name and connector type, and writes a ready-to-use mapping JSON automatically. See the [Quick start](#quick-start) example for a complete pipeline.

> If a connection cannot be matched automatically (e.g. the name differs between environments), the action throws an error — there is no partial output. In that case, use Option 2 (manual mapping) instead.

#### Option 2 — Manual mapping file

Manually author a JSON file and commit it to the repo (e.g. `mappings/uat.json`). Pass its path via the `mapping-file` input of the import action. Useful when auto-matching can't resolve all entries or you need explicit control over every override. In this case, you do not need to add `generate-mapping-action` as a middle job between export and import.

For detailed usage of `generate-mapping-action`, refer to the [official action guide](https://github.com/marketplace/actions/icedq-generate-mapping).

### Tip: keep mappings under version control

Store mapping files in your repo (e.g., `mappings/qa.json`, `mappings/uat.json`, `mappings/prod.json`) so changes to UUID mappings are auditable and reviewable in PRs.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Authentication failed: HTTP 401 from Keycloak` | Wrong client ID/secret, or set on the wrong GitHub Environment | Confirm the service account's credentials are set on the correct GitHub Environment |
| `Authentication failed: HTTP 401 from Keycloak` | Realm path in `ICEDQ_KEYCLOAK_URL` is wrong | Expected shape: `https://<host>/auth/realms/<realm>` — see [iceDQ Keycloak URL format](#b-icedq-keycloak-url) |
| `ConstraintViolation: Mapping connection[0].new id ... is not present in the target workspace` | The connection (or custom field) referenced in the mapping file doesn't exist in the target workspace | Connections and custom fields must be **pre-created in the target environment** — they are not auto-created by import |
| `HTTP 409: Active job blocking` | An export or import is already running in the target workspace | Wait for it to finish, or pass `terminate-on-conflict: true` on `import-action` to cancel it and retry |
| Import shows `Completed` but rules are missing | Imports are **not atomic** — if rule #37 of 50 fails validation, rules 1–36 commit and 38–50 continue | Check the import log for skipped rules; pass `strict: true` to fail the job on any skip |

## FAQ

**1. Can I run the CLI directly without the Actions?**

Yes. The Actions are convenience wrappers around [`@icedq/cli`](https://www.npmjs.com/package/@icedq/cli) — the CLI is the source of truth. `npm install -g @icedq/cli` and use `icedq export ...` / `icedq generate-mapping ...` / `icedq import ...` from a `run:` step, or anywhere else (Jenkins, Azure DevOps, ad-hoc terminal).

**2. Can I export and import in a single job?**

Yes — you don't need to upload an artifact between jobs if the same job does both. Splitting them across jobs is what gives you per-environment reviewer approval and an artifact for forensics.

**3. Does the Action support GitHub Enterprise Server?**

Yes — all three Actions are pure composite Actions with no GHES-specific code paths. As long as your runners can reach the iceDQ API and Keycloak, GHES works the same as github.com.

## References

| Action | Documentation |
|---|---|
| export-action | [View Documentation](https://github.com/marketplace/actions/icedq-export) |
| generate-mapping-action | [View Documentation](https://github.com/marketplace/actions/icedq-generate-mapping) |
| import-action | [View Documentation](https://github.com/marketplace/actions/icedq-import) |
