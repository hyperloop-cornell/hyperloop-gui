# SECTION 4 — Daily Git Workflow

## Purpose
Standardize contribution practices across the team and ensure smooth integration with the automated testing and deployment pipeline.

## Overview
Your project uses a **feature branch → Pull Request → main** workflow with GitHub Actions CI/CD. Tests run automatically on PRs, and merges to main trigger automatic deployment to EC2.

---

## Step 1 — Create a Feature Branch

Use a descriptive branch name that indicates the work being done:

```bash
git checkout -b feature/<your-feature>
# or
git checkout -b fix/<your-fix>
# or
git checkout -b refactor/<your-refactor>
```

### Examples
```bash
git checkout -b feature/websocket-reconnect-logic
git checkout -b fix/telemetry-parsing-bug
git checkout -b feature/health-metrics-dashboard
git checkout -b refactor/cloud-api-auth
```

**Naming convention**: 
- Use lowercase
- Separate words with hyphens
- Prefix with `feature/`, `fix/`, or `refactor/` to indicate the type of work
- Keep names concise and descriptive

---

## Step 2 — Make Changes in VS Code

1. Edit files in your branch
2. Test locally:
   ```bash
   # For RPi Hub Server
   cd rpi-hub-server && pytest tests/ -v
   
   # For Cloud Services
   cd cloud-services && pytest tests/ -v
   
   # For Web Client
   cd web-client && npm run type-check && npm run lint
   ```

3. Run the full build to catch any issues:
   ```bash
   # Web client
   cd web-client && npm run build
   ```

---

## Step 3 — Stage and Commit

Make focused commits with clear messages:

```bash
# Stage all changes
git add .

# Commit with a descriptive message
git commit -m "feat: add websocket reconnect logic

- Implement exponential backoff for reconnection attempts
- Add heartbeat monitoring to detect dead connections
- Respect max reconnect attempts configuration"
```

