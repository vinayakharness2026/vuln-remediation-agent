# Vulnerability Remediation Agent

A Claude Code agent that automates end-to-end security vulnerability remediation for Harness CI container images. What used to be a manual, multi-hour process per ticket is reduced to a single command.

## What It Does

Given a JIRA ticket number or raw CVE details, the agent:

1. **Fetches the JIRA ticket** and extracts all CVEs, affected packages, and required fix versions
2. **Scans the original image** via the Harness OnDemand Vulnerability Scanner to establish a baseline
3. **Clones the source repo** and analyzes all vulnerability sources — base image, bundled binaries, direct and transitive dependencies
4. **Researches available fixes** by querying GitHub releases and DockerHub for the latest versions
5. **Makes targeted upgrades** to the Dockerfile (and `go.mod` if needed), flagging any major version bumps that need QA attention
6. **Builds and pushes a test image** to DockerHub with tag format `{plugin}-{next-version}--debug` (e.g., `buildx-1.3.14--debug`)
7. **Triggers the OnDemand scanner** on the test image and polls until complete
8. **Fetches scan results** from the Harness STO API and computes the before/after delta
9. **Pauses for your approval**, then opens a GitHub PR with a full report including:
   - Vulnerability delta table (Critical / High / Medium / Low before vs. after)
   - Per-CVE status table (✅ Resolved / ⚠️ Partial / ❌ Blocked with reason)
   - Links to both scan executions in Harness
   - Major version upgrade warnings with QA checklist

## Supported Repositories

| JIRA Image | GitHub Repo |
|---|---|
| `plugins/buildx` | [drone-plugins/drone-buildx](https://github.com/drone-plugins/drone-buildx) |
| `plugins/buildah` | [drone-plugins/drone-buildah](https://github.com/drone-plugins/drone-buildah) |
| `plugins/docker` | [drone-plugins/drone-docker](https://github.com/drone-plugins/drone-docker) |
| `plugins/git` | [drone-plugins/drone-git](https://github.com/drone-plugins/drone-git) |
| `plugins/img` | [drone-plugins/drone-img](https://github.com/drone-plugins/drone-img) |
| `plugins/s3` | [drone-plugins/drone-s3](https://github.com/drone-plugins/drone-s3) |
| `plugins/s3-sync` | [drone-plugins/drone-s3-sync](https://github.com/drone-plugins/drone-s3-sync) |
| `plugins/codedeploy` | [drone-plugins/drone-codedeploy](https://github.com/drone-plugins/drone-codedeploy) |
| `plugins/opsworks` | [drone-plugins/drone-opsworks](https://github.com/drone-plugins/drone-opsworks) |
| `plugins/meltwater-cache` | [drone-plugins/drone-meltwater-cache](https://github.com/drone-plugins/drone-meltwater-cache) |

For any other `plugins/X` image, the agent automatically tries `github.com/drone-plugins/drone-X`.

## Prerequisites

### Tools
- [Claude Code](https://claude.ai/code) v1.0.33+
- Docker Desktop (running)
- `jq` — `brew install jq`

### Environment Variables

Add these to your `~/.zshrc`:

```bash
# Harness
export HARNESS_TOKEN="<your-harness-pat>"        # Profile → API Keys in Harness UI
export HARNESS_ACCOUNT_ID="l7B_kbSEQD2wjrM7PShm5w"
export HARNESS_ORG_ID="Security_and_Compliance"
export HARNESS_PROJECT_ID="ProdSec"
export ONDEMAND_PIPELINE_ID="Ondemand_Vulnerability_Scanner"

# GitHub
export GITHUB_TOKEN="<your-github-pat>"          # github.com → Settings → Developer Settings → PATs

# DockerHub
export DOCKERHUB_USER="<your-dockerhub-username>"
export DOCKERHUB_TOKEN="<your-dockerhub-token>"  # hub.docker.com → Account Settings → Security

# JIRA (optional — only needed if passing a ticket number)
export JIRA_EMAIL="<your-email>@harness.io"
export JIRA_TOKEN="<your-jira-api-token>"        # id.atlassian.com → Security → API tokens
```

Then `source ~/.zshrc`.

## Usage

### Step 1: Launch the agent

```bash
cd /path/to/vuln-remediation-agent/plugins/vuln-remediation
claude --dangerously-skip-permissions
```

### Step 2: Run the command

**With a JIRA ticket:**
```
/remediate CI-21415
```

**With raw CVE details:**
```
/remediate plugins/buildx:1.3.13 CVE-2026-24051 go.opentelemetry.io/otel/sdk@v1.31.0
```

**With multiple CVEs:**
```
/remediate plugins/docker:27.5.1 CVE-2026-24051 CVE-2025-68121
```

### Step 3: Review and approve

The agent runs fully autonomously through all 9 steps. It pauses only before opening the GitHub PR — you review the vulnerability delta and per-CVE table, then approve.

## One-liner Alias

Add to `~/.zshrc` for quick access from anywhere:

```bash
alias vuln-agent='cd /path/to/vuln-remediation-agent/plugins/vuln-remediation && claude --dangerously-skip-permissions'
```

Then just run `vuln-agent`.

## How the Test Image Is Tagged

The agent pushes test images to your DockerHub account using:
```
{DOCKERHUB_USER}/{plugin-name}-test:{plugin-name}-{next-patch-version}--debug
```

Example: original image `plugins/buildx:1.3.13` → test image `vinayakharness/buildx-test:buildx-1.3.14--debug`

## PR Description Format

Every PR opened by the agent includes:

- **Test image** reference and links to both scan executions in Harness
- **Vulnerability delta table** (before vs after counts by severity)
- **Per-CVE status table** with resolution status and reason for any unresolved CVEs
- **Changes made** (Dockerfile diffs with version bump details)
- **Major version upgrade warning** (if any component crossed a major version boundary) with a QA checklist asking for staging validation before merge
