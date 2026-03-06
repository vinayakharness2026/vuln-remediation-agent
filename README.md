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

## Supported Images

### Harness Internal (hosted on Harness Code)

| Image | Source Repo | Priority | Notes |
|---|---|---|---|
| `harness/ci-addon` | [harness-core](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/module/code/orgs/PROD/projects/Harness_Commons/repos/harness-core) | P1 | Rootless variant also published |
| `harness/ci-lite-engine` | [harness-core](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/module/code/orgs/PROD/projects/Harness_Commons/repos/harness-core) | P1 | Rootless variant also published |
| `harness/harness-cache-server` | [harness-cache](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/module/code/orgs/PROD/projects/Harness_Commons/repos/harness-cache) | P1 | |
| `harness/drone-git` | [drone-git](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/module/code/orgs/PROD/projects/Harness_Commons/repos/drone-git) | P1 | Optimised variant also published |

### Kaniko Plugins

| Image | Source Repo | Priority |
|---|---|---|
| `plugins/kaniko` | [drone/drone-kaniko](https://github.com/drone/drone-kaniko) | P1 |
| `plugins/kaniko-ecr` | [drone/drone-kaniko](https://github.com/drone/drone-kaniko) | P1 |
| `plugins/kaniko-gcr` | [drone/drone-kaniko](https://github.com/drone/drone-kaniko) | P1 |
| `plugins/kaniko-acr` | [drone/drone-kaniko](https://github.com/drone/drone-kaniko) | P1 |

### Docker Plugins (all from same repo)

| Image | Source Repo | Priority | Notes |
|---|---|---|---|
| `plugins/docker` | [drone-plugins/drone-docker](https://github.com/drone-plugins/drone-docker) | P1 | |
| `plugins/ecr` | [drone-plugins/drone-docker](https://github.com/drone-plugins/drone-docker) | P1 | |
| `plugins/acr` | [drone-plugins/drone-docker](https://github.com/drone-plugins/drone-docker) | P1 | |
| `plugins/gcr` | [drone-plugins/drone-docker](https://github.com/drone-plugins/drone-docker) | P1 | Deprecated |
| `plugins/gar` | [drone-plugins/drone-docker](https://github.com/drone-plugins/drone-docker) | P1 | |

### Buildx Plugins

| Image | Source Repo | Priority |
|---|---|---|
| `plugins/buildx` | [drone-plugins/drone-buildx](https://github.com/drone-plugins/drone-buildx) | P1 |
| `plugins/buildx-ecr` | [drone-plugins/drone-buildx-ecr](https://github.com/drone-plugins/drone-buildx-ecr) | P1 |
| `plugins/buildx-acr` | [drone-plugins/drone-buildx-acr](https://github.com/drone-plugins/drone-buildx-acr) | P1 |
| `plugins/buildx-gcr` | [drone-plugins/drone-buildx-gcr](https://github.com/drone-plugins/drone-buildx-gcr) | P1 |
| `plugins/buildx-gar` | [drone-plugins/drone-buildx-gar](https://github.com/drone-plugins/drone-buildx-gar) | P1 |

### Storage & Artifact Plugins

| Image | Source Repo | Priority |
|---|---|---|
| `plugins/s3` | [drone-plugins/drone-s3](https://github.com/drone-plugins/drone-s3) | P1 |
| `plugins/gcs` | [drone-plugins/drone-gcs](https://github.com/drone-plugins/drone-gcs) | P1 |
| `plugins/artifactory` | [harness/drone-artifactory](https://github.com/harness/drone-artifactory) | P1 |
| `plugins/cache` | [drone-plugins/drone-meltwater-cache](https://github.com/drone-plugins/drone-meltwater-cache) | P1 |
| `plugins/s3-sync` | [drone-plugins/drone-s3-sync](https://github.com/drone-plugins/drone-s3-sync) | P1 |

### Other P1 Plugins

| Image | Source Repo | Priority |
|---|---|---|
| `plugins/buildah` | [drone-plugins/drone-buildah](https://github.com/drone-plugins/drone-buildah) | P2 |
| `plugins/img` | [drone-plugins/drone-img](https://github.com/drone-plugins/drone-img) | P1 |

### P2 Plugins

| Image | Source Repo | Notes |
|---|---|---|
| `email` | [harness-community/drone-email](https://github.com/harness-community/drone-email) | |
| `githubaction` | [drone-plugins/github-actions](https://github.com/drone-plugins/github-actions) | |
| `plugins/test-analysis` | [harness-community/test-analysis](https://github.com/harness-community/test-analysis) | |
| `plugins/artifact-metadata-publisher` | [drone-plugins/artifact-metadata-publisher](https://github.com/drone-plugins/artifact-metadata-publisher) | |
| `plugins/aws-oidc` | [harness-community/drone-aws-oidc](https://github.com/harness-community/drone-aws-oidc) | |
| `plugins/gcp-oidc` | [harness-community/drone-gcp-oidc](https://github.com/harness-community/drone-gcp-oidc) | |
| `plugins/azure-oidc` | [harness-community/drone-azure-oidc](https://github.com/harness-community/drone-azure-oidc) | |
| `plugins/buildah-docker` | [drone-plugins/drone-buildah](https://github.com/drone-plugins/drone-buildah) | Same repo as buildah |
| `plugins/image-migration` | [harness-community/drone-docker-image-migration](https://github.com/harness-community/drone-docker-image-migration) | |
| `plugins/codedeploy` | [drone-plugins/drone-codedeploy](https://github.com/drone-plugins/drone-codedeploy) | |
| `plugins/opsworks` | [drone-plugins/drone-opsworks](https://github.com/drone-plugins/drone-opsworks) | |

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

**Single JIRA ticket:**
```
/remediate CI-21415
```

**Multiple JIRA tickets (same repo → one PR):**
```
/remediate CI-21415 CI-21416 CI-21417
```

**Multiple JIRA tickets (different repos → one PR per repo):**
```
/remediate CI-21415 CI-21500
```
The agent groups tickets by image. If `CI-21415` is for `plugins/buildx` and `CI-21500` is for `plugins/docker`, it opens two separate PRs — one per repo.

**Raw CVE details:**
```
/remediate plugins/buildx:1.3.13 CVE-2026-24051 go.opentelemetry.io/otel/sdk@v1.31.0
```

### Step 3: Review and approve

The agent runs fully autonomously. It pauses only before opening the GitHub PR — you review the full report, then approve.

### What the PR looks like for multiple tickets

Each PR contains:

- Links to all JIRA tickets it covers
- A combined vulnerability delta table (total before vs after across all tickets)
- A **per-ticket section** for each ticket, showing:
  - Ticket summary and JIRA link
  - CVE status table (✅ Resolved / ⚠️ Partial / ❌ Blocked with reason)
  - Dockerfile changes made specifically for that ticket's CVEs
- Links to both the baseline and after scan executions in Harness
- A major version upgrade warning (if any component crossed a major version boundary) with a QA checklist

Example PR structure for `/remediate CI-1234 CI-1235`:

```
## Vulnerability Remediation: plugins/buildx

Tickets: CI-1234, CI-1235
Baseline scan: [link] | After scan: [link]

### Vulnerability Delta
| Severity | Before (1.3.13) | After (buildx-1.3.14--debug) | Change |
| Critical | 3               | 0                            | -3     |
| High     | 5               | 2                            | -3     |

### Per-Ticket CVE Status

#### CI-1234
| CVE            | Package   | Before  | After   | Status      |
| CVE-2026-24051 | otel/sdk  | v1.31.0 | v1.38.0 | ⚠️ Partial  |
| CVE-2025-68121 | crypto/tls| v1.25.6 | v1.25.7 | ✅ Resolved |
Code changes: docker base 28→29, buildx v0.23→v0.31.1

#### CI-1235
| CVE            | Package | Before     | After      | Status      |
| CVE-2025-60876 | busybox | 1.37.0-r30 | 1.37.0-r30 | ❌ Blocked  |
| CVE-2026-23992 | go-tuf  | v2.3.0     | v2.3.1     | ✅ Resolved |
Code changes: resolved transitively via base image upgrade

### Changes Made
- FROM docker:28.1.1-dind → 29.2.1-dind  (MAJOR ⚠️)
- BUILDX_URL v0.23.0 → v0.31.1

> ⚠️ Major version upgrades — run sanity in QA before merging
```

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
