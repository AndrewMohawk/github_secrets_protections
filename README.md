# GitHub Secrets Protections Playground

This repository demonstrates how different types of GitHub Actions secrets behave when repository contributors try to exfiltrate them. The goal is to make it easy to reproduce common misconfigurations, show why environments matter, and offer a safe sandbox for experimenting with approval flows.

> ⚠️ This project intentionally prints secrets to the workflow logs. **Never** copy these patterns into production workflows.

## Overview

GitHub masks exact secret values in logs, so each workflow uses light string manipulation (splitting characters or reversing strings) to avoid automatic masking. This keeps the workflows illustrative while still highlighting which access controls succeed or fail.

| Workflow | Trigger | What it shows |
| --- | --- | --- |
| [`repo-secret-exfiltration.yml`](.github/workflows/repo-secret-exfiltration.yml) | `workflow_dispatch` | Repository-level secrets are available to anyone who can run a workflow. |
| [`environment-secret-exfiltration.yml`](.github/workflows/environment-secret-exfiltration.yml) | `workflow_dispatch` | Non-protected environments do **not** stop contributors with write access from reading environment secrets. |
| [`protected-environment-secret-exfiltration.yml`](.github/workflows/protected-environment-secret-exfiltration.yml) | `workflow_dispatch` | Protected environments add an approval gate before jobs can access their secrets. |
| [`print-everything.yml`](.github/workflows/print-everything.yml) | `workflow_dispatch` | Convenience workflow that reads every configured secret in one run. |

## Prerequisites

1. **Repository secret**
   * Name: `REPO_LEVEL_SECRET`
   * Value: anything distinct so you can recognise it in logs
   * Screenshot placeholder: `![Repository secrets settings](docs/images/repo-secret-settings.png)`
2. **Environment: `demo-environment`** (unprotected)
   * Secrets: add `DEMO_ENV_SECRET`
   * Screenshot placeholder: `![Demo environment settings](docs/images/demo-environment-settings.png)`
3. **Environment: `protected-environment`** (requires approval)
   * Require reviewers or wait timer in Settings → Environments → `protected-environment`
   * Secrets: add `PROTECTED_ENV_SECRET`
   * Screenshot placeholders:
     * `![Protected environment rules](docs/images/protected-environment-rules.png)`
     * `![Protected environment secrets](docs/images/protected-environment-secrets.png)`

> Tip: Store screenshots in `docs/images/` so the placeholders above render automatically.

## Running the Workflows

All workflows are manually triggered:

1. Open the **Actions** tab.
2. Select the workflow you want to explore.
3. Click **Run workflow**.
4. Monitor the job logs to see how each type of secret behaves.

### Expected outcomes

* **Repository secret exfiltration** – Succeeds immediately because repository secrets are available to anyone with write access (or pull request workflows with `pull_request_target`).
* **Environment secret exfiltration** – Succeeds immediately, demonstrating that environments without protection behave the same as repository secrets.
* **Protected environment exfiltration** – Stalls until an approver authorises the run. Once approved, the job prints the secret using the same masking bypass technique.
* **Print everything** – Runs three jobs in sequence, each targeting one of the secrets above to provide a single log trail.

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

Because each character is separated by a space, GitHub does not detect the exact secret value, but observers can easily recover it.

## Cleanup

If you run these workflows in a public repository, rotate or delete the secrets afterwards. Consider adding a `README` warning (like the one at the top of this document) to deter accidental reuse of these patterns.

