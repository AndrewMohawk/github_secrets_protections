# GitHub Secrets Protections Playground

This repository demonstrates a common **GitHub Actions footgun**: how secrets can be exfiltrated through pull requests. This is a critical security issue that catches many teams by surprise.

> ⚠️ This project intentionally prints secrets to the workflow logs. **Never** copy these patterns into production workflows.

## The Footgun

When workflows are triggered by `pull_request` events, they run with repository secrets accessible to the workflow context—even when triggered by external contributors on forked branches. This means:

- **Repository-level secrets** are exposed to pull request workflows
- **Environment secrets** (even from "protected" environments) can be bypassed if not properly configured
- Contributors without write access can potentially exfiltrate secrets by opening a malicious PR

## Overview

GitHub masks exact secret values in logs, so these workflows use light string manipulation (pipe-separated characters or ASCII codes) to demonstrate the vulnerability while avoiding automatic masking.

| Workflow | Trigger | What it shows |
| --- | --- | --- |
| [`exfil-repo-secrets.yml`](.github/workflows/exfil-repo-secrets.yml) | `pull_request` | All three types of secrets are accessible on pull requests **without** environment declarations. This is the most dangerous scenario. |
| [`exfil-repo-with-env.yml`](.github/workflows/exfil-repo-with-env.yml) | `pull_request` | Shows that even with environment declarations, secrets remain accessible on pull requests unless the environments are properly configured with branch protection. |

## Secrets

1. **Repository secret**
   * Name: `REPO_LEVEL_SECRET`
   * This secret is the stock secret that has been added to the repo, it has no protection from people with write access.
   * <img width="1120" height="812" alt="image" src="https://github.com/user-attachments/assets/ebf224e4-711b-48d1-b30b-a5925c6f089e" />
2. **Environment: `UNPROTECTED_ENVIRONMENT`** (unprotected)
   * Secrets: `DEMO_ENV_SECRET`
   * While this secret _is_ in an environment, that environement has no rules so its still available to anyone!
   * <img width="1157" height="1043" alt="image" src="https://github.com/user-attachments/assets/bfd83968-51e1-406c-8524-b759fd465fd4" />

3. **Environment: `PROTECTED_ENVIRONMENT`** (requires approval)
   * Secrets: `PROTECTED_ENV_SECRET`
   * <img width="1078" height="1402" alt="image" src="https://github.com/user-attachments/assets/bad547a4-9cb0-4be8-94e4-aeaece242b0b" />
   * <img width="1081" height="907" alt="image" src="https://github.com/user-attachments/assets/5d842dca-a541-465b-9d27-921efb111bc5" />



> Tip: Store screenshots in `docs/images/` so the placeholders above render automatically.

## Reproducing the Vulnerability

The workflows are triggered automatically on pull requests. To see the vulnerability in action:

1. Create a new branch with a trivial change (or use an existing PR)
2. Open a pull request to the main branch
3. Navigate to the **Actions** tab and observe the workflow runs
4. Check the job logs to see the exfiltrated secrets printed in plain view

> **Note:** You may need to enable Actions for pull requests from first-time contributors in your repository settings.

### Expected Outcomes

* **exfil-repo-secrets.yml** – All three secrets (repo-level, unprotected environment, and protected environment) are printed successfully, even though the workflow doesn't declare environment contexts. This shows that repository-level access to secrets bypasses environment protections.

* **exfil-repo-with-env.yml** – Even when properly declaring environment contexts, the secrets are still accessible on pull request triggers. This demonstrates that environment configuration alone isn't enough—you need additional branch protection rules.

### The Real Danger

The existing PR #5 ("NOT STEALING CREDS") is an example of what a malicious actor might submit. The PR appears innocent but triggers workflows that exfiltrate sensitive credentials. This is why you should:

- **Never** use `pull_request` triggers with workflows that access secrets
- Use `pull_request_target` with extreme caution and never checkout untrusted code
- Configure environment protection rules to require approval for sensitive deployments
- Prefer `workflow_dispatch` for sensitive operations that require secrets

## How the masking bypass works

GitHub redacts any log line that contains a secret value exactly as stored. By breaking the value apart, we can still reveal the underlying string:

```bash
python - <<'PY'
import os
value = os.environ['REPO_LEVEL_SECRET']
print('Value length:', len(value))
print('Reconstructed:', ' '.join(value))
PY
```

Because each character is separated by a space (or converted to ASCII codes), GitHub does not detect the exact secret value, but observers can easily recover it.

## How to Protect Your Secrets

### ✅ Recommended Practices

1. **Avoid `pull_request` triggers for workflows that use secrets**
   - Use `workflow_dispatch` for manual, trusted operations
   - Consider `workflow_run` to trigger workflows after PR checks pass

2. **Use environment protection rules**
   - Require manual approval for deployments
   - Restrict environments to protected branches only
   - Configure "Wait timer" for additional safety

3. **Use `pull_request_target` carefully**
   - Never checkout PR code with `actions/checkout@v2` without ref pinning
   - Only use trusted, reviewed code with secrets
   - See [GitHub's security hardening guide](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

4. **Audit workflow permissions**
   - Set `permissions` to minimum required scope
   - Use fine-grained personal access tokens where applicable
   - Regularly review who has write access to the repository

### ⚠️ What Doesn't Work

- ❌ Creating an "environment" without protection rules (as shown by `UNPROTECTED_ENVIRONMENT`)
- ❌ Assuming environment declarations in workflows automatically protect secrets
- ❌ Relying solely on secret masking in logs (as this repo demonstrates)

## Learn More

- [GitHub Actions Security Best Practices](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [Keeping your GitHub Actions and workflows secure](https://github.blog/2021-04-22-github-actions-update-helping-maintainers-combat-bad-actors/)
- [Preventing pwn requests](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)
