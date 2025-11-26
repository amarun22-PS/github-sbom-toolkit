# Quick Start Guide

This guide explains how to use the GitHub Actions workflows to automate SBOM security scanning for your organization.

## Prerequisites

1. **GitHub Token**: You need a GitHub token with the following scopes:
   - `repo` - For accessing repository SBOMs
   - `read:org` - For listing organization repositories  
   - `security_events` (write) - For SARIF upload to Code Scanning (optional)

2. **Repository Setup**: Fork or use this repository as your central scanning hub.

## Available Workflows

| Workflow | Purpose | Schedule |
|----------|---------|----------|
| `sync-malware-advisories.yml` | Download malware advisories from GitHub Advisory Database | 1st of month, 00:00 UTC |
| `sync-sboms.yml` | Collect SBOMs from all repos in an organization | 1st of month, 01:00 UTC |
| `match-malware.yml` | Match SBOMs against advisories, generate reports | 2nd of month, 02:00 UTC |
| `full-scan.yml` | Run all three steps in sequence | 1st of month, 03:00 UTC |

## Quick Start: Manual Run

### Option 1: Run Full Scan (Recommended)

The easiest way to scan an organization is to run the full scan workflow:

1. Go to **Actions** tab in your repository
2. Select **"Full SBOM Security Scan"** from the left sidebar
3. Click **"Run workflow"**
4. Fill in the inputs:
   - **organization**: Your GitHub organization (e.g., `amarun22-PS`)
   - Other options are optional with sensible defaults
5. Click **"Run workflow"**

### Option 2: Run Individual Workflows

You can also run workflows separately:

#### Step 1: Sync Malware Advisories
```
Actions → "Sync Malware Advisories" → Run workflow
```
- Optional: Set `malware_since` to fetch only recent advisories (e.g., `2025-01-01`)

#### Step 2: Sync Organization SBOMs
```
Actions → "Sync Organization SBOMs" → Run workflow
```
- Required: Set `organization` to your target org
- Optional: Adjust `concurrency` and `sbom_delay` for rate limiting

#### Step 3: Match Malware
```
Actions → "Match Malware" → Run workflow
```
- Required: Set `organization` (must match Step 2)
- Optional: Set `malware_cutoff`, enable `upload_sarif`

## Reusing Workflows from Another Repository

These workflows support `workflow_call`, so you can invoke them from your own repository:

```yaml
# .github/workflows/security-scan.yml
name: Weekly Security Scan

on:
  schedule:
    - cron: '0 0 * * 0'  # Every Sunday
  workflow_dispatch:
    inputs:
      organization:
        description: 'Organization to scan'
        required: true
        default: 'my-org'

jobs:
  full-scan:
    uses: amarun22-PS/github-sbom-toolkit/.github/workflows/full-scan.yml@main
    with:
      organization: ${{ inputs.organization || 'my-org' }}
      generate_sarif: true
      upload_sarif: false
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

### Scanning Multiple Organizations

```yaml
name: Multi-Org Security Scan

on:
  schedule:
    - cron: '0 0 1 * *'
  workflow_dispatch:

jobs:
  scan-org-1:
    uses: amarun22-PS/github-sbom-toolkit/.github/workflows/full-scan.yml@main
    with:
      organization: org-1
    secrets:
      token: ${{ secrets.SCAN_TOKEN }}

  scan-org-2:
    uses: amarun22-PS/github-sbom-toolkit/.github/workflows/full-scan.yml@main
    with:
      organization: org-2
    secrets:
      token: ${{ secrets.SCAN_TOKEN }}
```

## Workflow Inputs Reference

### Full Scan (`full-scan.yml`)

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `organization` | string | Yes | - | GitHub organization to scan |
| `malware_since` | string | No | - | ISO date for incremental advisory sync |
| `malware_cutoff` | string | No | - | Ignore advisories before this date |
| `ignore_unbounded` | boolean | No | `false` | Ignore advisories with unbounded version ranges |
| `generate_sarif` | boolean | No | `true` | Generate SARIF output files |
| `upload_sarif` | boolean | No | `false` | Upload SARIF to Code Scanning |
| `concurrency` | number | No | `5` | Parallel SBOM fetches |
| `sbom_delay` | number | No | `3000` | Delay (ms) between SBOM requests |

### Sync Malware Advisories (`sync-malware-advisories.yml`)

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `malware_since` | string | No | - | ISO date to fetch advisories from |

### Sync SBOMs (`sync-sboms.yml`)

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `organization` | string | Yes | - | GitHub organization to collect SBOMs from |
| `concurrency` | number | No | `5` | Number of parallel SBOM fetches |
| `sbom_delay` | number | No | `3000` | Delay in ms between SBOM fetch requests |

### Match Malware (`match-malware.yml`)

| Input | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `organization` | string | Yes | - | GitHub organization to match |
| `malware_cutoff` | string | No | - | Ignore advisories before this ISO date |
| `ignore_unbounded` | boolean | No | `false` | Ignore unbounded version range advisories |
| `generate_sarif` | boolean | No | `true` | Generate SARIF output files |
| `upload_sarif` | boolean | No | `false` | Upload SARIF to Code Scanning |

## Output Artifacts

After each workflow run, artifacts are available for download:

| Artifact Name Pattern | Contents | Retention |
|-----------------------|----------|-----------|
| `malware-advisories-YYYYMMDD_HHMMSS` | Downloaded advisory JSON | 90 days |
| `malware-advisories-cache` | Latest advisory cache (rolling) | 90 days |
| `sbom-{org}-YYYYMMDD_HHMMSS` | Organization SBOMs | 90 days |
| `sbom-cache-{org}` | Latest SBOM cache (rolling) | 90 days |
| `malware-report-{org}-YYYYMMDD_HHMMSS` | Match results JSON | 90 days |
| `sarif-{org}-YYYYMMDD_HHMMSS` | SARIF files for Code Scanning | 90 days |

## Example: Scanning `amarun22-PS` Organization

1. Navigate to: `Actions` → `Full SBOM Security Scan`
2. Click `Run workflow`
3. Enter:
   - **organization**: `amarun22-PS`
   - **generate_sarif**: ✓ (checked)
4. Click `Run workflow`
5. Wait for completion (~5-15 minutes depending on org size)
6. Download artifacts from the workflow run summary

## Troubleshooting

### Rate Limiting
If you encounter rate limit errors:
- Increase `sbom_delay` (e.g., `5000` ms)
- Decrease `concurrency` (e.g., `3`)

### Missing SBOMs
- Ensure the token has access to the target organization
- Some repositories may not have dependency graphs enabled

### SARIF Upload Fails
- Verify the token has `security_events` write permission
- Check that Code Scanning is enabled for the target repositories

## Token Permissions

For a **Fine-grained PAT**:
- `Contents`: Read-only
- `Code scanning alerts`: Read and write (if using `upload_sarif`)

For a **Classic PAT**:
- `repo`
- `read:org`
- `security_events` (if using `upload_sarif`)
