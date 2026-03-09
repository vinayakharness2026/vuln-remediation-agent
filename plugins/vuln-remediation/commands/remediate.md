---
description: Kick off end-to-end vulnerability remediation. Accepts one or more JIRA ticket numbers (or raw CVE details). Multiple tickets for the same repo are merged into a single PR with per-ticket breakdown.
---

Invoke the `vuln-remediator` agent to remediate the vulnerabilities described below.

## Input

The user has provided: $ARGUMENTS

Parse `$ARGUMENTS` as a space or comma separated list. Each item is either:
- A JIRA ticket number: `CI-1234`
- Raw vulnerability details: `plugins/buildx:1.3.13 CVE-2026-24051 go.opentelemetry.io/otel/sdk@v1.31.0`

### Examples
```
/remediate CI-1234                              # single ticket
/remediate CI-1234 CI-1235 CI-1236             # multiple tickets, likely same repo → one PR
/remediate CI-1234 CI-1235                     # two tickets, could be different repos → separate PRs per repo
```

If no arguments were provided, ask the user:
```
Please provide one or more JIRA ticket numbers (e.g., CI-1234 CI-1235) or raw CVE details.
```

## Step 1: Fetch and Group All Tickets

For each JIRA ticket in the list, fetch it:
```bash
curl -s "https://harness.atlassian.net/rest/api/3/issue/CI-XXXX" \
  -H "Authorization: Basic $(printf '%s' "$JIRA_EMAIL:$JIRA_TOKEN" | base64)" \
  -H "Content-Type: application/json"
```

Extract from each ticket:
- Image name and tag
- All CVE IDs
- Vulnerable packages and versions
- Required fix versions
- Severity

**Group tickets by image name.** If all tickets point to the same image (e.g., `plugins/buildx:1.3.13`), process them together as one remediation run producing one PR. If tickets point to different images, process each group separately and produce one PR per repo.

## Pre-flight Checks

### 1. Set hardcoded Harness infrastructure vars (same for all team members)
```bash
export HARNESS_ACCOUNT_ID="l7B_kbSEQD2wjrM7PShm5w"
export HARNESS_ORG_ID="Security_and_Compliance"
export HARNESS_PROJECT_ID="ProdSec"
export ONDEMAND_PIPELINE_ID="Ondemand_Vulnerability_Scanner"
```

### 2. Load personal tokens from .env file
```bash
for dir in . .. ../.. ../../..; do
  if [ -f "$dir/.env" ]; then
    set -a && source "$dir/.env" && set +a
    echo "Loaded .env from $dir"
    break
  fi
done
```

### 3. Verify personal tokens are present
```bash
for var in HARNESS_TOKEN GITHUB_TOKEN DOCKERHUB_USER DOCKERHUB_TOKEN JIRA_EMAIL JIRA_TOKEN; do
  val=$(printenv "$var")
  if [ -z "$val" ]; then echo "MISSING: $var"; else echo "OK: $var"; fi
done
```

If any personal tokens are missing, stop and tell the user:
```
Some tokens are missing. Create a .env file in the plugin directory:

  cp .env.example .env
  # then fill in your tokens

Tokens needed:
  HARNESS_TOKEN  — Harness UI → Profile (top right) → API Keys → New Token
  GITHUB_TOKEN   — github.com → Settings → Developer Settings → PATs
  DOCKERHUB_USER / DOCKERHUB_TOKEN — hub.docker.com → Account Settings → Security
  JIRA_EMAIL     — your @harness.io email
  JIRA_TOKEN     — id.atlassian.com → Security → API tokens
```

## Execution

Once all tickets are fetched and grouped, proceed with the full remediation workflow in the `vuln-remediator` agent, passing the complete merged CVE list and the per-ticket breakdown for use in the PR description.
