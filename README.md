# EAO GitHub Workflows

Centralized reusable GitHub Actions workflows for EAO Holdings repositories.

## Available Workflows

### Slack Notifications (`slack-notify.yml`)

Sends Slack notifications for push events and workflow completions.

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

### 3. Per-Repo Channel Routing

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
