# Day Log: AWS CodeBuild + GitHub Webhook Integration

Today's goal: connect a GitHub repository to AWS CodeBuild so that every `git push` automatically triggers a build, without manually clicking "Start build."

## Final working setup

- **Source provider**: GitHub
- **Credential**: AWS managed GitHub App (via CodeConnections)
- **Repository source type**: "Repository in my GitHub account"
- **Webhook**: Enabled — "Rebuild every time a code change is pushed to this repository"
- **Webhook event filter**: `PUSH`
- **Service role**: CodeBuild's IAM role with explicit `codeconnections:UseConnection` and `codeconnections:GetConnection` permissions

## Issues faced and how they were fixed

### 1. GitHub connection showed "Pending" / connection never finished
**Cause**: The GitHub authorization flow has two steps — "Authorize" and then a separate "Install" step. Stopping after the first screen leaves the connection incomplete, which AWS shows as "Pending" with no valid ARN.

**Fix**: Re-triggered the GitHub authorization flow and made sure to click all the way through to the **Install** screen (selecting specific repositories), not just **Authorize**.

### 2. "Token must be a valid CodeConnections arn"
**Cause**: Selected an incomplete/pending connection in the CodeBuild credential dropdown.

**Fix**: Deleted the broken connection, created a new one from scratch via Developer Tools → Settings → Connections, and completed the full GitHub install flow before selecting it in CodeBuild.

### 3. "Invalid source location"
**Cause**: The "GitHub scoped webhook" source option was selected instead of "Repository in my GitHub account." The scoped webhook mode doesn't bind to a fixed repo — it expects the source location to be passed in dynamically at build time, so it shows as invalid when nothing is configured to do that.

**Fix**: Switched the Repository selection to **"Repository in my GitHub account"** and picked the actual repo from the dropdown.

### 4. "Failed to create webhook. GitHub API limit reached or permission issue"
**Cause**: The "AWS Connector for GitHub" app was authorized but never actually **installed** on the GitHub account — it appeared under "Authorized GitHub Apps" but not "Installed GitHub Apps." Webhook creation requires the installed app with repo-level admin access, not just OAuth authorization.

**Fix**: Revoked the incomplete authorization, then installed the app directly from the GitHub Marketplace listing, explicitly selecting the target repository during install. Confirmed it then appeared under **Installed GitHub Apps**.

### 5. "Access denied to connection arn:aws:codeconnections:..."
**Cause**: The CodeBuild project's IAM service role didn't have permission to use the CodeConnections connection — having a working GitHub connection isn't enough if the IAM role calling it lacks the right policy.

**Fix**: Added an inline policy to the CodeBuild service role:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codeconnections:UseConnection",
        "codeconnections:GetConnection"
      ],
      "Resource": "arn:aws:codeconnections:REGION:ACCOUNT_ID:connection/CONNECTION_ID"
    }
  ]
}
```

## Verification

After fixing the above, a `git push` triggered an automatic build with **Submitter: `GitHub-Hookshot/<id>`**, confirming the webhook fired correctly without any manual "Start build" click.

## Key takeaway

The error messages from AWS in this flow are often misleading about the *actual* root cause:
- "Invalid source location" was really a wrong-source-type problem, not a broken location string.
- "Permission issue" in the webhook error was a GitHub App installation problem, not a GitHub API rate limit.
- "Access denied" was an IAM role policy gap, not a connection/auth problem.

When debugging AWS + GitHub integration errors, check (in order): **GitHub App installation status → IAM role permissions → source type configuration** — rather than trusting the literal error text.
