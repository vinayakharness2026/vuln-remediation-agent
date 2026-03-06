---
description: Kick off end-to-end vulnerability remediation for a container image. Accepts a JIRA ticket number or raw CVE details and delegates to the vuln-remediator agent.
---

Invoke the `vuln-remediator` agent to remediate the security vulnerability described below.

## Input

The user has provided: $ARGUMENTS

If `$ARGUMENTS` is a JIRA ticket number (e.g., `CI-1234`), fetch the ticket and extract:
- Image name and tag
- CVE ID(s)
- Vulnerable package(s) and versions
- Required fix version(s)
- Severity

If `$ARGUMENTS` contains raw details, parse them directly.

If no arguments were provided, ask the user:
```
Please provide one of:
1. A JIRA ticket number (e.g., CI-1234)
2. Raw vulnerability details:
   - Image: plugins/buildx:1.3.13
   - CVE: CVE-2026-24051
   - Vulnerable package: go.opentelemetry.io/otel/sdk@v1.31.0
   - Required fix: v1.40.0+
   - Severity: High
```

## Pre-flight Checks

Before starting, verify required environment variables are set:
```bash
for var in GITHUB_TOKEN HARNESS_TOKEN DOCKERHUB_USER DOCKERHUB_TOKEN HARNESS_ACCOUNT_ID ONDEMAND_PIPELINE_ID; do
  if [ -z "${!var}" ]; then
    echo "MISSING: $var"
  else
    echo "OK: $var"
  fi
done
```

If any are missing, tell the user which ones are needed and where to get them:
- `GITHUB_TOKEN`: github.com → Settings → Developer Settings → Personal Access Tokens
- `HARNESS_TOKEN`: Harness UI → Profile → API Keys → New Token
- `DOCKERHUB_USER` / `DOCKERHUB_TOKEN`: hub.docker.com → Account Settings → Security
- `HARNESS_ACCOUNT_ID`: Harness UI URL (`/account/<id>/...`)
- `ONDEMAND_PIPELINE_ID`: Harness UI → Pipelines → your OnDemand scanner → copy ID from URL

## Execution

Once input is parsed and environment is confirmed, proceed with the full 9-step remediation workflow defined in the `vuln-remediator` agent.
