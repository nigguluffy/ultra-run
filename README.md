# QuizBots CI Workflow

This repository hosts a GitHub Actions workflow to automate running multiple Python-based quiz bots (e.g., TSS-Bot, BCS Odyssey Bot) by cloning their repositories, installing dependencies, and executing their `main.py` scripts. A looping mechanism re-triggers the workflow via an empty commit, enabling continuous operation.

## Overview

The workflow:
- **Triggers**: Runs on pushes to the `main` branch or manual dispatch.
- **Dynamic Bots**: Uses a matrix strategy to run 1 to 12 (or more) bots, configured via a single secret (`BOT_REPOS`).
- **Jobs**:
  - `quizbots`: Clones and runs each bot in parallel, using Python 3.10.
  - `looper`: Creates an empty commit to re-run the workflow after all bots complete.
- **Timeouts**: 340 minutes per bot run, 5 minutes for setup steps.
- **Debugging**: Enables detailed logging for troubleshooting.

This repository is a "runner" repo (typically empty except for the workflow). Bot repositories are cloned dynamically and do not need to be part of this repo.

## Prerequisites

- **GitHub Account**: You need a GitHub account to create the runner repo and manage secrets.
- **Bot Repositories**: Each bot must have:
  - A `requirements.txt` file listing Python dependencies.
  - A `main.py` script as the entry point.
  - Public or private visibility (private requires token access).
- **GitHub Personal Access Token (PAT)**: A token with `repo` scope to clone bot repos and push commits to this repo.

## Setup Instructions

### 1. Create the Runner Repository
1. Go to GitHub > New repository.
2. Name: e.g., `quizbots-runner`.
3. Visibility: Public (recommended for simplicity; private works too).
4. Initialize with no files (no README, .gitignore, etc.).
5. Create the repository.

### 2. Prepare Bot Repositories
Each bot needs its own GitHub repository (e.g., `yourusername/tss-bot`, `yourusername/bcs-bot`).
- Ensure each repo has:
  - `requirements.txt` (e.g., `requests==2.28.1`).
  - `main.py` (your bot's main script).
- Repos can be public or private. If private, the PAT must have access (see Step 3).
- Bots can be in the same account as the runner or different accounts, but the token needs access to all.

### 3. Generate a GitHub Personal Access Token
1. Go to GitHub Settings > Developer settings > Personal access tokens > Tokens (classic) > Generate new token.
2. Name: e.g., "QuizBots Workflow Token".
3. Expiration: Choose as needed (e.g., no expiration).
4. Scopes: Select `repo` (includes all sub-scopes for cloning and pushing).
5. Generate and copy the token (e.g., `ghp_abc123...`).
6. If bot repos are in a different account, add the token's account as a collaborator to those repos or use an organization token with access.

### 4. Add Secrets to the Runner Repository
1. In the runner repo (e.g., `quizbots-runner`), go to Settings > Secrets and variables > Actions > New repository secret.
2. Add the following secrets:
   - `GITHUBMAIL`: Your GitHub email (e.g., `user@example.com`).
   - `GITHUBNAME`: Your GitHub username (e.g., `yourusername`).
   - `GH_TOKEN`: The PAT from Step 3 (e.g., `ghp_abc123...`).
   - `BOT_REPOS`: A JSON array of bot repository slugs (e.g., `["yourusername/tss-bot", "yourusername/bcs-bot"]`).
     - Example for 3 bots: `["yourusername/tss-bot", "yourusername/bcs-bot", "yourusername/quiz-bot3"]`.
     - For 1 bot: `["yourusername/tss-bot"]`.
     - For 10 bots: List up to 10 slugs (e.g., `["yourusername/bot1", ..., "yourusername/bot10"]`).

### 5. Add the Workflow File
1. In the runner repo, create a file: `.github/workflows/quizbots-ci.yaml`.
2. Copy the workflow YAML content (provided separately in this repo or from the guide).
3. Commit and push to the `main` branch. This triggers the first run.

### 6. Run and Monitor
1. Go to the runner repo > Actions tab.
2. Find the "CI for QuizBots" workflow.
3. Trigger manually via "Run workflow" if needed.
4. Monitor logs:
   - Each bot runs in parallel (matrix strategy).
   - Check for errors in cloning, dependency installation, or script execution.
   - The `looper` job runs after all bots complete, creating an empty commit to restart the workflow.
5. To stop the loop:
   - Delete or rename `.github/workflows/quizbots-ci.yaml`.
   - Disable the workflow in the Actions tab.
   - Add a delay (e.g., `sleep 3600` in the looper) to slow the loop.

## Configuring the Number of Bots
To change the number of bots (1 to 12 or more):
1. Go to Settings > Secrets and variables > Actions.
2. Update the `BOT_REPOS` secret with a new JSON array.
   - Example for 2 bots: `["yourusername/tss-bot", "yourusername/bcs-bot"]`.
   - Example for 10 bots: `["yourusername/bot1", "yourusername/bot2", ..., "yourusername/bot10"]`.
   - For 1 bot: `["yourusername/tss-bot"]`.
3. Save the secret. The next workflow run will use the updated list.
4. Note: GitHub's matrix strategy has limits (e.g., 256 total jobs). For free accounts, concurrent jobs are capped (e.g., 20). Monitor rate limits.

## Example Configuration
Assume:
- Username: `quizmaster`
- Email: `quiz@master.com`
- Token: `ghp_exampletoken123`
- Runner Repo: `quizmaster/quizbots-runner`
- Bot Repos: `quizmaster/tss-bot`, `quizmaster/bcs-bot`, `quizmaster/quiz-bot3`

Secrets:
- `GITHUBMAIL`: `quiz@master.com`
- `GITHUBNAME`: `quizmaster`
- `GH_TOKEN`: `ghp_exampletoken123`
- `BOT_REPOS`: `["quizmaster/tss-bot", "quizmaster/bcs-bot", "quizmaster/quiz-bot3"]`

Result: Workflow runs 3 bots in parallel, then loops via an empty commit.

## Troubleshooting
- **Cloning Fails**: Check `GH_TOKEN` scopes (`repo` required) or repo slugs in `BOT_REPOS`. Ensure repos exist and are accessible.
- **Push Fails in Looper**: Verify `GH_TOKEN` has write access to the runner repo.
- **Bot Run Fails**: Confirm `requirements.txt` and `main.py` exist in each bot repo. Check logs for dependency or script errors.
- **Rate Limits**: GitHub may limit frequent runs (e.g., 60 runs/hour for Pro accounts). Add `sleep 3600` in the looper's `run` step for a 1-hour delay.
- **Timeouts**: Bots exceeding 340 minutes? Optimize scripts or increase `timeout-minutes` (max 360 minutes per job).
- **Debugging**: Use `ACTIONS_RUNNER_DEBUG` and `ACTIONS_STEP_DEBUG` logs (enabled by default). View in Actions tab.

## Security Notes
- **Token Security**: Store `GH_TOKEN` only in secrets. Never hardcode in YAML. Use minimal scopes (`repo` only).
- **Private Repos**: Ensure `GH_TOKEN` has access via collaborator settings or org membership.
- **Loop Control**: Uncontrolled looping can hit GitHub rate limits. Monitor usage or add delays.

## Extending the Workflow
- **Notifications**: Add actions like `slack-notify` or `email-notify` for run failures.
- **Conditional Looping**: Add a condition to stop looping (e.g., based on time or external flag).
- **Custom Python Versions**: Modify `python-version` in the `setup-python` step if bots require different versions.

## References
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Using Secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions)
- [Personal Access Tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

For further assistance, check workflow logs or contact the repository owner.
