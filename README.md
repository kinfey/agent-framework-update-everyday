# Agent Framework Daily Updates

![bg](./imgs/bg.png)

Automated daily tracking of [Microsoft Agent Framework](https://github.com/microsoft/agent-framework) changes using **GitHub Copilot CLI** and **GitHub Copilot SDK**.

## Overview

This project automatically analyzes merged Pull Requests from Microsoft Agent Framework repository and generates comprehensive blog posts summarizing the changes. It leverages the power of GitHub Copilot to understand code changes and produce high-quality technical content.

## Project Structure

```
.
├── .copilot_skills/
│   └── pr-analyzer/
│       └── SKILL.md          # Copilot skill definition for PR analysis
├── .github/
│   └── workflows/
│       └── daily-pr-analysis.yml  # GitHub Action workflow
├── blog/                     # Generated blog posts output directory
├── pr_trigger_v2.py          # Main Python script to trigger analysis
└── README.md
```

## How It Works

### Architecture

![arch](./imgs/arch.png)

### GitHub Action Workflow Steps

The workflow ([.github/workflows/daily-pr-analysis.yml](.github/workflows/daily-pr-analysis.yml)) runs Monday to Friday at UTC 00:00:

| Step | Action | Description |
|------|--------|-------------|
| 1 | **Checkout repository** | Clone the repository using `actions/checkout@v4` |
| 2 | **Setup Node.js** | Configure Node.js 22 environment for Copilot CLI |
| 3 | **Install GitHub Copilot CLI** | Install via `npm i -g @github/copilot` |
| 4 | **Setup Python** | Configure Python 3.11 environment |
| 5 | **Install Python dependencies** | Install `github-copilot-sdk` package |
| 6 | **Run PR Analysis** | Execute `pr_trigger_v2.py` with Copilot authentication |
| 7 | **Commit and push** | Auto-commit generated blog posts to repository |

### Key Components

#### 1. GitHub Copilot CLI (`@github/copilot`)

The command-line interface that provides AI capabilities in CI/CD pipelines:

```bash
npm i -g @github/copilot
```

#### 2. GitHub Copilot SDK (`github-copilot-sdk`)

Python SDK for programmatic interaction with GitHub Copilot:

```python
from copilot import CopilotClient

client = CopilotClient()
await client.start()

session = await client.create_session({
    "model": "claude-sonnet-4.5",
    "streaming": True,
    "skill_directories": ["./.copilot_skills/pr-analyzer/SKILL.md"]
})

await session.send_and_wait({"prompt": "Analyze PRs from microsoft/agent-framework merged yesterday"})
```

#### 3. Copilot Skill Definition

The skill file (`.copilot_skills/pr-analyzer/SKILL.md`) defines:
- PR analysis behavior
- Blog post structure
- Breaking changes priority
- Code snippet extraction rules

## Setup

### Prerequisites

- GitHub Copilot subscription with CLI access
- Repository with Actions enabled

### Configuration

1. **Create a Fine-grained Personal Access Token (PAT)**
   - Go to GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens
   - Create token with **Copilot Requests: Read** permission

2. **Add Repository Secret**
   - Navigate to repository → Settings → Secrets and variables → Actions
   - Add new secret: `COPILOT_GITHUB_TOKEN` with your PAT value

3. **Enable Workflow**
   - The workflow runs automatically on schedule
   - Manual trigger available via "Run workflow" button in Actions tab

## Output

Generated blog posts are saved to the `blog/` directory with naming convention:
```
blog/agent-framework-pr-summary-{YYYY-MM-DD}.md
```

### Blog Post Structure

Each generated post includes:
1. **Breaking Changes** (highlighted first)
2. **Major Updates** with code examples
3. **Minor Updates and Bug Fixes**
4. **Summary** and impact assessment

## Manual Trigger

You can manually trigger the workflow:

```bash
gh workflow run daily-pr-analysis.yml
```

Or via GitHub UI: Actions → Daily PR Analysis → Run workflow

## Blog Posts

<!-- BLOG_LIST_START -->
| 2026-02-05 | Agent Framework Updates - February 5, 2026 | [Read](./blog/agent-framework-pr-summary-2026-02-05.md) |<br/>
| 2026-02-04 | Agent Framework Updates - February 4, 2026 | [Read](./blog/agent-framework-pr-summary-2026-02-04.md) |<br/>
| 2026-02-03 | Agent Framework Updates - February 3, 2026 | [Read](./blog/agent-framework-pr-summary-2026-02-03.md) |<br/>
| 2026-02-02 | Agent Framework Updates - February 2, 2026 | [Read](./blog/agent-framework-pr-summary-2026-02-02.md) |<br/>
| 2026-02-01 | Agent Framework Updates - February 1, 2026 | [Read](./blog/agent-framework-pr-summary-2026-02-01.md) |<br/>
| 2026-01-29 | Agent Framework Updates - January 29, 2026 | [Read](./blog/agent-framework-pr-summary-2026-01-29.md) |<br/>
| 2026-01-28 | Agent Framework Updates - January 28, 2026 | [Read](./blog/agent-framework-pr-summary-2026-01-28.md) |<br/>
| 2026-01-27 | Agent Framework Updates - January 27, 2026 | [Read](./blog/agent-framework-pr-summary-2026-01-27.md) |<br/>
| 2026-01-26 | Agent Framework Updates - January 26, 2026 | [Read](./blog/agent-framework-pr-summary-2026-01-26.md) |<br/>
| 2026-01-23 | Agent Framework Updates - January 23, 2026 | [Read](./blog/agent-framework-pr-summary-2026-01-23.md) |
<!-- BLOG_LIST_END -->

## References

- [GitHub Copilot CLI Documentation](https://docs.github.com/en/copilot/github-copilot-in-the-cli)
- [Microsoft Agent Framework](https://github.com/microsoft/agent-framework)
- [Injecting AI Agents into CI/CD](https://dev.to/vevarunsharma/injecting-ai-agents-into-cicd-using-github-copilot-cli-in-github-actions-for-smart-failures-58m8)

## License

MIT
