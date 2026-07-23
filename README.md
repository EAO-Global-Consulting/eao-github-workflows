# EAO GitHub Workflows

Centralized reusable GitHub Actions workflows for EAO Holdings repositories.

## Available Workflows

### Slack Notifications (`slack-notify.yml`)

Sends Slack notifications for push events and workflow completions.

### Jira Ticket Check (`jira-check.yml`)

Blocks a PR from merging unless its branch name references a valid Jira issue key for a specific project (e.g. `DSB-475`, found anywhere in the branch name — `feat/DSB-475-card-name`, `fix/DSB-475-bug`, `DSB-475-quick-change` all match). Calls the Jira REST API to confirm the ticket exists and isn't in a disallowed status (defaults to `Done(Prod)` or `To Do` — configurable per caller via `disallowed_statuses`). Supports a `no-jira-needed` label to bypass the check for PRs that genuinely don't need a ticket.

## Usage

### 1. Create a Slack Webhook

1. Go to your Slack workspace settings
2. Create an Incoming Webhook for your desired channel
3. Add the webhook URL as a repository secret named `SLACK_WEBHOOK_URL`

### 2. Add to Your Repository

Create `.github/workflows/slack-notifications.yml` in your repo:

```yaml
name: Slack Notifications

on:
  push:
    branches:
      - '**'
  workflow_run:
    workflows: ['Your Workflow Names Here']
    types:
      - completed

jobs:
  notify-push:
    if: github.event_name == 'push'
    uses: EAO-Global-Consulting/eao-github-workflows/.github/workflows/slack-notify.yml@main
    with:
      event_type: push
      repository: ${{ github.repository }}
      branch: ${{ github.ref_name }}
      actor: ${{ github.actor }}
      commit_message: ${{ toJson(github.event.head_commit.message) }}
      commit_url: ${{ github.event.head_commit.url }}
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  notify-workflow-success:
    if: github.event_name == 'workflow_run' && github.event.workflow_run.conclusion == 'success'
    uses: EAO-Global-Consulting/eao-github-workflows/.github/workflows/slack-notify.yml@main
    with:
      event_type: workflow_success
      repository: ${{ github.repository }}
      branch: ${{ github.event.workflow_run.head_branch }}
      actor: ${{ github.event.workflow_run.actor.login }}
      workflow_name: ${{ github.event.workflow_run.name }}
      workflow_url: ${{ github.event.workflow_run.html_url }}
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  notify-workflow-failure:
    if: github.event_name == 'workflow_run' && github.event.workflow_run.conclusion == 'failure'
    uses: EAO-Global-Consulting/eao-github-workflows/.github/workflows/slack-notify.yml@main
    with:
      event_type: workflow_failure
      repository: ${{ github.repository }}
      branch: ${{ github.event.workflow_run.head_branch }}
      actor: ${{ github.event.workflow_run.actor.login }}
      workflow_name: ${{ github.event.workflow_run.name }}
      workflow_url: ${{ github.event.workflow_run.html_url }}
    secrets:
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 3. Jira Ticket Check Setup

1. Create a Jira service account (or use an existing bot account) with at least "Browse projects" permission on the relevant project(s).
2. Generate an API token for that account: Atlassian ID → Account Settings → Security → Create API token.
3. Add two repository secrets:
   - `JIRA_EMAIL` — the service account's email
   - `JIRA_API_TOKEN` — the token from step 2
4. Create `.github/workflows/jira-check.yml` in your repo:

```yaml
name: Jira Ticket Check

on:
  pull_request:
    types: [opened, edited, synchronize, reopened, labeled, unlabeled]

jobs:
  jira-check:
    uses: EAO-Global-Consulting/eao-github-workflows/.github/workflows/jira-check.yml@main
    with:
      branch: ${{ github.head_ref }}
      jira_base_url: https://yourcompany.atlassian.net
      project_key: DSB
      pr_labels: ${{ toJson(github.event.pull_request.labels.*.name) }}
    secrets:
      JIRA_EMAIL: ${{ secrets.JIRA_EMAIL }}
      JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}
```

5. In the repo's branch protection rules (Settings → Branches → Branch protection rule for `main`), add **"Jira Ticket Check / check"** to **Require status checks to pass before merging**. This is the step that actually makes it a gate — without it, the check runs but doesn't block the merge button.
6. Branch names must contain the project's Jira key (`DSB-475`) anywhere in the name — no required prefix, so `feat/DSB-475-card-name`, `fix/DSB-475-bug`, `hotfix/DSB-475`, and `DSB-475-quick-change` all pass. Matching is case-insensitive. PRs that don't need a ticket can be exempted by adding a `no-jira-needed` label, which re-triggers the check via the `labeled` event type above.

### 4. Per-Repo Channel Routing

Each repository uses its own `SLACK_WEBHOOK_URL` secret, which points to a different Slack channel:

| Repository | Slack Channel | Secret |
|------------|---------------|--------|
| joinbash-infrastructure | #joinbash-infra | `SLACK_WEBHOOK_URL` |
| konnetta-infrastructure | #konnetta-infra | `SLACK_WEBHOOK_URL` |

To route notifications to a different channel, create a new Slack Incoming Webhook for that channel and add it as the `SLACK_WEBHOOK_URL` secret in that repository.

## Repository Requirements

This repository must be:
1. Public, OR
2. Part of a GitHub Enterprise organization with reusable workflows enabled

For private repos to use these workflows, enable "Access to reusable workflows" in your organization settings.
