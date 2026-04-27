# Manual Approval Gate — Production Deployment

## Overview

A manual approval gate requires a designated reviewer to explicitly approve the deployment before it proceeds to production. Without approval, the pipeline halts indefinitely.

The gate sits between **Push to Registry** and **Update Helm Chart** — only for **production** deployments. Dev deployments continue automatically.

This is implemented as a **caller-side job** in each service repository's CI workflow (e.g., `ci-advanced.yml`). The reusable `_update_helm_chart.yml` template supports an `approved-by` input for audit trail.

---

## Actual Pipeline Flow (ci-advanced.yml)

Based on the real caller workflow used in repositories like `ChapterOne-Frontend`:

```
Push to dev/production
        │
        ▼
┌───────────────┐
│  Prepare      │  Resolve tag (semver for prod, dev-SHA for dev)
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  Test         │  Run unit tests
└───────┬───────┘
        │
   ┌────┴────┐
   ▼         ▼
┌──────┐ ┌──────┐
│Sonar │ │Snyk  │  (parallel security scans)
└──┬───┘ └──┬───┘
   └────┬────┘
        ▼
┌───────────────┐
│ Build & Trivy │  Build Docker + Trivy scan
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  Push         │  Push image to Docker Hub
└───────┬───────┘
        │
        ▼
┌───────────────────────────────────┐
│  Approval Gate (production only)   │  ⛔ WAITS FOR HUMAN
│  Skipped for dev deployments       │
└───────┬───────────────────────────┘
        │
        ▼
┌───────────────┐
│ Update Helm   │  Update Helm chart → triggers GitOps deploy
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  Notify       │  Send email notification
└───────────────┘
```

### Reusable Templates Used

| Job | Reusable Template | Purpose |
|-----|-------------------|---------|
| sonar | `_sonar.yml` | SonarQube code quality scan |
| snyk | `_snyk.yml` | Snyk dependency vulnerability scan |
| build_trivy | `_build_trivy.yml` | Docker build + Trivy container scan |
| push | `_push.yml` | Push image to Docker Hub |
| update-helm | `_update_helm_chart.yml` | Update Helm chart values |
| notify | `_notify.yml` | Email notification |

---

## Where Is the Gate Added?

**Between `push` and `update-helm` jobs** in the caller workflow (e.g., `ci-advanced.yml` in each service repo).

### Why This Location?

| Stage | Action | Risk | Reversible? |
|-------|--------|------|-------------|
| Build & Scan | Build image + scan | Low | Yes |
| Push to Registry | Publish image | Medium | Mostly |
| **Update Helm Chart** | **Commit values → GitOps deploys** | **High** | **No** |

The gate sits at the **CI/CD boundary**:
- **CI** = Build + Push → automatic, fast feedback
- **CD** = Helm update → production goes live, needs human judgment

Once Helm values are pushed, ArgoCD/Flux auto-deploys. This is the **point of no return**.

### Why NOT Before Build/Push?
- Blocking build/push delays feedback — devs need immediate scan results
- Build/push are preparation, not deployment — no users affected

### Why NOT for Dev?
- Dev = fast iteration, low blast radius
- Production = real customers, high blast radius

---

## What Does the Gate Intend to Do?

It enforces **separation of concerns**:
1. **Automation** — pipeline builds, scans, publishes automatically
2. **Human judgment** — reviewer decides if artifact is safe for production

### What It Prevents
- Broken builds reaching production
- Vulnerable images (CRITICAL/HIGH CVEs) being deployed
- Unintended `production` branch pushes triggering deployment
- Compromised pipelines auto-deploying malicious code

---

## What Should the Reviewer Check?

### 1. Security Scan Results
- **Trivy**: No CRITICAL/HIGH container vulnerabilities
- **Snyk**: No high-severity dependency issues
- **SonarQube**: Quality gate passed

### 2. Image Tag Verification
- Tag (commit SHA or version) matches intended release
- Tag exists in Docker registry

### 3. Branch Verification
- Deploying from `production` branch (not accidental)
- Commit is the expected one

### 4. Change Review
- Diff contains expected changes only
- No unexpected files, config, or dependency updates

### 5. Deployment Readiness
- Database migrations prepared
- Rollback plan exists
- Monitoring/alerting in place

### Quick Checklist
```
☐ Trivy: No CRITICAL/HIGH vulnerabilities
☐ Snyk: No high-severity dependency issues
☐ SonarQube: Quality gate passed
☐ Image tag: Correct version/commit
☐ Branch: Deploying from production
☐ Changes: Reviewed and expected
☐ Rollback: Plan exists
```

---

## Technical Implementation

### GitHub Environment Protection Rules

