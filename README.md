# ai-code-reviewer

A GitHub App that reviews pull requests using Claude's tool use API. It posts structured inline comments organized by severity, writes a PR summary, and can be triggered conversationally via slash commands typed directly in GitHub comments. Every review shows token usage, files analyzed, and lines reviewed so teams can see exactly what happened and why.

---

## The Problem

AI coding assistants have fundamentally broken the math of code review. Output per engineer jumped roughly 60% from 2025 to 2026 as tools like Copilot and Cursor became standard. A developer with AI tools now produces 5-6 PRs per day. Human reviewers can still only handle the same number they always could. Code review has become the bottleneck; not writing, not testing, review.

The issue is not just volume. The character of the code being reviewed has changed:

**AI-generated code carries a different bug profile.** A December 2025 analysis confirmed that AI-authored code contains worse bug categories than hand-written code and lacks the intent signal that human-written code carries. Reviewers who learned to review human code patterns are less equipped to catch what AI tools produce.

**Security issues are almost invisible in manual review.** Research across four major open-source projects found that security-related comments account for less than 1% of all review comments. Memory errors, resource management issues, and race conditions account for 39% of caught security defects when they are caught, which means they require security expertise that most reviewers do not have on demand.

**Logic errors pass at scale.** Reviewers cannot hold the full call graph in working memory across a 30-file PR. Off-by-one conditions, silent error swallowing, and incorrect boolean operators are caught inconsistently depending on reviewer attention and familiarity with the codebase.

**Existing tools are expensive or limited.** CodeRabbit is $19/developer/month and is a black box. GitHub Copilot code review is tightly coupled to the GitHub ecosystem and cannot be customized. PR-Agent is the best open-source reference but is positioned as a complete product, not a codebase you are expected to understand and extend.

The gap: an open-source code review agent that is transparent about what it does, extensible via a config file, and built in a stack that is readable to any TypeScript developer on the team.

---

## Core Features

- Automatic review trigger on PR open, push, and reopen via GitHub App webhook
- Structured inline comments by severity tier: Critical, Warning, Suggestion
- Comment categories: security, logic, performance, style
- High-level PR summary posted as the first review comment
- Slash command UX: type `/review`, `/explain`, `/fix` in any PR comment to trigger on demand
- Custom ruleset via `.reviewrc` in the repo root (plain English rules, injected as system prompt context)
- Repository context pull: imports referenced type definitions and interfaces beyond the diff for cross-file reasoning
- Token usage footer on every review: "reviewed 847 lines across 12 files, 3,240 tokens used"
- Web dashboard: review history, per-repo quality trends, token cost over time
- Optional adversarial review mode: a second Claude call seeded to find what the first pass missed

---

## Tech Stack Options

### Option A: TypeScript + Next.js + GitHub App (Recommended)

All application logic is TypeScript. Next.js API routes serve as the webhook receiver. `@octokit/app` handles GitHub App authentication and webhook signature verification. The Claude API handles both review generation and structured output via tool use.

**Core packages**

| Package | Role |
|---|---|
| `next` | App framework, API routes as webhook receiver, web dashboard |
| `@octokit/app` | GitHub App auth, installation token rotation, webhook verification |
| `@octokit/rest` | GitHub REST API calls (fetch diff, post review comments) |
| `@anthropic-ai/sdk` | Claude API with streaming and tool use |
| `parse-diff` | Parses unified diff format into structured file/hunk/line objects. Critical for computing the `position` field in GitHub review comments correctly. |
| `better-sqlite3` or `Prisma + Postgres` | Review history, token usage logs, repo config |
| `shadcn/ui` + `Recharts` | Dashboard UI, review history charts |
| `zod` | Schema validation for GitHub webhook payloads and Claude tool output |

**Deployment**

- Next.js app on Vercel (one-click deploy, automatic HTTPS)
- SQLite locally, Postgres on Neon or Supabase for production
- GitHub App registered at github.com/settings/apps/new

**Why this stack:** Next.js is deployable on Vercel in under two minutes, which matters for a portfolio project that needs a live demo URL. TypeScript gives you type safety on both the GitHub API payloads and the Claude tool call schemas. This is also the stack most enterprise teams reviewing your GitHub will recognize: Octokit, Next.js, and the Anthropic SDK are all first-class maintained libraries.

