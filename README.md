# Quackbot 🦆

Quackbot is a Linear coding agent. Delegate a Linear issue to it and it writes the spec, builds the change test-first,
reviews its own work and opens a pull request in your repository. It runs on your GitHub Actions runner and your own
Claude subscription.

This action is the launcher. It holds no code of the agent itself and no secrets. Quackbot only works in repositories
that are linked to a Linear project (see [Setup](#setup)), so on any other repository the action does nothing.

## What a run does

Delegating an issue (or @mentioning Quackbot on it) starts one run that goes through these stages:

| Stage | What happens | Model |
|---|---|---|
| Intake | Classifies the issue as feature, bugfix or chore, and flags risky and UI changes | Opus |
| Spec writer | Rewrites the issue description as a spec: Problem, Acceptance criteria, Out of scope, Notes. Your original text is kept in a collapsed section. Skipped when the issue already uses that template | Opus |
| TDD | Writes a failing test for each acceptance criterion | Sonnet |
| Builder | Writes the code until the tests pass | Opus |
| Gate | Runs your test, lint, typecheck and build commands. No AI involved | — |
| Review | Checks the diff against the acceptance criteria | Sonnet |
| Code review | Looks for bugs and weak tests | Haiku, or Sonnet for risky changes |
| UX review | Only for changes that touch the UI | Sonnet |
| Security review | Only for risky changes (auth, secrets, migrations, infra, CI) | Sonnet |

Gate failures and blocker or major review findings go back to the Builder, for up to two fix rounds. Then Quackbot
pushes the branch `agent/<issue-id>`, opens a pull request and posts a "🦆 Quackbot review" on it with every stage and
finding. When findings are still open after two rounds, the pull request opens as a draft.

When Quackbot needs a decision, it asks in the Linear session and waits. Answer there to continue.

Quackbot never merges, approves or closes a pull request. A person reviews and merges every change.

## Setup

You need four things. Quackbot only starts when all of them are in place, and otherwise posts an error on the issue that
names the missing piece.

**1. Install the Quackbot GitHub App** on the repository:
<https://github.com/apps/rubber-duck-quackbot/installations/new>

**2. Link the Linear project to the repository.** In the Linear project, add a GitHub link to the repository under
Resources, or put a line `repository: owner/name` in the project description.

**3. Add `.github/quackbot.yml`** and list the Linear project's URL:

```yaml
linear_projects:
  - https://linear.app/your-workspace/project/your-project-0123456789ab
```

**4. Add the workflow** as `.github/workflows/linear-agent.yml`:

```yaml
name: Linear agent

on:
  repository_dispatch:
    types: [linear-agent]

permissions:
  contents: read
  id-token: write # proves the run's identity to Quackbot

concurrency:
  group: linear-agent-${{ github.event.client_payload.issueId }}
  cancel-in-progress: false

jobs:
  quackbot:
    runs-on: ubuntu-latest
    timeout-minutes: 180 # a run makes several agent calls
    steps:
      - uses: rubber-duck-ops/quackbot-action@v1
        with:
          claude-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          relay-url: ${{ secrets.QUACKBOT_RELAY_URL }}
```

Then add two repository (or organization) secrets:

| Secret | Value |
|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | The token from `claude setup-token`. The run uses your Claude subscription. |
| `QUACKBOT_RELAY_URL` | The relay address you got when you were onboarded. |

Do not set `ANTHROPIC_API_KEY` in the workflow. Quackbot refuses to start when it is present, so a run never bills an API key by accident.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `claude-token` | yes | | Your `CLAUDE_CODE_OAUTH_TOKEN`, from a secret. |
| `relay-url` | yes | | Your `QUACKBOT_RELAY_URL`, from a secret, so it stays masked in logs. |
| `model` | no | Opus | The model for the Builder stage, for example `sonnet`. The other stages keep their own models. |
| `timeout-minutes` | no | `60` | Minutes before one agent call is stopped. The whole job runs several, so keep the job's `timeout-minutes` well above this. |
| `doodles` | no | `on` | Short playful duck updates in Linear while Quackbot works. Set to `off` to hide them. |

## Configure the pipeline

`.github/quackbot.yml` takes these optional keys next to `linear_projects`:

```yaml
# Status that also starts a run when a delegated issue is moved into it.
build_state: Ready for Agent

# Commands the Gate runs, in order. All must succeed.
# Without this key the Gate runs the npm scripts typecheck, lint, test and build that exist (after `npm ci` when needed).
gate:
  - npm run lint
  - npm test

# Extra skills for the TDD and Builder stages, and for every review stage.
build_skills:
  - my-plugin:react-conventions
review_skills:
  - security-checklist

# true: use only your skills for those stages and drop Quackbot's defaults.
replace_skills: false
```

Quackbot's default skills come from [mattpocock/skills](https://github.com/mattpocock/skills): `tdd`, `codebase-design`,
`diagnosing-bugs`, `code-review`, `writing-for-agents` and `domain-modeling`. Skills in your repository's
`.claude/skills` are loaded too, and every stage follows your `CLAUDE.md` or `AGENTS.md`.

### Break an issue down into tickets

Add the label `breakdown` to an issue and delegate it. Instead of building, Quackbot reads the repository and creates up
to 20 sub-issues in the same template, with "blocks" relations for their order.

## Security

- Your repository holds no Quackbot secrets except your own Claude token. The run proves who it is with a short-lived
  GitHub OIDC token, and the relay only answers runs from linked repositories.
- The agent never gets push rights. After it finishes, the runner pushes with a token that is limited to this
  repository and expires within the hour.
- A push is refused when a commit touches `.github/workflows/` or the checked-out toolkit in `.quackbot/`.
- The Gate commands and the agent run without the OIDC request token, so code in your repository cannot act as the run.

## Troubleshooting

| Symptom | Cause |
|---|---|
| An error on the Linear issue names a missing App, link or `quackbot.yml` | Finish that step of [Setup](#setup). Link checks are cached for five minutes. |
| The job is cancelled before a pull request opens | The job's `timeout-minutes` is too low. Use 180. |
| "Quackbot is already running this issue" | A run for the issue started less than two minutes ago. Follow that session. |
| A usage-limit error | Your Claude subscription hit its limit. The error says when it resets; delegate again after that. |
| The pull request is a draft | Findings were still open after two fix rounds. They are listed in the Quackbot review on the PR. |
