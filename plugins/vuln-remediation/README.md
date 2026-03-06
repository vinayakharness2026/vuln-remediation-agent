# vuln-remediation

A Claude Code plugin that automates end-to-end security vulnerability remediation for Harness CI container images.

## What It Does

Given a JIRA ticket or raw CVE details, this plugin:

1. Parses the ticket to extract image, CVE, vulnerable package, required version
2. Finds and clones the source GitHub repository
3. Analyzes all vulnerability sources (base image, bundled binaries, direct/transitive deps)
4. Researches the latest available fixes via GitHub/DockerHub APIs
5. Makes targeted Dockerfile and dependency upgrades
6. Builds and pushes a test image
7. Triggers your Harness OnDemand vulnerability scan pipeline
8. Parses before/after results and computes reduction
9. Pauses for your approval, then creates a PR with the full report

## Usage

```
# By JIRA ticket
/vuln-remediation:remediate CI-1234

# By raw details
/vuln-remediation:remediate plugins/buildx:1.3.13 CVE-2026-24051
```

## Required Environment Variables

Set these in your shell before starting Claude Code:

```bash
export GITHUB_TOKEN=<github-pat>
export HARNESS_TOKEN=<harness-pat>
export DOCKERHUB_USER=<your-dockerhub-username>
export DOCKERHUB_TOKEN=<your-dockerhub-token>
export HARNESS_ACCOUNT_ID=<account-id>
export HARNESS_ORG_ID=<org-id>
export HARNESS_PROJECT_ID=<project-id>
export ONDEMAND_PIPELINE_ID=<pipeline-id>

# Optional: if your JIRA is separate from Harness
export JIRA_EMAIL=<your-email>
export JIRA_TOKEN=<jira-api-token>
```

## Getting Credentials

| Variable | Where to get it |
|----------|----------------|
| `GITHUB_TOKEN` | github.com → Settings → Developer Settings → PATs |
| `HARNESS_TOKEN` | Harness UI → Profile (top right) → API Keys → New Token |
| `DOCKERHUB_USER/TOKEN` | hub.docker.com → Account Settings → Security |
| `HARNESS_ACCOUNT_ID` | Harness URL: `/account/<id>/...` |
| `ONDEMAND_PIPELINE_ID` | Harness Pipelines → your scanner → URL |

## What the Agent Will NOT Do Without Your Approval

- Push to production image tags
- Create a GitHub PR
- Merge anything

The agent pauses before step 9 (PR creation) and presents a full vulnerability reduction report for your review.

## Plugin Components

- `agents/vuln-remediator.md` — The autonomous agent with the full 9-step workflow
- `commands/remediate.md` — The `/remediate` slash command entry point
