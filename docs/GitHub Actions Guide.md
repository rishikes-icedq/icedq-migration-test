# iceDQ GitHub Actions Guide

This guide will help you get started with iceDQ's GitHub Actions to automate your tasks.

## Available GitHub Actions

| Action | What it does |
|---|---|
| [`icedq-tools/export-action`](https://github.com/marketplace/actions/icedq-export) | Initiates an export, polls until complete, downloads the bundle ZIP, optionally uploads it as a workflow artifact. |
| [`icedq-tools/generate-mapping-action`](https://github.com/marketplace/actions/icedq-generate-mapping) | Analyses the export bundle, queries the target workspace to auto-match connections, parameters, and custom fields by name and type, and produces a ready-to-use mapping JSON for the import action. |
| [`icedq-tools/import-action`](https://github.com/marketplace/actions/icedq-import) | Submits a bundle to a target workspace, polls until complete, parses the import log for skipped rules, optionally fails the workflow on any skip (strict: true). |

---

## Prerequisites

### 1. Create a Service Account

For promoting your resources from lower environment to higher, you will need to create **service account** in both environments. Follow the steps in [How to Create a Service Account](Service Account.md#how-to-create-a-service-account).

> **Note:** Service accounts are available from **iceDQ version 7.8.0** and above.

### 2. Assign Roles to the Service Account

You are required to provide appropriate role to each service account to Export or Import resources.

| Action | Minimum required Role |
|---|---|
| Export | Reader |
| Generate Mapping | Contributor |
| Import | Contributor |

Follow the steps in [How to Assign Role to the Service Account](Service Account.md#how-to-assign-role-to-the-service-account).

### 3. Collect your iceDQ identifiers

You'll need these for each environment of your iceDQ application:

- **Org ID**
- **Account ID**
- **Workspace ID**
- **iceDQ instance URL**
- **Keycloak URL** — base URL up to the realm name, e.g. `https://auth.example.com/realms/icedq`

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

### Create Mapping Files

The `import-action` requires a `mapping-file` that tells it how to re-link source connections, parameters, and custom fields to their counterparts in the target workspace. There are two ways to produce this file.

#### Option 1 (recommended) — Auto-generate with `generate-mapping-action`

Use `icedq-tools/generate-mapping-action` as a middle job between export and import. It queries the target workspace, matches resources by name and connector type, and writes a ready-to-use mapping JSON automatically. See the [Quick start](#quick-start) example for a complete pipeline.

> If a connection cannot be matched automatically (e.g. different name in target), you can still fall back to a manual mapping for that specific entry.

#### Option 2 — Manual mapping file

Manually author a JSON file and commit it to the repo (e.g. `mappings/uat.json`). Pass its path via the `mapping-file` input of the import action. Useful when auto-matching can't resolve all entries or you need explicit control over every override. In this case, you do not need to add `generate-mapping-action` as a middle job between export and import.

### Mapping file syntax

```json
{
  "useFqn": false,
  "mapping": {
    "connections": [
      {
        "existingId": "conn-source-uuid", // id from source environment
        "newId":      "conn-target-uuid", // id from target environment
        "action":     "override"
      }
    ],
    "parameters": [
      {
        "existingId": "param-source-uuid",
        "action":     "append"
      }
    ],
    "customFields": [
      {
        "existingId": "source-field-name",
        "newId":      "target-field-name",
        "action":     "override"
      }
    ]
  }
}
```

### Mapping Field Reference

| Object | Supported actions | What each does |
|---|---|---|
| Connections | `override` only | Re-link rules in the target to use `newId` instead of the source's `existingId`. Target connection must already exist. |
| Custom fields | `override` only | Same as connections. Target field must already exist. |
| Parameters | `append`, `override`, `upsert` | `append`: add new parameter keys to target without overwriting existing values. `override`: target parameter must already exist. `upsert`: overwrites all matching keys and appends others. **In production**, always specify `append`, `override` or `upsert` explicitly. |
| useFqn | `true`, `false` | `false`: asset with same name doesn't already exist in the target environment. `true`: an asset with same name does exist in target. |

### Tip: keep mappings under version control

Store mapping files in your repo (e.g., `mappings/qa.json`, `mappings/uat.json`, `mappings/prod.json`) so changes to UUID mappings are auditable and reviewable in PRs.

## References

| Action | Documentation |
|---|---|
| export-action | [View Documentation](https://github.com/marketplace/actions/icedq-export) |
| generate-mapping-action | [View Documentation](https://github.com/marketplace/actions/icedq-generate-mapping) |
| import-action | [View Documentation](https://github.com/marketplace/actions/icedq-import) |
