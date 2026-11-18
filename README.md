# Quackbot

Quackbot is a Linear coding agent. You delegate a Linear issue to it, and it clarifies the issue, plans it, and later builds it in your repository, on your own runner and with your own Claude subscription.

This action is only the launcher. Quackbot works only in repositories its owner has linked, so using this action elsewhere does nothing.

## Use

```yaml
# .github/workflows/linear-agent.yml
name: Linear agent
on:
  repository_dispatch:
    types: [linear-agent]
permissions:
  contents: read
  id-token: write
concurrency:
  group: linear-agent-${{ github.event.client_payload.issueId }}
  cancel-in-progress: false
jobs:
  quackbot:
    runs-on: ubuntu-latest
    timeout-minutes: 70
    steps:
      - uses: rubber-duck-ops/quackbot-action@v1
        with:
          claude-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          relay-url: ${{ secrets.QUACKBOT_RELAY_URL }}
```

Set both values as repository secrets:

- `CLAUDE_CODE_OAUTH_TOKEN`: the token from `claude setup-token`;
- `QUACKBOT_RELAY_URL`: the relay address you received when you were onboarded.