1. A GitHub Environment named `production` is configured in repo settings
2. Environment set to **"Required reviewers"** with designated approvers
3. The `approve-production` job references `environment: production`
4. GitHub **pauses the workflow** and sends approval requests
5. Reviewer clicks **Approve** or **Reject** in the Actions UI
6. After approval, downstream jobs proceed

### Approval Gate Job (in caller workflow)

```yaml
  # ─────────────────────────────────────────────
  # APPROVAL GATE — Production deployment approval
  # ─────────────────────────────────────────────
  approve-production:
    name: "Approval Gate: Production Deployment"
    needs: [prepare, push]
    if: needs.prepare.outputs.is_production == 'true'
    runs-on: ubuntu-latest
    environment: production
    outputs:
      approved-by: ${{ github.actor }}
    steps:
      - name: Await Approval
        run: |
          echo "✅ Production deployment approved by reviewer."
          echo ""
          echo "## 🔒 Production Approval Gate" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Property | Value |" >> $GITHUB_STEP_SUMMARY
          echo "|----------|-------|" >> $GITHUB_STEP_SUMMARY
          echo "| **Service** | ${{ needs.prepare.outputs.image_name }} |" >> $GITHUB_STEP_SUMMARY
          echo "| **Image Tag** | ${{ needs.prepare.outputs.tag }} |" >> $GITHUB_STEP_SUMMARY
          echo "| **Environment** | production |" >> $GITHUB_STEP_SUMMARY
          echo "| **Approved By** | ${{ github.actor }} |" >> $GITHUB_STEP_SUMMARY
          echo "| **Approved At** | $(date -u) |" >> $GITHUB_STEP_SUMMARY
```

### Updated update-helm Job (in caller workflow)

```yaml
  # ─────────────────────────────────────────────
  # UPDATE HELM — Update Helm chart with new image tag
  # ─────────────────────────────────────────────
  update-helm:
    name: Update Helm Chart
    needs: [prepare, push, approve-production]
    if: >-
      always() &&
      needs.prepare.outputs.push == 'true' &&
      needs.push.result == 'success' &&
      (needs.approve-production.result == 'success' ||
       needs.approve-production.result == 'skipped')
    uses: Googleeyy/ChapterOne-Reusable-Templates/.github/workflows/_update_helm_chart.yml@main
    with:
      service-name: frontend
      image-tag: ${{ needs.prepare.outputs.tag }}
      environment: ${{ needs.prepare.outputs.is_production == 'true' && 'production' || 'dev' }}
      approved-by: ${{ needs.approve-production.outputs.approved-by || '' }}
    secrets:
      HELM_REPO_PAT: ${{ secrets.HELM_REPO_PAT }}
```

### Key Conditional Logic

| Condition | Meaning |
|-----------|---------|
| `needs.approve-production.result == 'success'` | Production branch — reviewer approved |
| `needs.approve-production.result == 'skipped'` | Dev branch — gate was skipped (auto-deploy) |
| `needs.push.result == 'success'` | Image was pushed successfully |
| `needs.prepare.outputs.push == 'true'` | Push was intended for this run |

### Reusable Template Enhancement (`_update_helm_chart.yml`)

The reusable `_update_helm_chart.yml` now accepts an optional `approved-by` input:

```yaml
  approved-by:
    required: false
    type: string
    default: ""
    description: "GitHub username of the reviewer who approved the production deployment"
```

When provided:
- The **Git commit message** includes `(approved by @username)` for audit trail
- The **Job Summary** includes an **Approved By** row

This creates a permanent record of who authorized each production deployment.

---

## Setup Instructions

### Step 1: Create the `production` Environment

1. Go to **Settings → Environments**
2. Click **New environment** → name it `production`
3. Click **Configure environment**

### Step 2: Add Required Reviewers

1. Under **Protection rules**, check **Required reviewers**
2. Add GitHub usernames of approvers (1–6 people)
3. Click **Save protection rules**

### Step 3: (Optional) Add Wait Timer

Set a wait timer (e.g., 5 min) before reviewers can approve — buffer for last-minute checks.

### Step 4: (Optional) Restrict Branches

Under **Deployment branches**, select only `production`.

---

## How to Approve/Reject a Deployment

1. Go to **Actions** tab in your repository
2. Click the running workflow
3. Find the **"Approval Gate: Production Deployment"** job
4. Click **Review deployments**
5. Click **Approve** or **Reject**
6. Add an optional comment explaining your decision

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Gate not triggering | Environment not configured | Create `production` environment in Settings |
| No approval request sent | No reviewers configured | Add reviewers to environment protection rules |
| Dev deployment blocked | `if` condition wrong | Verify `if: github.ref == 'refs/heads/production'` |
| Update-helm skipped after approval | Conditional logic error | Check `needs.approve-production.result` in `if` |
| Timeout waiting for approval | No reviewer available | Add more reviewers or use wait timer |