---

### Option B: Python + FastAPI + GitHub Webhooks

A pure Python approach. FastAPI receives webhooks, `PyGithub` handles GitHub API calls, and the Anthropic Python SDK handles Claude. No frontend framework; the dashboard is a simple Jinja2 template or a separate React SPA.

**Core packages**

| Package | Role |
|---|---|
| `fastapi` | Webhook receiver and REST API |
| `PyGithub` | GitHub REST API calls |
| `anthropic` | Claude API, Python SDK |
| `unidiff` | Unified diff parsing. Better than parse-diff for Python: handles edge cases in large diffs. |
| `httpx` | Async HTTP for GitHub API calls |
| `sqlalchemy` + `sqlite` | Review history, token logs |

**Why this exists as an option:** Python has a more mature diff parsing library (`unidiff`) and is easier to extend if you want to add ML-adjacent features later (embedding-based similarity for detecting related bugs, local model fallback via Ollama). If you already have Python skills locked in from doc-intelligence and want to ship faster, this gets you to a working prototype in less time.

**Tradeoff:** Deployment is more involved (Render or Railway vs. Vercel one-click). TypeScript is a stronger signal for full-stack AI roles in the tri-state market. Go with Option B only if you want to build a Python CLI tool rather than a web-deployed GitHub App.

**Recommendation:** Option A. The deployment story, the TypeScript signal, and the live Vercel URL make it the better portfolio piece.

---

## Design Directions

### Direction 1: GitHub App with Web Dashboard

The primary product is a GitHub App that installs in two clicks and starts reviewing PRs automatically. The web dashboard at your deployed URL shows review history, per-repo quality trends over time, and a token cost breakdown. A settings page lets repo admins configure severity thresholds and enable or disable review categories.

This direction emphasizes the full-stack engineering story: webhook handling, GitHub API integration, structured LLM output, and a React dashboard. The live GitHub App installation is the demo. A recruiter can install it on a test repo in 30 seconds and see a review posted.

**UX flow:**

1. **Install page:** GitHub App install button, one-click OAuth, redirects to dashboard.
2. **Dashboard:** Repos table with last reviewed PR, average severity distribution, token cost this month. Click a repo to see per-PR history.
3. **PR review view:** The review posted on GitHub shows a structured summary at the top. Inline comments are grouped under severity headers. A footer shows "reviewed 12 files, 2,841 tokens."
4. **Settings:** Toggle categories on/off per repo. Set minimum severity for inline comments (suppress Suggestion-level noise for high-velocity teams). Upload or edit `.reviewrc` content via UI.

**The GitHub review format:**

```
## Review Summary

**Severity:** REQUEST CHANGES

**Files reviewed:** 8 | **Lines analyzed:** 412 | **Tokens used:** 2,841

---

### Critical (2)

**src/auth/session.ts, line 47**
The JWT secret falls back to a hardcoded string when `JWT_SECRET` is not set.
In production this means sessions signed with the fallback are accepted as valid.
Set the secret to a required env var and throw at startup if it is missing.

**src/db/queries.ts, line 112**
String interpolation in the SQL query on line 112 is exploitable via SQL injection.
Use a parameterized query: `db.query('SELECT * FROM users WHERE id = $1', [userId])`.

---

### Warning (3)
...

### Suggestion (5)
...
```

---

### Direction 2: Slash-Command-First Conversational Agent

The product is conversational rather than automatic. It does not post unsolicited reviews. Instead, developers interact with it by typing slash commands in PR comments, the same way they use `/label` or `/assign` in GitHub today.

- `/review` runs a full review of the current diff
- `/review security` runs a security-only scan
- `/explain src/auth/session.ts:47` explains what a specific line does and why it might be a problem
- `/fix src/db/queries.ts:112` opens a new PR with the suggested fix applied (Copilot-style stacked PR)
- `/ask why does this function need a mutex?` answers a free-form question about the code

This direction requires listening to `issue_comment` webhook events in addition to `pull_request` events, parsing the slash command from the comment body, and routing to the appropriate Claude prompt.

The advantage is that this UX feels like a collaborator, not a bot. It does not spam the PR with 15 Suggestion-level comments on a Friday afternoon. It is on demand, which is what most developers actually want.

