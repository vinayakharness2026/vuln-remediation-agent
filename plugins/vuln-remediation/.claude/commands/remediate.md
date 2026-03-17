---
description: Kick off end-to-end vulnerability remediation. Accepts one or more JIRA ticket numbers (or raw CVE details). Multiple tickets for the same repo are merged into a single PR with per-ticket breakdown.
---

Invoke the `vuln-remediator` agent to remediate the vulnerabilities described below.

## Input

The user has provided: $ARGUMENTS

Parse `$ARGUMENTS` as a space or comma separated list. Each item is one of:
- A JIRA ticket number: `CI-1234`
- Raw vulnerability details: `plugins/buildx:1.3.13 CVE-2026-24051 go.opentelemetry.io/otel/sdk@v1.31.0`
- A path to a Trivy JSON scan result: `--trivy /tmp/trivy-results.json`

### Examples
```
/remediate CI-1234                                    # single JIRA ticket
/remediate CI-1234 CI-1235 CI-1236                   # multiple tickets → one PR per repo
/remediate --trivy /tmp/trivy.json                   # trivy scan as input
/remediate CI-1234 --trivy /tmp/trivy.json           # JIRA + trivy combined
```

If `--trivy <path>` is provided, parse the Trivy JSON file:
```bash
cat "$TRIVY_FILE" | jq '[.Results[].Vulnerabilities // [] | .[] | {
  cve: .VulnerabilityID,
  package: .PkgName,
  installed: .InstalledVersion,
  fixed: .FixedVersion,
  severity: .Severity,
  title: .Title
}]'
```
Extract the image name from `.ArtifactName` in the Trivy output. Merge CVEs from Trivy with any CVEs from JIRA tickets, deduplicating by CVE ID.

If no arguments were provided, ask the user:
```
Please provide one or more of:
  - JIRA ticket numbers (e.g., CI-1234 CI-1235)
  - Raw CVE details
  - A Trivy scan file: --trivy /path/to/trivy-results.json
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

⚠️ **Run this ONCE at the very start. Never repeat it. Never prefix later commands with `source .env`.**

```bash
# Hardcoded infrastructure vars — set once, reuse throughout
export HARNESS_ACCOUNT_ID="l7B_kbSEQD2wjrM7PShm5w"
export HARNESS_ORG_ID="Security_and_Compliance"
export HARNESS_PROJECT_ID="ProdSec"
export ONDEMAND_PIPELINE_ID="Ondemand_Vulnerability_Scanner"

# Check tokens are present — user launched with `source .env && claude`
MISSING=""
for var in HARNESS_TOKEN GITHUB_TOKEN DOCKERHUB_USER DOCKERHUB_TOKEN JIRA_EMAIL JIRA_TOKEN; do
  [ -z "$(printenv $var)" ] && MISSING="$MISSING $var"
done

if [ -n "$MISSING" ]; then
  echo "ERROR - missing:$MISSING"
  echo "Exit claude, run: source /path/to/.env && claude --dangerously-skip-permissions"
  exit 1
fi
echo "OK - all tokens present"
```

After this block succeeds, use `$HARNESS_TOKEN`, `$GITHUB_TOKEN` etc. directly in all subsequent commands. **Do not source any file again.**

## Execution

Once all tickets are fetched and grouped, proceed with the full remediation workflow in the `vuln-remediator` agent, passing the complete merged CVE list and the per-ticket breakdown for use in the PR description.
