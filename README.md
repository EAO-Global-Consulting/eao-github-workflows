# EAO GitHub Workflows

Centralized reusable GitHub Actions workflows for EAO Holdings repositories.

## Available Workflows

### Slack Notifications (`slack-notify.yml`)

Sends Slack notifications for push events and workflow completions.

### Jira Ticket Check (`jira-check.yml`)

Blocks a PR from merging unless its branch name references a valid Jira issue key for a specific project (e.g. `DSB-475`, found anywhere in the branch name — `feat/DSB-475-card-name`, `fix/DSB-475-bug`, `DSB-475-quick-change` all match), **and** the ticket is currently in the exact status the caller requires (e.g. `Dev Done`). On merge, it can also transition the ticket forward automatically (e.g. to `Done(Prod)`), so the board reflects real deploy state without relying on developers to move cards by hand. Supports a `no-jira-needed` label to bypass both the pre-merge check and the post-merge transition.

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
4. Create `.github/workflows/jira-check.yml` in your repo. Each target branch needs two jobs: one that gates the PR (`required_status`) and one that fires on merge (`merged: true`, `transition_to`):

```yaml
name: Jira Ticket Check

on:
  pull_request:
    types: [opened, edited, synchronize, reopened, labeled, unlabeled, closed]
    branches: [main]

jobs:
  jira-check:
    if: github.event.action != 'closed'
    uses: EAO-Global-Consulting/eao-github-workflows/.github/workflows/jira-check.yml@main
    with:
      branch: ${{ github.head_ref }}
      pr_title: ${{ github.event.pull_request.title }}
      jira_base_url: https://yourcompany.atlassian.net
      project_key: DSB
      pr_labels: ${{ toJson(github.event.pull_request.labels.*.name) }}
      required_status: 'Dev Done'
    secrets:
      JIRA_EMAIL: ${{ secrets.JIRA_EMAIL }}
      JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}

  jira-transition:
    if: github.event.action == 'closed' && github.event.pull_request.merged == true
    uses: EAO-Global-Consulting/eao-github-workflows/.github/workflows/jira-check.yml@main
    with:
      branch: ${{ github.head_ref }}
      pr_title: ${{ github.event.pull_request.title }}
      jira_base_url: https://yourcompany.atlassian.net
      project_key: DSB
      pr_labels: ${{ toJson(github.event.pull_request.labels.*.name) }}
      required_status: 'Dev Done'
      merged: true
      transition_to: 'Done(Prod)'
    secrets:
      JIRA_EMAIL: ${{ secrets.JIRA_EMAIL }}
      JIRA_API_TOKEN: ${{ secrets.JIRA_API_TOKEN }}
```

For a repo with multiple gated branches requiring different statuses (e.g. a `preprod` step before `main`), duplicate both jobs per branch and guard each with `github.base_ref == '<branch>'` — see `Konnetta-Mobile-app-dev/.github/workflows/jira-check.yml` for a working example with two gates.

5. In the repo's branch protection rules (Settings → Branches → Branch protection rule for each gated branch), add the check job name(s) (e.g. **"Jira Ticket Check / jira-check"**) to **Require status checks to pass before merging**. This is the step that actually makes it a gate — without it, the check runs but doesn't block the merge button. The `jira-transition` job should NOT be added as a required check — it only runs after merge and has nothing to block.
6. The Jira key is looked up first in the branch name, then in the PR title if the branch name has none. Feature branches (`feat/DSB-475-card-name`) are covered automatically. For promotion PRs between persistent environment branches (e.g. `dev` → `main`, where the branch name itself will never contain a ticket key), put the key in the **PR title** instead — e.g. `DSB-489: promote to prod`. Matching is case-insensitive, and the key can appear anywhere in either string. PRs that don't need a ticket can be exempted by adding a `no-jira-needed` label, which re-triggers the check via the `labeled` event type above.
7. `transition_to` failures never fail the workflow (the PR is already merged by then, so there's nothing left to block) — they post a `::warning` instead. Check the Actions log if a card doesn't move; the most common cause is that Jira's workflow doesn't allow a direct transition from the ticket's current status to the target status.

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