### Commit Message Format
Use the [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>: <subject>

<body (optional)>
```

**Types**:
- `feat:` — New feature
- `fix:` — Bug fix
- `refactor:` — Code refactoring (no feature or bug fix)
- `docs:` — Documentation only
- `test:` — Test additions or changes
- `chore:` — Build process, dependencies, tooling

**Example commits**:
```bash
git commit -m "feat: add network latency metric to health dashboard"
git commit -m "fix: resolve base64 decoding error in telemetry parser"
git commit -m "test: add unit tests for hub agent reconnection"
git commit -m "refactor: simplify buffer manager message eviction logic"
```

---

## Step 4 — Push to GitHub

Push your branch to the remote repository:

```bash
# First time pushing the branch
git push -u origin feature/websocket-reconnect-logic

# Subsequent pushes
git push
```

---

## Step 5 — Open a Pull Request

1. Go to your repository on GitHub
2. Click **Compare & Pull Request** (or go to the Pull Requests tab and create a new one)
3. **Fill out the PR details**:
   - Clear title: `feat: add health metrics dashboard`
   - Description:
     ```markdown
     ## What does this PR do?
     Adds a new health metrics display showing network latency, CPU usage, and uptime.
     
     ## Why?
     Provides visibility into system health for debugging and monitoring.
     
     ## How to test?
     1. Start hub server
     2. Start cloud API backend
     3. Start web client
     4. Navigate to /health-dashboard
     5. Verify metrics display and update every 30 seconds
     
     ## Screenshots
     [optional: add screenshots if UI changes]
     ```
4. **Assign reviewers** (required before merging)
5. **Link any related issues** (if applicable): `Closes #42`

---

## Step 6 — CI/CD Pipeline Runs Automatically

When you push your branch or create a PR, GitHub Actions automatically runs:

### For Cloud Services
- Install Python dependencies
- Run pytest tests: `pytest tests/ -v`
- Upload test results as artifacts

### For Web Client
- Install Node dependencies
- TypeScript type checking: `npm run type-check`
- ESLint linting: `npm run lint`
- Build: `npm run build`

**Check the results**:
- Green checkmarks = all tests passed ✅
- Red X = tests failed ❌

If tests fail:
1. Click **Details** to see the error
2. Fix the issue locally
3. Commit and push again — CI will re-run automatically

---

## Step 7 — Address Review Feedback

1. Reviewers may request changes
2. Make updates locally and commit:
   ```bash
   git add .
   git commit -m "refactor: address review feedback on error handling"
   ```
3. Push the new commits:
   ```bash
   git push
   ```
4. The PR updates automatically with your new commits

---

## Step 8 — Merge and Deploy

Once tests pass and a reviewer approves:

1. **Click "Merge pull request"** on GitHub
2. Select **"Squash and merge"** (recommended) or **"Create a merge commit"**
3. Delete the feature branch (GitHub offers this after merge)

**Automatic deployment happens next**:
- The `deploy.yml` workflow triggers
- Code is deployed to EC2 for cloud-services and web-client
- You can monitor the deployment in the **Actions** tab

---

## Step 9 — Verify Your Changes in Production

After deployment completes:

1. Check the **Actions** tab to confirm deployment succeeded
2. Visit your live instance to verify changes work correctly
3. Monitor logs if needed (ask for access to EC2)

---

## Quick Reference: Full Workflow from Start to Finish

```bash
# 1. Create and switch to feature branch
git checkout -b feature/my-new-feature

# 2. Make changes, commit locally
git add .
git commit -m "feat: add new feature

- Implementation details
- Why this change helps"

# 3. Push to GitHub
git push -u origin feature/my-new-feature

# 4. Open PR on GitHub (or click the link that appears in terminal)

# 5. If tests fail, fix and push again
git add .
git commit -m "fix: address test failures"
git push

# 6. Once approved and tests pass, merge on GitHub

# 7. Delete local branch (optional cleanup)
git checkout main
git fetch --prune
```

---

## Troubleshooting

### "Your branch is ahead of 'origin/main' by 5 commits"
This is normal and expected. Your feature branch is ahead of main, which is why you're opening a PR.

### "Tests failed on GitHub but pass locally"
GitHub might be using different versions or configurations. Check:
- Python version: `python --version` (should match workflow)
- Node version: `node --version` (should match workflow)
- Dependencies: `pip list` or `npm list`

### "I need to update my branch with latest main changes"
```bash
git fetch origin
git rebase origin/main
git push -f  # Force push because rebase rewrites history
```

### "I accidentally committed to main instead of a branch"
```bash
# Create a new branch from current state
git branch feature/my-mistake

# Reset main to before your commits
git reset --hard origin/main

# Switch to your new branch with the commits
git checkout feature/my-mistake
```

---

## Important Reminders

- ✅ **Always test locally before pushing** 
- ✅ **Write clear commit messages** — future you will thank current you
- ✅ **Keep PRs focused** — easier to review and faster to merge
- ✅ **Review others' PRs** — knowledge sharing and code quality
- ❌ **Don't commit secrets** — use `.env` files and `.gitignore`
- ❌ **Don't force-push to main** — main is protected anyway
- ❌ **Don't skip the CI tests** — they catch real bugs

---

## Git Workflow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Local Development                                           │
│  git checkout -b feature/xyz                               │
│  [Make changes] → git add . → git commit -m "..."          │
│  git push -u origin feature/xyz                            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ GitHub: Pull Request Created                               │
│  - Assign reviewers                                        │
│  - Add description                                         │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ GitHub Actions: CI/CD Pipeline                             │
│  ✓ Cloud Services Tests (pytest)                           │
│  ✓ Web Client Tests (type-check, lint, build)             │
│  ✓ RPi Hub Server Tests (if applicable)                   │
└─────────────────────────────────────────────────────────────┘
                          ↓
                  [Tests Pass? → Yes]
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ Code Review                                                 │
│  - Reviewer approves or requests changes                   │
│  - If changes needed: fix → push → re-run CI              │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ Merge to Main                                               │
│  - Click "Merge pull request" on GitHub                    │
│  - Triggers automatic deployment workflow                  │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ Automatic Deployment to EC2                                │
│  - Cloud Services API updated                              │
│  - Web Client updated                                      │
│  - Check Actions tab for deployment status                │
└─────────────────────────────────────────────────────────────┘
```
