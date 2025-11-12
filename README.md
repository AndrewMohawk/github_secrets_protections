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
