# Vulnerability Remediation Agent

A Claude Code agent that automates end-to-end security vulnerability remediation for Harness CI container images. What used to be a manual multi-hour process per ticket is reduced to a single command.

---

## How It Works

Given a JIRA ticket number, a Trivy scan result, or raw CVE details, the agent:

1. Fetches the ticket and extracts all CVEs, affected packages, and required fix versions
2. Scans the original image via the Harness OnDemand Vulnerability Scanner (baseline)
3. Clones the source repo and identifies vulnerability sources (base image, bundled binaries, Go deps)
4. Finds the **minimum safe version** that fixes each CVE — not necessarily the latest
5. Makes targeted Dockerfile upgrades, flags any major version bumps that need QA
6. Builds and pushes a test image to DockerHub (`{plugin}-{next-version}--debug`)
7. **Runs two scans in parallel** on both baseline and test image:
   - Harness OnDemand scanner (Prisma Cloud) via MCP
   - Trivy locally — catches CVEs the OnDemand scanner may miss
8. Merges results from both scanners and computes before/after delta
9. **Pauses for your approval**, then opens a GitHub PR with the full report

---

## Setup (One Time)

### 1. Prerequisites

- [Claude Code](https://claude.ai/code) — install with `npm install -g @anthropic-ai/claude-code`
- Docker Desktop — must be running when the agent builds images
- `jq` — `brew install jq`
- `node` / `npx` — required for the Harness MCP server (`brew install node`)

### 2. Clone the repo

```bash
git clone https://github.com/vinayakharness2026/vuln-remediation-agent
cd vuln-remediation-agent
```

### 3. Create your `.env` file

```bash
cp .env.example .env
```

Open `.env` and fill in your personal tokens:

```bash
HARNESS_TOKEN=pat.l7B_kbSEQD2wjrM7PShm5w.<your-token>
GITHUB_TOKEN=ghp_<your-token>
DOCKERHUB_USER=<your-dockerhub-username>
DOCKERHUB_TOKEN=dckr_pat_<your-token>
JIRA_EMAIL=<your-name>@harness.io
JIRA_TOKEN=<your-atlassian-api-token>
```

Where to get each token:

| Token | Where to get it |
|---|---|
| `HARNESS_TOKEN` | Harness UI → click your avatar (top right) → My Profile → API Keys → New Token |
| `GITHUB_TOKEN` | github.com → Settings → Developer Settings → Personal Access Tokens → New token (classic). Scopes: `repo` |
| `DOCKERHUB_USER` | Your DockerHub username |
| `DOCKERHUB_TOKEN` | hub.docker.com → Account Settings → Security → New Access Token |
| `JIRA_EMAIL` | Your `@harness.io` email |
| `JIRA_TOKEN` | id.atlassian.com → Security → API tokens → Create API token |

> The `.env` file is gitignored. Your tokens are never committed.
>
> Harness infrastructure values (account ID, org, project, pipeline ID) are already hardcoded in the agent — you don't need to set them.

### 4. Harness MCP server (pre-configured, no setup needed)

The agent uses [Harness MCP v2](https://github.com/thisrohangupta/harness-mcp-v2) to trigger pipelines and check execution status — no raw API calls. The MCP server is already configured in `.claude/settings.json`. It starts automatically when Claude Code launches from the plugin directory.

To verify it's connected, launch the agent and run:
```
What is the status of the Ondemand_Vulnerability_Scanner pipeline?
```
Claude will use the MCP to answer with real Harness data.

---

## How to Use

### Step 1: Launch the agent

```bash
cd vuln-remediation-agent/plugins/vuln-remediation
claude --dangerously-skip-permissions
```

> `--dangerously-skip-permissions` lets the agent run shell commands autonomously without asking for approval on every step. It will still pause before opening a PR.

### Step 2: Run the remediate command

**Single JIRA ticket:**
```
/remediate CI-21415
```

**Multiple tickets for the same image → single PR:**
```
/remediate CI-21415 CI-21416 CI-21417
```

**Multiple tickets for different images → one PR per repo:**
```
/remediate CI-21415 CI-21500
```

**From a Trivy scan result (no JIRA ticket needed):**
```
/remediate --trivy /tmp/trivy-results.json
```

**Trivy + JIRA combined:**
```
/remediate CI-21415 --trivy /tmp/trivy-results.json
```

**Raw CVE details (no JIRA ticket):**
```
/remediate plugins/buildx:1.3.13 CVE-2026-24051 go.opentelemetry.io/otel/sdk@v1.31.0
```

### Step 3: Watch it run

The agent works through all steps autonomously. Typical runtime is 15–25 minutes depending on image build time. You'll see output for each step in real time.

### Step 4: Review and approve the PR

The agent stops before creating the PR and shows you a full report:

- Vulnerability delta (before vs after, by severity)
- Per-CVE status for each ticket (✅ Resolved / ⚠️ Partial / ❌ Blocked + reason)
- Exactly what changed in the Dockerfile
- Links to both scan runs in Harness STO
- A QA warning if any component crossed a major version boundary

Type `yes` to approve. The agent opens the PR on GitHub.

---

## Optional: Alias for quick access

Add to `~/.zshrc`:

```bash
alias vuln-agent='cd /path/to/vuln-remediation-agent/plugins/vuln-remediation && claude --dangerously-skip-permissions'
```

Then just run `vuln-agent` from anywhere.

---

## What the PR Looks Like

```
## Vulnerability Remediation: plugins/buildx

Tickets: CI-1234, CI-1235
Test image: vinayakharness/buildx-test:buildx-1.3.14--debug
Baseline scan: [link] | After scan: [link]

### Vulnerability Delta
| Severity | Before (1.3.13) | After (buildx-1.3.14--debug) | Change |
|----------|-----------------|-------------------------------|--------|
| Critical | 3               | 0                             | -3     |
| High     | 5               | 2                             | -3     |
| Medium   | 12              | 7                             | -5     |
| Low      | 4               | 2                             | -2     |

### Per-Ticket CVE Status

#### CI-1234 — [link]
| CVE            | Package    | Before  | After   | Required | Status     | Reason |
|----------------|------------|---------|---------|----------|------------|--------|
| CVE-2026-24051 | otel/sdk   | v1.31.0 | v1.40.1 | v1.40.0+ | ✅ Resolved |       |
| CVE-2025-68121 | crypto/tls | v1.25.6 | v1.25.7 | v1.25.7+ | ✅ Resolved |       |

Code changes: buildx v0.23.0 → v0.19.3 (first release with otel/sdk v1.40.1)

#### CI-1235 — [link]
| CVE            | Package | Before     | After      | Required | Status     | Reason                    |
|----------------|---------|------------|------------|----------|------------|---------------------------|
| CVE-2025-60876 | busybox | 1.37.0-r30 | 1.37.0-r30 | unknown  | ❌ Blocked | No upstream fix available |

### Changes Made
- BUILDX_URL: v0.23.0 → v0.19.3 (patch bump, first version with otel/sdk fix)

> ⚠️ Major version upgrades included — run sanity in QA before merging
> - docker base: 28.1.1-dind → 29.0.0-dind
```

---

## Test Image Naming

```
{DOCKERHUB_USER}/{plugin-name}-test:{plugin-name}-{next-patch-version}--debug
```

Example: `plugins/buildx:1.3.13` → `vinayakharness/buildx-test:buildx-1.3.14--debug`

---

## Supported Images

### Harness Internal (Harness Code repos)

| Image | Source Repo | Priority | Notes |
|---|---|---|---|
| `harness/ci-addon` | [harness-core](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/module/code/orgs/PROD/projects/Harness_Commons/repos/harness-core) | P1 | Rootless variant also published |
| `harness/ci-lite-engine` | [harness-core](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/module/code/orgs/PROD/projects/Harness_Commons/repos/harness-core) | P1 | Rootless variant also published |
| `harness/harness-cache-server` | [harness-cache](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/module/code/orgs/PROD/projects/Harness_Commons/repos/harness-cache) | P1 | |
| `harness/drone-git` | [drone-git](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/module/code/orgs/PROD/projects/Harness_Commons/repos/drone-git) | P1 | |

### Kaniko Plugins

| Image | Source Repo | Priority |
|---|---|---|
| `plugins/kaniko` | [drone/drone-kaniko](https://github.com/drone/drone-kaniko) | P1 |
| `plugins/kaniko-ecr` | [drone/drone-kaniko](https://github.com/drone/drone-kaniko) | P1 |
| `plugins/kaniko-gcr` | [drone/drone-kaniko](https://github.com/drone/drone-kaniko) | P1 |
| `plugins/kaniko-acr` | [drone/drone-kaniko](https://github.com/drone/drone-kaniko) | P1 |

### Docker Plugins

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

### P2 Plugins

| Image | Source Repo |
|---|---|
| `plugins/buildah` | [drone-plugins/drone-buildah](https://github.com/drone-plugins/drone-buildah) |
| `plugins/buildah-docker` | [drone-plugins/drone-buildah](https://github.com/drone-plugins/drone-buildah) |
| `plugins/img` | [drone-plugins/drone-img](https://github.com/drone-plugins/drone-img) |
| `email` | [harness-community/drone-email](https://github.com/harness-community/drone-email) |
| `githubaction` | [drone-plugins/github-actions](https://github.com/drone-plugins/github-actions) |
| `plugins/test-analysis` | [harness-community/test-analysis](https://github.com/harness-community/test-analysis) |
| `plugins/artifact-metadata-publisher` | [drone-plugins/artifact-metadata-publisher](https://github.com/drone-plugins/artifact-metadata-publisher) |
| `plugins/aws-oidc` | [harness-community/drone-aws-oidc](https://github.com/harness-community/drone-aws-oidc) |
| `plugins/gcp-oidc` | [harness-community/drone-gcp-oidc](https://github.com/harness-community/drone-gcp-oidc) |
| `plugins/azure-oidc` | [harness-community/drone-azure-oidc](https://github.com/harness-community/drone-azure-oidc) |
| `plugins/image-migration` | [harness-community/drone-docker-image-migration](https://github.com/harness-community/drone-docker-image-migration) |
| `plugins/codedeploy` | [drone-plugins/drone-codedeploy](https://github.com/drone-plugins/drone-codedeploy) |
| `plugins/opsworks` | [drone-plugins/drone-opsworks](https://github.com/drone-plugins/drone-opsworks) |
