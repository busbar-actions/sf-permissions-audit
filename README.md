# busbar-actions/sf-permissions-audit

Cedar-based security analysis of Salesforce metadata. Surfaces over-privileged permission sets, permissive sharing models, FLS mismatches, flows running without sharing, permission escalation, and missing security controls — with citations to the affected components and suggested remediation.

Built on `sf-metadata-security`: Cedar schema generation for 1,442+ metadata types, security fact extraction from PermissionSet / Profile / SharingRules / Flow XML, entity-graph building, and policy evaluation.

## What it does

Wraps `busbar-sf security audit` over a metadata tree (or MDAPI zip). It builds a Cedar entity graph from the metadata facts, evaluates policies (built-in baseline or your own Cedar bundle), and emits each `SecurityIssue` with its severity, category, affected components, and remediation. Operates on **metadata files** — no live org connection required, so it chains naturally after `metadata-pull` or runs on a committed `force-app` tree.

## Finding categories

`SharingBypass`, `OverprivilegedPermissionSet`, `SharingRuleConflict`, `FieldAccessMismatch`, `MissingSecurityControl`, `PermissiveSharingModel`, `MissingAuditTrail`, `FlowRunsWithoutSharing`, `PermissionEscalation`, `InsecureExternalAccess`, `CredentialExposure`, and the Apex-derived `InsecureDml` / `InjectionVulnerability` / `ExposedEndpoint` / `ApexBypassesMetadataPermissions` (see status note below).

Severity scale: `critical` / `high` / `medium` / `low` / `info`.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `metadata-path` | yes | — | Directory or MDAPI zip of metadata to audit. |
| `package-xml` | no | `` | Scope the audit to a `package.xml`. |
| `policies` | no | `` | Cedar policy bundle (dir of `.cedar` files) for custom org rules. Empty = built-in baseline. |
| `severity-threshold` | no | `medium` | Drop findings below this severity. |
| `fail-on-severity` | no | `high` | Fail the workflow if any finding meets or exceeds this. `never` disables. |
| `report-path` | no | `security-findings.json` | JSON report destination. |
| `sarif` | no | `true` | Also emit SARIF. |
| `sarif-path` | no | `security-findings.sarif` | SARIF destination. |
| `upload-sarif` | no | `true` | Upload SARIF to Code Scanning (needs `security-events: write`). |
| `comment-pr` | no | `true` | Post summary on pull_request events. |
| `upload-artifact` | no | `true` | Upload report(s) as a workflow artifact. |
| `artifact-name` | no | `security-findings` | Artifact name. |
| `version` | no | `latest` | `busbar-sf` release tag. |
| `binary-repo` | no | `busbar-actions/actions-dist` | Where to fetch the binary. |

## Outputs

| Output | Description |
|---|---|
| `findings-count` | Total findings at or above `severity-threshold`. |
| `critical-count` / `high-count` / `medium-count` | Counts by severity. |
| `threshold-breached` | `"true"` if `fail-on-severity` was met. |
| `report-path` | Path to the JSON report. |
| `sarif-path` | Path to the SARIF (empty if `sarif=false`). |

## Example: audit on PR touching security-relevant metadata

```yaml
name: Permissions Audit

on:
  pull_request:
    paths:
      - 'force-app/**/permissionsets/**'
      - 'force-app/**/profiles/**'
      - 'force-app/**/sharingRules/**'
      - 'force-app/**/flows/**'

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: busbar-actions/sf-permissions-audit@v1
        with:
          metadata-path: force-app/main/default
          fail-on-severity: high
```

## Example: audit freshly pulled metadata with custom Cedar policies

```yaml
name: Pull + Audit

on: workflow_dispatch

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  flow:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: busbar-actions/sf-metadata-pull@v1
        with:
          sf-access-token: ${{ secrets.SF_ACCESS_TOKEN }}
          sf-instance-url: ${{ secrets.SF_INSTANCE_URL }}
          target: force-app/pulled
          commit: false

      - uses: busbar-actions/sf-permissions-audit@v1
        with:
          metadata-path: force-app/pulled
          policies: .busbar/cedar-policies
          fail-on-severity: critical
```

## Dependencies (current status)

This action is fully scaffolded but **not yet runnable end-to-end**. Pieces that need to land:

1. **`busbar-sf security audit` subcommand** — wraps `sf-metadata-security`. Library-only today.

   Expected shape:
   ```
   busbar-sf security audit \
     --metadata <dir-or-zip> \
     [--package-xml <path>] \
     [--policies <cedar-bundle-dir>] \
     [--severity <min>] \
     [--output <findings.json>] \
     [--format json|sarif|both] \
     [--sarif <path>] \
     [--fail-on <severity>] \
     [--json]
   ```

   Pipeline:
   - Parse the metadata tree/zip via `sf-mdpkg` (`MdPackage`); use `metadata-etl` for XML where needed. All metadata typing via `busbar_sf_types`.
   - Build the entity graph (`EntityGraphBuilder`) from `MetadataSecurityFacts`.
   - Evaluate Cedar policies — built-in baseline, or a user bundle via `--policies`.
   - Emit `SecurityIssue[]` as JSON (existing serde shape: `severity`, `category`, `description`, `affected_components`, `remediation`).
   - SARIF mapping: `category` → ruleId, `severity` → level (critical/high → error, medium → warning, low/info → note), `affected_components` → locations, `description` + `remediation` → message.
   - Exit non-zero when `--fail-on <severity>` is met.

2. **Binary publication to `busbar-actions/actions-dist`** — same dependency as the other actions.

### Note: Apex-derived findings are dormant

Categories that require Apex parsing (`InsecureDml`, `InjectionVulnerability`, `ExposedEndpoint`, `ApexBypassesMetadataPermissions`) won't fire until the `sf-apex-analyzer` ↔ `typesynth` tree-sitter version conflict (0.24 vs 0.23) is resolved and the `apex` feature is re-enabled in `sf-dependency-graph` / `sf-metadata-security`. Metadata-derived findings (sharing, permission sets, profiles, flows) work without it.

Once the subcommand and binary publication land, tag this action `v1` and consumers can pin it.