**What makes this direction stronger:** PR-Agent built its following on exactly this pattern. A portfolio project that demonstrates slash-command routing, multi-turn PR conversation, and the `/fix` stacked-PR workflow is doing something that hiring managers at tool companies (Linear, GitHub, Vercel, Atlassian) will immediately understand and respect.

---

## How GitHub Review Comments Work (Critical Implementation Detail)

The most common implementation mistake: the GitHub API requires a `position` field in review comments, not a line number. `position` is the number of lines from the first `@@` hunk header in the diff, counting every line including context lines.

```
@@ -10,6 +10,8 @@    <- position 1
 context line          <- position 2
 context line          <- position 3
+new line              <- position 4  (this is the line you want to comment on)
 context line          <- position 5
```

The `parse-diff` npm package handles this correctly. Do not try to compute position manually from file line numbers; it is wrong in almost every edge case involving multi-hunk diffs.

**Review creation payload:**

```typescript
await octokit.rest.pulls.createReview({
  owner,
  repo,
  pull_number: prNumber,
  commit_id: headSha,
  body: "Overall summary here.",
  event: "REQUEST_CHANGES", // or "COMMENT" or "APPROVE"
  comments: [
    {
      path: "src/auth/session.ts",
      position: 47, // diff position, not file line number
      body: "JWT secret falls back to hardcoded string. See review summary."
    }
  ]
})
```

---

## How Claude Tool Use Powers Structured Reviews

Instead of parsing free-form LLM text to extract file paths and line numbers (fragile), define a tool that represents the exact shape of a GitHub review. Claude fills it in; you pass it directly to the GitHub API.

```typescript
const tools = [
  {
    name: "post_review",
    description: "Post a structured code review with inline comments.",
    input_schema: {
      type: "object",
      properties: {
        summary: { type: "string" },
        event: { type: "string", enum: ["APPROVE", "COMMENT", "REQUEST_CHANGES"] },
        comments: {
          type: "array",
          items: {
            type: "object",
            properties: {
              path: { type: "string" },
              position: { type: "number" },
              severity: { type: "string", enum: ["critical", "warning", "suggestion"] },
              category: { type: "string", enum: ["security", "logic", "performance", "style"] },
              body: { type: "string" }
            },
            required: ["path", "position", "severity", "category", "body"]
          }
        }
      },
      required: ["summary", "event", "comments"]
    }
  }
]
```

Claude returns `stop_reason: "tool_use"` with the tool inputs populated. Map `comments` directly to the GitHub API payload. No regex, no parsing, no hallucinated line numbers.

**For large PRs:** Filter the diff first. Skip lock files, generated files (`*.min.js`, `*.pb.go`), test fixtures, and binary files. Use the `GET /pulls/{number}/files` endpoint to get the file list sorted by change size, and batch the review into chunks that fit within the context window. Use `POST /v1/messages/count_tokens` to check before sending.

---

## Repos to Study

