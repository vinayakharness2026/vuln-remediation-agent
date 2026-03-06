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

Before starting, verify required environment variables are set:
```bash
for var in GITHUB_TOKEN HARNESS_TOKEN DOCKERHUB_USER DOCKERHUB_TOKEN HARNESS_ACCOUNT_ID ONDEMAND_PIPELINE_ID; do
  val=$(printenv "$var")
  if [ -z "$val" ]; then echo "MISSING: $var"; else echo "OK: $var"; fi
done
```

If any are missing, tell the user which ones are needed and where to get them:
- `GITHUB_TOKEN`: github.com → Settings → Developer Settings → Personal Access Tokens
- `HARNESS_TOKEN`: Harness UI → Profile → API Keys → New Token
- `DOCKERHUB_USER` / `DOCKERHUB_TOKEN`: hub.docker.com → Account Settings → Security
- `HARNESS_ACCOUNT_ID`: from Harness UI URL (`/account/<id>/...`)
- `ONDEMAND_PIPELINE_ID`: Harness UI → Pipelines → your OnDemand scanner → copy ID from URL

## Execution

Once all tickets are fetched and grouped, proceed with the full remediation workflow in the `vuln-remediator` agent, passing the complete merged CVE list and the per-ticket breakdown for use in the PR description.
