# Decision Guide: SSH vs HTTPS

Use this guide when configuring repository access for developers, CI/CD systems, or automated tooling.

---

## Quick Decision

```
New developer workstation or personal machine?
└─> Use SSH

CI/CD pipeline (GitHub Actions, Jenkins, CircleCI)?
└─> Use HTTPS with a token or Deploy Key (SSH) depending on scope
    - Single repo: Deploy Key (SSH)
    - Multiple repos: Fine-grained PAT (HTTPS)

Corporate proxy or firewall blocking port 22?
└─> Use HTTPS

Self-hosted Git server with strict access control?
└─> Use SSH with per-user keys for auditability
```

---

## Comparison

| Criteria | SSH | HTTPS |
|----------|-----|-------|
| Authentication mechanism | Public/private key pair | Username + token (no passwords) |
| Initial setup effort | Higher (key generation, agent, known_hosts) | Lower (only token needed) |
| Credential storage | SSH agent or keychain | Credential helper (keychain, env var) |
| Token rotation | Key revocation on GitHub | Token revocation on GitHub |
| Corporate proxy support | Blocked on port 22 at many enterprises | Works on port 443 (always open) |
| CI/CD | Deploy keys (repo-scoped) | Fine-grained PATs or OIDC |
| Auditability | Per-key access logs | Per-token access logs |
| `git clone` URL format | `git@github.com:org/repo.git` | `https://github.com/org/repo.git` |
| Works without network (cached) | Keys stored locally, always available | Token may expire |

---

## SSH in Detail

### When to use SSH

- **Developer workstations** with a persistent SSH agent session
- **Multiple repositories from the same organization** (one key works everywhere)
- **Scripts that need to authenticate without user interaction** (key in SSH agent)
- **Multiple GitHub accounts** on one machine (different SSH aliases per account)

### Setup

```bash
# Generate a key (Ed25519 is preferred over RSA)
ssh-keygen -t ed25519 -C "your-email@example.com" -f ~/.ssh/id_ed25519_github

# Add to SSH agent
ssh-add ~/.ssh/id_ed25519_github

# Copy public key to GitHub
cat ~/.ssh/id_ed25519_github.pub
# Paste at: GitHub Settings → SSH and GPG keys → New SSH key

# Test
ssh -T git@github.com
# Hi username! You've successfully authenticated...
```

### Multiple accounts

```ssh-config
# ~/.ssh/config
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
```

```bash
# Use the alias in clone URL
git clone git@github-work:org/repo.git
git clone git@github-personal:myusername/project.git
```

---

## HTTPS in Detail

### When to use HTTPS

- **CI/CD pipelines** where deploying SSH keys is complex or against policy
- **Enterprise environments** where SSH port 22 is blocked
- **Temporary access** (short-lived tokens, OIDC)
- **Service accounts** where token scope needs to be limited (fine-grained PATs)

### Setup

```bash
# Clone with HTTPS
git clone https://github.com/org/repo.git

# On first push, Git prompts for credentials
# Enter your GitHub username and a Personal Access Token (NOT your password)
# GitHub stopped accepting passwords for Git operations in 2021

# Store credentials in macOS Keychain
git config --global credential.helper osxkeychain

# Store credentials in a credential manager
git config --global credential.helper store   # Plaintext — use only in secure environments
```

### Fine-grained Personal Access Tokens

Fine-grained PATs (GitHub Settings → Developer Settings → Fine-grained tokens) allow:
- Repository-scoped access (not all repositories in the account)
- Permission-scoped (read-only contents, write issues, etc.)
- Short expiry (7 days, 30 days, 90 days)

Prefer fine-grained PATs over classic PATs for all new automation.

### CI/CD: GitHub Actions OIDC (best practice)

For GitHub Actions, avoid long-lived tokens entirely:

```yaml
# .github/workflows/deploy.yml
permissions:
  id-token: write       # Required for OIDC
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
      aws-region: us-east-1
      # No static credentials — AWS trusts GitHub's OIDC token
```

OIDC tokens are ephemeral — they expire when the job ends. No credentials to rotate, no credentials to leak.

---

## Decision for Common Scenarios

### Scenario: New team member setup

→ **SSH** with Ed25519 key. Setup takes 10 minutes once and eliminates all future credential prompts.

### Scenario: GitHub Actions deploying to AWS

→ **OIDC** (no credentials at all). No PAT, no SSH key, no rotation required.

### Scenario: CI reads from private repository (not GitHub Actions)

→ **Deploy key** (SSH): generate a key with no passphrase, add the public key as a read-only Deploy Key on the repository. Store the private key as a CI secret.

### Scenario: Automation reads from 5 private repositories

→ **Fine-grained PAT** with read access to those 5 repositories. Deploy keys are per-repository, making PAT more manageable at this scale.

### Scenario: Enterprise with HTTP proxy and no port 22

→ **HTTPS**. Configure proxy: `git config --global https.proxy http://proxy:3128`

### Scenario: SSH required but port 22 is blocked

→ **SSH over HTTPS port**: GitHub accepts SSH connections on port 443 via `ssh.github.com`:

```ssh-config
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519
```

---

## Security Considerations

- **Never commit SSH private keys or HTTPS tokens** — use `.gitignore` for `*.pem`, `*.key`, `.env`
- **SSH key passphrase:** Always set a passphrase on SSH keys stored on developer machines. Use the SSH agent to avoid typing it repeatedly.
- **Token scopes:** Use the minimum required scope. A token with `repo` (full repository access) is too broad for most automation.
- **Token expiry:** Set expiry on all PATs. Unexpired tokens are a persistent credential leak risk.
- **Audit regularly:** GitHub → Settings → Security log shows all authentication events.

---

## Related

- [Security Reference](../security/README.md)
- [Enterprise Workflows](../enterprise-workflows/README.md)
- [Decision Guides Index](README.md)

---

[← Decision Guides Index](README.md) | [GitFlow vs Trunk-Based →](gitflow-vs-trunk-based.md)