---

## Exact Changes Needed in Caller Workflow (ci-advanced.yml)

### Step 1: Add the `approve-production` job

Insert this job **after** the `push` job and **before** the `update-helm` job:

```yaml
  # ─────────────────────────────────────────────
  # APPROVAL GATE — Production deployment approval
  # ─────────────────────────────────────────────
  approve-production:
    name: "Approval Gate: Production Deployment"
    needs: [prepare, push]
    if: needs.prepare.outputs.is_production == 'true'
    runs-on: ubuntu-latest
    environment: production
    outputs:
      approved-by: ${{ github.actor }}
    steps:
      - name: Await Approval
        run: |
          echo "✅ Production deployment approved by reviewer."
          echo ""
          echo "## 🔒 Production Approval Gate" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Property | Value |" >> $GITHUB_STEP_SUMMARY
          echo "|----------|-------|" >> $GITHUB_STEP_SUMMARY
          echo "| **Service** | ${{ needs.prepare.outputs.image_name }} |" >> $GITHUB_STEP_SUMMARY
          echo "| **Image Tag** | ${{ needs.prepare.outputs.tag }} |" >> $GITHUB_STEP_SUMMARY
          echo "| **Environment** | production |" >> $GITHUB_STEP_SUMMARY
          echo "| **Approved By** | ${{ github.actor }} |" >> $GITHUB_STEP_SUMMARY
          echo "| **Approved At** | $(date -u) |" >> $GITHUB_STEP_SUMMARY
```

### Step 2: Update the `update-helm` job

Change from:
```yaml
  update-helm:
    name: Update Helm Chart
    needs: [prepare, push]
    if: needs.prepare.outputs.push == 'true'
    uses: Googleeyy/ChapterOne-Reusable-Templates/.github/workflows/_update_helm_chart.yml@main
    with:
      service-name: frontend
      image-tag: ${{ needs.prepare.outputs.tag }}
      environment: ${{ needs.prepare.outputs.is_production == 'true' && 'production' || 'dev' }}
    secrets:
      HELM_REPO_PAT: ${{ secrets.HELM_REPO_PAT }}
```

To:
```yaml
  update-helm:
    name: Update Helm Chart
    needs: [prepare, push, approve-production]
    if: >-
      always() &&
      needs.prepare.outputs.push == 'true' &&
      needs.push.result == 'success' &&
      (needs.approve-production.result == 'success' ||
       needs.approve-production.result == 'skipped')
    uses: Googleeyy/ChapterOne-Reusable-Templates/.github/workflows/_update_helm_chart.yml@main
    with:
      service-name: frontend
      image-tag: ${{ needs.prepare.outputs.tag }}
      environment: ${{ needs.prepare.outputs.is_production == 'true' && 'production' || 'dev' }}
      approved-by: ${{ needs.approve-production.outputs.approved-by || '' }}
    secrets:
      HELM_REPO_PAT: ${{ secrets.HELM_REPO_PAT }}
```

### Step 3: Update the `notify` job

Add `approve-production` to the `needs` list:

```yaml
  notify:
    name: Email Notification
    needs: [prepare, sonar, snyk, build_trivy, push, approve-production, update-helm]
    if: always()
```

### Step 4: Create the `production` Environment in GitHub

1. Go to the service repo → **Settings → Environments**
2. Click **New environment** → name it `production`
3. Under **Protection rules**, check **Required reviewers**
4. Add GitHub usernames of approvers (1–6 people)
5. (Optional) Set a **Wait timer** (e.g., 5 minutes)
6. (Optional) Restrict **Deployment branches** to `production` only
7. Click **Save protection rules**

> ⚠️ Without this step, the `environment: production` in the workflow will pass through without requiring approval.

---

## Summary

| Aspect | Detail |
|--------|--------|
| **Location** | Between `push` and `update-helm` in caller workflow |
| **Scope** | Production deployments only (`is_production == 'true'`) |
| **Mechanism** | GitHub Environment protection rules |
| **Reusable template change** | `_update_helm_chart.yml` now accepts `approved-by` input for audit trail |
| **Reviewer checks** | Security scans, tag, branch, changes, readiness |
| **Dev behavior** | Gate skipped (`result: skipped`), auto-deploy continues |
| **Production behavior** | Gate pauses pipeline until reviewer approves |
| **Audit trail** | Approver username in Git commit message + Job Summary |
| **Caller repos** | Each service repo must create `production` environment with reviewers |
