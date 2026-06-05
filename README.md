> [!WARNING]
> **`busbar-actions` is under heavy active development — expect breaking changes.**
> These repositories are public, but **not ready for use yet** — please don't depend on them.
> A pilot is starting soon: **[star and watch the busbar-actions organization](https://github.com/busbar-actions)** for the launch of Discussions and the pilot announcement.

# busbar-actions/sf-permissions-audit

Static, Cedar-backed security analysis of Salesforce **metadata** — no live org
connection required. Surfaces over-privileged permission sets, permissive sharing
models, FLS mismatches, flows running without sharing, permission escalation, and
missing security controls — each with the affected components and a suggested
remediation — as a JSON findings report and an optional SARIF report for GitHub
Code Scanning.

Because it operates on metadata files (a source-format tree or an MDAPI zip), it
chains naturally after `sf-metadata-pull` or runs directly on a committed
`force-app` tree. **No `SF_ACCESS_TOKEN`, OIDC token, or `id-token: write` is
needed.**

## What the binary does

The `sf-permissions-audit` binary owns all logic and UX:

1. Extracts `MetadataSecurityFacts` from the metadata directory
   (`UnifiedFactExtractor`) — PermissionSet / Profile / SharingRules / Flow XML.
2. Runs `sf-metadata-security`'s Cedar-backed analyzer (`SecurityAnalysis`) to
   produce `SecurityIssue`s.
3. Filters findings to those at or above `severity-threshold`.
4. Writes the JSON findings report (and, when `sarif=true`, a SARIF 2.1.0 report
   mapping `category` → ruleId, `severity` → level, `affected_components` →
   locations).
5. Writes a PR-comment-ready markdown body (grouped by category, worst severity +
   count) for the action to post with `gh pr comment`.
6. Emits `GITHUB_OUTPUT`, a `$GITHUB_STEP_SUMMARY` job summary, and workflow
   annotations via the `github-actions-ux` crate.
7. Exits non-zero when any finding meets or exceeds `fail-on-severity`.

> **Apex-derived findings are dormant.** Categories that require Apex parsing
> (`InsecureDml`, `InjectionVulnerability`, `ExposedEndpoint`,
> `ApexBypassesMetadataPermissions`) won't fire until the `sf-apex-analyzer` ↔
> `typesynth` tree-sitter version conflict is resolved and the `apex` feature is
> re-enabled in `sf-metadata-security`. Metadata-derived findings (sharing,
> permission sets, profiles, flows) work today.

## Finding categories

`SharingBypass`, `OverprivilegedPermissionSet`, `SharingRuleConflict`,
`FieldAccessMismatch`, `MissingSecurityControl`, `PermissiveSharingModel`,
`MissingAuditTrail`, `FlowRunsWithoutSharing`, `PermissionEscalation`,
`InsecureExternalAccess`, `CredentialExposure`, plus the Apex-derived categories
above (see the dormant-Apex note).

Severity scale: `critical` / `high` / `medium` / `low` / `info`.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `metadata-path` | yes | — | Directory or MDAPI zip of metadata to audit (e.g. `force-app/main/default`). |
| `package-xml` | no | `` | Path to a `package.xml` to scope the audit. *(Accepted but Cedar cross-validation is not yet wired.)* |
| `policies` | no | `` | Cedar policy bundle (dir of `.cedar` files) for custom org rules. Empty = built-in baseline. *(Accepted but not yet wired.)* |
| `severity-threshold` | no | `medium` | Minimum severity to include. One of `info`, `low`, `medium`, `high`, `critical`. |
| `fail-on-severity` | no | `high` | Exit non-zero if any finding meets or exceeds this severity. `never` disables. |
| `report-path` | no | `security-findings.json` | JSON findings report destination. |
| `sarif` | no | `true` | Also emit a SARIF report. |
| `sarif-path` | no | `security-findings.sarif` | SARIF destination (only used when `sarif=true`). |
| `upload-sarif` | no | `true` | Upload the SARIF to GitHub Code Scanning (needs `security-events: write`). |
| `comment-pr` | no | `true` | Post a findings summary as a PR comment on `pull_request` events (needs `pull-requests: write`). |
| `upload-artifact` | no | `true` | Upload the report(s) as a workflow artifact. |
| `artifact-name` | no | `security-findings` | Artifact name when `upload-artifact=true`. |
| `version` | no | `latest` | `sf-permissions-audit` release tag to download (e.g. `v0.4.2`). |
| `binary-repo` | no | `busbar-actions/actions-dist` | Repo that publishes the `sf-permissions-audit` binary releases. |

## Outputs

| Output | Description |
|---|---|
| `findings-count` | Total findings at or above `severity-threshold`. |
| `critical-count` | Number of Critical-severity findings. |
| `high-count` | Number of High-severity findings. |
| `medium-count` | Number of Medium-severity findings. |
| `threshold-breached` | `"true"` if any finding met or exceeded `fail-on-severity`. |
| `report-path` | Path to the JSON findings report. |
| `sarif-path` | Path to the SARIF report (empty if `sarif=false`). |

## Auth / permissions model

This action reads metadata from the checked-out repo only — there is **no
Salesforce auth and no OIDC**. The permissions it needs are GitHub-native, for
publishing results:

```yaml
permissions:
  contents: read           # checkout
  security-events: write   # upload-sarif → Code Scanning
  pull-requests: write     # comment-pr → PR comment
```

Drop `security-events: write` if you set `upload-sarif: false`, and
`pull-requests: write` if you set `comment-pr: false`.

## Observability

The binary routes outputs, the job summary, and annotations through the
`github-actions-ux` `Reporter` (auto-selecting workflow-command output on GitHub
vs plain output locally). The job summary tallies findings by severity and lists
the top findings; up to 10 findings are emitted as `::error`/`::warning`/
`::notice` annotations; the fail-on breach is surfaced as a final error
annotation. Findings are also persisted to the JSON/SARIF reports and the
artifact, so results survive regardless of run mode.

## Example: audit on a PR touching security-relevant metadata

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
          target: force-app/pulled
          commit: false

      - uses: busbar-actions/sf-permissions-audit@v1
        with:
          metadata-path: force-app/pulled
          policies: .busbar/cedar-policies   # accepted; cross-validation not yet wired
          fail-on-severity: critical
```