| Repo | What to focus on |
|---|---|
| [github.com/The-PR-Agent/pr-agent](https://github.com/The-PR-Agent/pr-agent) | The best open-source reference implementation. Apache 2.0. Study the slash command routing, the GitHub webhook handler, and the prompt templates. |
| [github.com/probot/probot](https://github.com/probot/probot) | Node.js framework for GitHub Apps. Handles auth token rotation, webhook signature verification, and event routing. Good starting point before switching to raw Octokit. |
| [github.com/octokit/octokit.js](https://github.com/octokit/octokit.js) | Official GitHub SDK. `@octokit/app` for GitHub App auth. `@octokit/rest` for API calls. This is what Probot is built on. |
| [github.com/kodustech/awesome-ai-code-review](https://github.com/kodustech/awesome-ai-code-review) | Curated list of agents, research papers, and benchmarks in the AI code review space. |
| [github.com/sourcery-ai/sourcery](https://github.com/sourcery-ai/sourcery) | Commercial tool with open architecture. Study how they structure per-file feedback and the change summary format. |
| [github.com/ejentum/agent-teams](https://github.com/ejentum/agent-teams) | Adversarial multi-model review: two agents disagree and reconcile. Reference for the devil's advocate review mode. |

---

## Reading List

### Start here

- **How CodeRabbit Actually Works** at medium.com/data-science-collective — Explains the microVM-per-PR architecture, the Codegraph cross-file dependency system, and the multi-stage linter pipeline. Understanding what the market leader does tells you where to compete and where to differentiate.
- **Code Review Is the Real Bottleneck of 2026** at dev.to/code-board — The framing article. This is the opening argument for why this project exists. Read it before writing the introduction on any application that mentions this project.
- **GitHub REST API: Pull Request Reviews** at docs.github.com — The authoritative reference for the review creation endpoint. The `position` field documentation is here. Read this before writing any webhook handler.

### GitHub App architecture

- **Deciding When to Build a GitHub App** at docs.github.com/en/apps/creating-github-apps — Explains GitHub App vs. OAuth App, permission model, and rate limit differences. The short answer: GitHub Apps are correct for bots; OAuth Apps are for user-impersonating integrations.
- **GitHub Webhooks for Pull Request Events** at docs.github.com — Subscribe to `pull_request` (opened, synchronize, reopened) and `issue_comment` (for slash commands). Payload schemas are documented here.
- **Probot docs** at probot.github.io — The fastest way to get a working GitHub App in an afternoon.
- **Automate Code Reviews with Claude API and GitHub Actions in TypeScript** at dev.to/whoffagents — Published April 2026. Closest public tutorial to this exact project.

### Claude API

- **Claude Tool Use Overview** at platform.claude.com/docs — How to define tools, the two-turn round trip, and streaming `input_json_delta` events for real-time tool call generation.
- **Anthropic TypeScript SDK** at github.com/anthropic-ai/anthropic-typescript-sdk — Focus on `client.messages.stream()` and the tool use examples.

### Competitive landscape

- **CodeRabbit Documentation** at docs.coderabbit.ai — The most complete documentation of what a production AI code reviewer does. Read the architecture section.
- **Qodo (formerly CodiumAI) PR-Agent docs** at docs.qodo.ai — Slash command reference and configuration options. This is the product yours will be compared to.
- **GitHub Copilot Code Review** at docs.github.com/en/copilot — GA April 2025, 1M preview users. Study the severity level system they shipped in May 2026; it is the right UX pattern.
- **AI code review vs. static analysis** at graphite.com — Explains precisely what semantic review catches that linters miss. Useful framing for the README and for interviews.
- **Toward Effective Secure Code Reviews (Springer, 2024)** — Academic source behind the "less than 1% of review comments are security-related" finding. Cite this when explaining why the project matters.

---

## Architecture Overview

```
[GitHub PR opened / pushed]
        |
[GitHub App Webhook -> POST /api/webhook]
  - Verify HMAC signature
  - Route: pull_request event OR issue_comment event (slash command)
        |
[Diff Fetcher]
  - GET /repos/{owner}/{repo}/pulls/{pr}/files
  - Filter: skip lock files, generated files, binaries
  - Fetch full file content for referenced imports (cross-file context)
  - Parse diff into structured hunk/line objects via parse-diff
        |
[Review Generator (Claude)]
  - Build prompt: system instructions + .reviewrc rules + diff + file context
  - Call claude-sonnet-4-6 with post_review tool definition
  - Receive structured tool_use response: summary, event, comments[]
  - Stream response to show "reviewing..." in the dashboard
        |
[GitHub Review Poster]
  - Map tool output comments[] to GitHub API payload
  - Convert diff positions from parse-diff to GitHub position values
  - POST /repos/{owner}/{repo}/pulls/{pr}/reviews
  - Append token usage footer to summary
        |
[Storage Layer]
  - Log review: repo, PR number, timestamp, severity counts, token usage
  - Store per-comment data for dashboard history
        |
[Web Dashboard (Next.js)]
  - Review history per repo
  - Severity trend charts over time
  - Token cost breakdown by repo and date
  - Settings: per-repo category toggles, .reviewrc editor
```

---

## What This Demonstrates to Employers

- GitHub App and webhook architecture, not just "I wrote a script that calls an API"
- Claude tool use for structured output, which is the pattern every production LLM app uses to avoid parsing fragile free-form text
- Understanding of code review as a systems problem: diff parsing, position computation, review comment threading
- Full-stack TypeScript: webhook receiver, database, dashboard UI, GitHub API integration
- Context window management for large PRs (batching, filtering, token counting)
- A live deployed product that hiring managers can install on a repo and watch work in real time
