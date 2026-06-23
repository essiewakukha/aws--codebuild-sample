# AWS CodeBuild + GitHub Webhook Integration

Connected a GitHub repository to AWS CodeBuild so that every `git push` automatically triggers a build — no manual "Start build" needed.

## What was built

- AWS CodeBuild project connected to GitHub via an AWS-managed GitHub App (CodeConnections)
- Webhook configured to trigger on `PUSH` events
- IAM service role updated with explicit permissions to use the GitHub connection
- Verified end-to-end: pushing a commit triggers a build automatically (confirmed via `Submitter: GitHub-Hookshot` in build history)

## Stack

- **AWS**: CodeBuild, IAM, CodeConnections
- **CI/CD trigger**: GitHub webhook (push event)
- **Source control**: Git/GitHub

## Files

- `CODEBUILD-WEBHOOK-SETUP.md` — full troubleshooting log: every error hit (pending connections, invalid source location, webhook permission failures, IAM access denied) and how each was fixed

## Why this matters

Most of the real work here wasn't writing code — it was diagnosing misleading AWS error messages and tracing them back to their actual cause (GitHub App installation state, source type mismatch, IAM policy gaps). That debugging process is documented in full in `CODEBUILD-WEBHOOK-SETUP.md`.
