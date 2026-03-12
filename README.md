# jsk-printing Codex Bot

Automated PR review bot for the `Jai-jawa/jsk-printing` repo using an OpenAI model and GitHub Actions.

## How it works

- A PR opens/updates in `jsk-printing`
- A workflow in `jsk-printing` gathers a diff and triggers this repo’s workflow (`workflow_dispatch`)
- This repo calls OpenAI to generate a review and posts it back as a PR comment

## Repo setup (this repo: `jsk-printing-codex-bot`)

### Secrets (Settings → Secrets and variables → Actions)

- **`OPENAI_API_KEY`**: OpenAI API key
- **`GITHUB_TOKEN_JSK`**: GitHub token that can **comment on PRs** in `Jai-jawa/jsk-printing`
  - Recommended: **Fine-grained PAT**
  - Permissions: `Issues: Read and write` on the `jsk-printing` repo (PR comments are issue comments)

### (Optional) Variables

Add Actions variable:

- **`OPENAI_MODEL`**: defaults to `gpt-4o-mini` if not set

## Integration setup (in `Jai-jawa/jsk-printing`)

Create `.github/workflows/trigger-codex-bot.yml`.

Important: **`GITHUB_TOKEN` cannot trigger workflows in a different repository.**  
You must create a PAT that can dispatch workflows in `Jai-jawa/jsk-printing-codex-bot` and store it as a secret in `jsk-printing`, e.g. **`CODEX_BOT_DISPATCH_TOKEN`**.

Suggested fine-grained PAT permissions (scoped to `jsk-printing-codex-bot`):
- `Actions: Read and write`

Then use it like this:

```yaml
name: Trigger Codex Bot
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main, dev]

jobs:
  trigger-bot:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get PR diff
        id: diff
        run: |
          git fetch origin pull/${{ github.event.pull_request.number }}/head:pr-branch
          DIFF="$(git diff origin/${{ github.base_ref }} pr-branch | head -c 30000)"
          echo "diff<<EOF" >> "$GITHUB_OUTPUT"
          echo "$DIFF" >> "$GITHUB_OUTPUT"
          echo "EOF" >> "$GITHUB_OUTPUT"

      - name: Trigger Codex Bot Review
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.CODEX_BOT_DISPATCH_TOKEN }}
          script: |
            await github.rest.actions.createWorkflowDispatch({
              owner: 'Jai-jawa',
              repo: 'jsk-printing-codex-bot',
              workflow_id: 'codex-review.yml',
              ref: 'main',
              inputs: {
                pr_number: String(${{ github.event.pull_request.number }}),
                repo: 'Jai-jawa/jsk-printing',
                diff: `${{ steps.diff.outputs.diff }}`
              }
            });
```

## Testing

1. Open a PR in `jsk-printing`.
2. Confirm `Trigger Codex Bot` runs.
3. In `jsk-printing-codex-bot`, confirm `Codex Review (External)` runs.
4. Verify a comment is posted on the PR.
