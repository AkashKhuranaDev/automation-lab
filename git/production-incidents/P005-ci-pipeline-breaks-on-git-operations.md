# P005 — CI Pipeline Breaks on Git Operations

> **Severity:** P2 | **Expected Recovery Time:** 30–60 minutes
>
> **Related:** [`hooks/`](../hooks/) | [`enterprise-workflows/`](../enterprise-workflows/) | [`performance/`](../performance/)

---

## Symptoms

- CI pipeline fails at the git clone or checkout step
- Error messages contain: `fatal: repository not found`, `error: RPC failed`, `SSL certificate problem`, `fatal: unsafe directory`, `Authentication failed`
- Pipeline hangs indefinitely during `git fetch` or `git clone`
- Shallow clone fails with `fatal: --unshallow on a complete repository does not make sense`
- `git lfs` errors during checkout in CI
- CI passes locally but fails in the CI environment

---

## Immediate Actions (Do This First)

```bash
# Step 1: Capture the full error message
# Read the CI log from the beginning — do not skip to the end
# The first error line is usually the root cause

# Step 2: Check if the issue is local or remote
git clone --depth=1 git@github.com:$ORG/$REPO.git /tmp/local-test
# If this succeeds: the issue is environment-specific (runner config, CI secrets, UID)
# If this fails: the issue is network, auth, or repository state

# Step 3: Match the error message to the diagnosis table below
```

---

## Diagnosis

Match your exact error message to the root cause:

| Error Message | Root Cause | Recovery Section |
|---|---|---|
| `fatal: repository not found` | Wrong remote URL or SSH key not configured | [Authentication Issues](#authentication-issues) |
| `Authentication failed for 'https://...'` | Expired PAT or wrong credentials | [Authentication Issues](#authentication-issues) |
| `error: RPC failed; curl 28 Operation timed out` | Network timeout or repository too large | [Network and Timeout Issues](#network-and-timeout-issues) |
| `fatal: unsafe repository ('.../repo' is owned by someone else)` | UID mismatch in container (Git 2.35.2+ security change) | [Unsafe Directory](#unsafe-directory-error) |
| `Error downloading object: ... not found` | LFS object missing from remote store | [LFS Issues](#git-lfs-issues) |
| `fatal: --unshallow on a complete repository` | Checkout configuration conflict | [Shallow Clone Issues](#shallow-clone-issues) |
| Pipeline hangs at git step for > 2 minutes with no output | SSH host key verification prompt waiting for input | [SSH Host Verification Hang](#ssh-host-verification-hang) |

---

## Recovery

---

### Authentication Issues

```bash
# Diagnose: Test SSH connectivity from the CI runner
ssh -T git@github.com
# Expected: Hi username! You've successfully authenticated...
# If: Permission denied (publickey) → SSH key not loaded or wrong key

# Fix: Verify the SSH key is loaded in CI
eval "$(ssh-agent -s)"
ssh-add /path/to/deploy-key
ssh -T git@github.com

# For GitHub Actions — use the SSH key secret correctly
# .github/workflows/ci.yml:
- name: Configure SSH
  run: |
    mkdir -p ~/.ssh
    echo "${{ secrets.DEPLOY_KEY }}" > ~/.ssh/id_ed25519
    chmod 600 ~/.ssh/id_ed25519
    ssh-keyscan github.com >> ~/.ssh/known_hosts

# Diagnose: HTTPS PAT expired
git ls-remote https://github.com/$ORG/$REPO.git
# If: 401 or "Authentication failed" → PAT is expired or has wrong scopes

# Fix: Rotate the PAT and update the CI secret
# GitHub → Developer settings → Personal access tokens → Generate new token
# Scopes needed: repo (for private repos) or public_repo (for public)
# Update in: GitHub Actions secrets, Jenkins credentials, etc.
```

---

### Network and Timeout Issues

```bash
# Diagnose: Is this a size/timeout issue?
git clone --depth=1 git@github.com:$ORG/$REPO.git /tmp/test-clone
# If this times out: repository is too large for current network conditions

# Fix option 1: Reduce what is cloned
# In GitHub Actions:
- uses: actions/checkout@v4
  with:
    fetch-depth: 1          # Shallow clone — only latest commit
    lfs: false              # Skip LFS objects initially

# Fix option 2: Increase timeouts (Git HTTP transport)
git config --global http.postBuffer 524288000  # 500 MB buffer
git config --global http.lowSpeedLimit 1000
git config --global http.lowSpeedTime 60

# Fix option 3: Use a repository cache on the CI runner
# (Clone once, fetch updates rather than fresh clone each time)
# GitHub Actions: actions/cache for .git directory
- name: Cache git repository
  uses: actions/cache@v4
  with:
    path: .git
    key: git-${{ github.repository }}-${{ github.sha }}
    restore-keys: git-${{ github.repository }}-
```

---

### Unsafe Directory Error

Introduced in Git 2.35.2 as a security fix. Common in containerized CI where the runner UID does not match the directory owner UID.

```bash
# Error:
# fatal: detected dubious ownership in repository at '/home/runner/work/repo'
# To add an exception for this directory, call:
#   git config --global --add safe.directory /home/runner/work/repo

# Diagnose:
ls -la /home/runner/work/
# If the repo directory is owned by a different UID than the runner process, this triggers

# Fix: Add the safe.directory exception in CI
# In GitHub Actions (add before any git operations):
- name: Configure git safe directory
  run: git config --global --add safe.directory "$GITHUB_WORKSPACE"

# Fix: Set in the Docker image used by CI
# Dockerfile:
RUN git config --global --add safe.directory /workspace

# Fix: In Jenkins pipeline (Groovy)
sh 'git config --global --add safe.directory $(pwd)'
```

---

### Git LFS Issues

```bash
# Error: "Error downloading object: file.bin (sha256): not found"

# Diagnose: Check if LFS is installed in the CI environment
git lfs version
# If command not found: install git-lfs

# Diagnose: Check if LFS objects exist in the store
git lfs ls-files
git lfs status

# Fix: Install LFS in CI
# Ubuntu:
apt-get install -y git-lfs
git lfs install

# GitHub Actions — actions/checkout handles LFS if configured:
- uses: actions/checkout@v4
  with:
    lfs: true

# Fix: If LFS objects are genuinely missing from the remote
# (Check in GitHub → LFS objects tab, or via API)
curl -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/$ORG/$REPO/git/large_files" | jq '.'

# If objects are gone: restore from backup or accept data loss for that object
```

---

### Shallow Clone Issues

```bash
# Error: "fatal: --unshallow on a complete repository does not make sense"
# This happens when a step tries to deepen a clone that was not shallow

# Diagnose:
git rev-parse --is-shallow-repository
# true → shallow   false → not shallow

# Fix: Be explicit about fetch-depth in checkout
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Full history — no --unshallow needed later

# Fix: If a subsequent step needs full history but started with shallow:
git fetch --unshallow
# Only works on shallow clones — check first:
if git rev-parse --is-shallow-repository | grep -q true; then
  git fetch --unshallow
fi
```

---

### SSH Host Verification Hang

```bash
# Symptom: Pipeline hangs at git clone with no output for > 2 minutes
# Cause: SSH is waiting for interactive confirmation of github.com host key

# Diagnose: Run the clone manually in the CI environment
# If it prompts: "Are you sure you want to continue connecting (yes/no/[fingerprint])?"
# This is the hang

# Fix: Pre-populate known_hosts
ssh-keyscan github.com >> ~/.ssh/known_hosts
# or:
ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts

# Fix: Add to SSH config
echo "Host github.com
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null" >> ~/.ssh/config
# Note: Disabling StrictHostKeyChecking reduces security — use known_hosts instead

# GitHub Actions: Use actions/checkout which handles this automatically
```

---

## Verification

```bash
# Verify the specific CI step that was failing now succeeds
# Run the failing step in isolation

# For authentication issues:
ssh -T git@github.com  # Should succeed

# For clone issues:
time git clone --depth=1 git@github.com:$ORG/$REPO.git /tmp/verify-clone
# Should complete in < 3 minutes

# For LFS issues:
cd /tmp/verify-clone
git lfs pull  # Should succeed

# Trigger a new CI run and confirm it passes
```

---

## Prevention

- Store CI SSH deploy keys in a secret manager (AWS Secrets Manager, GitHub Secrets) with automatic rotation reminders
- Add the `ssh-keyscan github.com >> ~/.ssh/known_hosts` step before any SSH git operations in CI
- Add `git config --global --add safe.directory $CI_WORKSPACE` to your CI base image — do not add it per-job
- Monitor CI clone times — alert if they exceed a threshold (2 minutes for most repos, 5 for large repos)
- Use `actions/checkout@v4` in GitHub Actions — it handles LFS, safe.directory, and known_hosts correctly with minimal configuration
