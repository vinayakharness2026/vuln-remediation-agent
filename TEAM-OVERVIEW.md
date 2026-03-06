# Automated Vulnerability Remediation with Claude Code

Hey team,

I wanted to share something I've been working on that I think can meaningfully reduce the toil around our security vulnerability remediation process.

---

## What Is It?

We now have an **automated CVE remediation workflow** built on top of Claude Code. Given a JIRA ticket number or raw CVE details, it handles the entire remediation lifecycle end-to-end — from identifying the root cause all the way to opening a reviewed, scan-verified GitHub PR — with a human approval gate before any irreversible action.

**Real example run today:**

| | |
|---|---|
| **Input (JIRA)** | [CI-21050](https://harness.atlassian.net/browse/CI-21050) — `CVE-2026-24051` in `plugins/buildx:1.3.13` |
| **Vulnerable package** | `go.opentelemetry.io/otel/sdk@v1.31.0` |
| **Output (PR)** | [drone-plugins/drone-buildx#91](https://github.com/drone-plugins/drone-buildx/pull/91) |
| **Time taken** | ~15 minutes (vs. 2–4 hours manually) |

The PR includes Dockerfile upgrades, a test image pushed to DockerHub, a full Harness OnDemand scan, and a before/after vulnerability delta — all documented automatically.

---

## Why Was This Built?

Security vulnerability remediation today is a largely manual, repetitive process:

1. Receive a CVE report or JIRA ticket
2. Find which source repo the image comes from
3. Trace whether the vulnerable package is a direct dep, transitive dep, base image, or bundled binary
4. Research what version fixes it and whether an upstream fix even exists
5. Make the change, build a test image, push it to DockerHub
6. Manually trigger the Harness OnDemand scanner
7. Interpret the scan results
8. Write a PR with all of that context documented

This process takes **2–4 hours per CVE** on a good day, and is error-prone — it's easy to miss that a vulnerability comes from an embedded binary rather than a direct dependency, or to forget to document scan evidence in the PR.

The automated workflow does all of this in **under 15 minutes**, with better documentation than we typically write manually.

---

## Is This a Plugin, an Agent, or a Skill?

It is technically a **Claude Code Plugin** — specifically, it bundles two components:

| Component | Type | Role |
|-----------|------|------|
| `vuln-remediator` | **Agent** | The autonomous AI that executes the 9-step workflow |
| `/vuln-remediation:remediate` | **Command (Skill)** | The slash command entry point you invoke |

**How these concepts relate:**

- A **Plugin** is an installable package (like a VS Code extension) that bundles agents, commands, and skills together. This one lives in our internal Claude marketplace.
- An **Agent** is an autonomous subprocess that can plan, call tools (bash, GitHub API, Harness API, file edits), and make multi-step decisions. The `vuln-remediator` agent is what does the actual work.
- A **Skill/Command** is the user-facing trigger (`/vuln-remediation:remediate CI-1234`) that invokes the agent with the right context.

Think of it like this: the **plugin** is the package you install, the **command** is the button you press, and the **agent** is the autonomous worker that runs the job.

---

## How It Works (The 9-Step Workflow)

```
Input (JIRA ticket or raw CVE details)
    |
    v
Step 1: Parse input — extract image, CVE, package, required fix version
    |
    v
Step 2: Find source repo — map image name to GitHub org/repo, clone it
    |
    v
Step 3: Analyze vulnerability sources
        |-- Direct Go/npm/pip dependencies (go.mod, package.json)
        |-- Transitive dependencies (go mod graph)
        |-- Base image (FROM line in Dockerfile)
        `-- Bundled binaries (wget/curl downloads in Dockerfile)
    |
    v
Step 4: Research available fixes
        |-- Query DockerHub for latest base image tags
        |-- Query GitHub Releases for latest binary versions
        `-- Query Go module proxy for latest package versions
    |
    v
Step 5: Make targeted code changes (Dockerfile, go.mod, go.sum)
    |
    v
Step 6: Build & push test image to DockerHub
        (always uses -test suffix, never touches prod tags)
    |
    v
Step 7: Trigger Harness OnDemand vulnerability scan pipeline,
        poll for completion (~3-5 minutes)
    |
    v
Step 8: Fetch scan results via Harness STO API
        |-- Before/after vulnerability counts by severity
        |-- CVE-specific status: Resolved / Partial / Blocked
        `-- Any new issues introduced by the upgrade
    |
    v
Step 9: PAUSE — Present full report and await human approval
        `-- On approval: create branch + open PR with scan evidence embedded
```

> **Human approval is required before any irreversible action.** The agent will never push to a production image tag, force-push, or create a PR without you explicitly saying "yes, proceed."

---

## Example Usage

### Example 1 — Via JIRA Ticket

```
/vuln-remediation:remediate CI-21050
```

The agent fetches [CI-21050](https://harness.atlassian.net/browse/CI-21050), extracts the image (`plugins/buildx:1.3.13`), CVE (`CVE-2026-24051`), and vulnerable package (`go.opentelemetry.io/otel/sdk@v1.31.0`), then runs the full 9-step workflow automatically.

### Example 2 — Via Raw Details

```
/vuln-remediation:remediate plugins/buildx:1.3.13 CVE-2026-24051 go.opentelemetry.io/otel/sdk@v1.31.0
```

Useful when you have a CVE report but no JIRA ticket yet, or want to quickly remediate without creating a ticket first.

### What the Agent Produced (CI-21050)

**Scan evidence (step 7–8):**

| | |
|---|---|
| **Test image pushed** | `vinayakharness/buildx-test:linux-amd64` |
| **Harness OnDemand scan** | [Execution `0sihybi9TmKn63WWDi7mBg`](https://harness0.harness.io/ng/account/l7B_kbSEQD2wjrM7PShm5w/all/orgs/Security_and_Compliance/projects/ProdSec/pipelines/Ondemand_Vulnerability_Scanner/executions/0sihybi9TmKn63WWDi7mBg/pipeline) |
| **Scan ID** | `jJYUVc5H6Ept1b8nrxUA4Q` |

**Changes made (step 5):**

| File | Change |
|------|--------|
| `Dockerfile.linux.amd64` | Base image `docker:28.1.1-dind` → `29.2.1-dind`; buildx `v0.23.0` → `v0.31.1` |
| `Dockerfile.linux.arm64` | Same upgrades |

**CVE outcome:**

| CVE | Package | Before | After | Status |
|-----|---------|--------|-------|--------|
| CVE-2026-24051 | `go.opentelemetry.io/otel/sdk` | v1.31.0 | v1.38.0 | Partial — full fix (v1.40.0) blocked upstream |

**Output PR:** [drone-plugins/drone-buildx#91](https://github.com/drone-plugins/drone-buildx/pull/91)

The PR description was auto-generated with the full scan report, upgrade rationale, upstream blocker note, and links to the scan pipeline — no manual write-up needed.

---

## How to Use It

### Prerequisites

Set these environment variables before starting Claude Code:

```bash
export GITHUB_TOKEN=<github-pat>          # github.com -> Settings -> Developer Settings -> PATs
export HARNESS_TOKEN=<harness-pat>        # Harness UI -> Profile -> API Keys -> New Token
export DOCKERHUB_USER=<username>          # hub.docker.com -> Account Settings -> Security
export DOCKERHUB_TOKEN=<token>
export HARNESS_ACCOUNT_ID=<id>            # From your Harness URL: /account/<id>/...
export ONDEMAND_PIPELINE_ID=<id>          # Harness -> Pipelines -> OnDemand scanner -> URL

# Optional: if your JIRA is separate from Harness
export JIRA_EMAIL=<your-email>
export JIRA_TOKEN=<jira-api-token>
```

### Invocation

```bash
# By JIRA ticket
/vuln-remediation:remediate CI-21050

# By raw details
/vuln-remediation:remediate plugins/buildx:1.3.13 CVE-2026-24051 go.opentelemetry.io/otel/sdk@v1.31.0
```

---

## How We Can Extend This

The plugin is designed to be a starting point, not a finished product. Here is a roadmap of what would make it significantly more powerful:

### 1. Jira Integration — Auto-comment on Tickets

After a PR is opened, automatically post a comment back to the JIRA ticket with the PR link, scan summary, and CVE status. This closes the loop without any manual tracking.

```
CI-21050 <- "PR #91 opened. CVE partially remediated (v1.31.0 -> v1.38.0).
             Full fix blocked upstream. Scan: [link]"
```

### 2. Multi-Repo Support

Right now the agent has a hardcoded mapping of image names to repos (e.g., `plugins/buildx` → `drone-plugins/drone-buildx`). Extending this to a **configurable registry** — a YAML file or a Harness-internal API lookup — would let it handle any of our repos automatically: `harness/ci-addon`, `harness/delegate`, internal services, etc.

### 3. Batch / Multiple CVE Remediation

Today it handles one CVE at a time. We could extend it to:

- Accept a list of CVEs for the same image and fix them in a single PR
- Process a batch of JIRA tickets in parallel (one agent per ticket)
- Produce a weekly digest: "Here are all open CVEs — which ones have fixes available?"

### 4. Smarter Upstream Blocking Detection

When a fix is not yet available (as with `otel/sdk@v1.40.0` today — no `buildx` release ships it yet), the agent could:

- Watch the upstream GitHub repo for a new release
- Re-trigger remediation automatically when the fix lands
- Update the JIRA ticket proactively

### 5. Severity-Based Triage and Prioritization

Connect the agent to our full CVE backlog and have it prioritize automatically: fix Criticals first, batch Mediums together, defer Lows. It could generate a weekly "remediation plan" ticket.

### 6. Slack Notifications

Post to a `#security-cve-remediations` channel when a PR is opened, with the scan results and PR link — so the security team has full visibility without monitoring JIRA manually.

---

## Where the Code Lives

The plugin source is in our internal Claude marketplace repo:

```
Dev/vuln-remediation-plugin/plugins/vuln-remediation/
├── agents/vuln-remediator.md   # Full agent definition and 9-step workflow
├── commands/remediate.md       # Slash command entry point
└── README.md                   # Setup and usage guide
```

Happy to walk anyone through it or pair on extending it. Feedback welcome.
