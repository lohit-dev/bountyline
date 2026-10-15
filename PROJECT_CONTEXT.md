# Bountyline: project context for AI agents

Read this whole file before writing code, answering questions, or proposing changes. It is the single source of truth for what we are building, why, and how. If something here conflicts with a request, ask before deviating. Items marked **VERIFY** are things the author believed to be true but could not confirm; check them against official docs before relying on them.

"Bountyline" is a placeholder name. Check that the name is free (GitHub, Devpost, domain) before committing to it everywhere.

---

## 1. Summary

Bountyline is a GitHub App plus a small web dashboard. A maintainer funds a GitHub issue through PayPal, a contributor opens a pull request that closes it, an AI reviews the pull request and scores it, and after a configurable review window the contributor is paid automatically through PayPal. Maintainers can veto during the window.

Almost everything happens inside GitHub (slash commands, labels, bot comments, check runs). The dashboard is a second front end on the same backend, used for the full list of bounties, review details, payout countdowns, private payout details, and settings.

It is being built for the **PayPal AI Hackathon on Devpost**. The submission deadline is **Nov 12, 2026, 12:00pm PST**. The build window is about four weeks starting Oct 15, 2026.

## 2. Why we are building it

**Problem.** Paying for open-source work is awkward. Maintainers who want an issue fixed have no simple, trustworthy way to attach money to it. Contributors do not know if they will be paid, or when. The hard part is trust: who decides the fix is good enough, and when is it safe to release money that cannot be taken back?

**Idea.** Put the money on the issue itself, inside GitHub where both sides already work. Use an AI reviewer to give an objective, explained score, but never let the AI be the only decision-maker: money is released only after a human-visible review window in which maintainers can veto.

**Users.**
- Maintainer or funder: creates and funds bounties, can approve or veto payouts.
- Contributor: claims a bounty by opening a PR, sets a payout address once, gets paid.
- Org admin: sees all bounties across repos in the dashboard and sets per-repo settings.

**Why this idea (hackathon fit).** The AI decides when PayPal money moves, so both PayPal and AI are central, not bolted on. It is novel (AI-judged payouts inside GitHub), has a clear real audience, and is easy to demo end to end.

## 3. Hackathon context

Source pages: `https://paypalaihackathon.devpost.com/` plus its Rules and sponsor pages.

**Requirements (hard).**
- Meaningfully use at least one PayPal developer capability (free sandbox) and AI.
- Working prototype. Judges must be able to run it: hosted demo URL or clear setup and run instructions in the repo. Mockups and non-functional demos do not count.
- Public code repository (GitHub, GitLab or Bitbucket), public and open source, with a license file visible in the repo's About section. It must contain all source, assets and instructions.
- Demo video under 3 minutes, public on YouTube, showing the project working on the device it was built for, with no third-party trademarks or copyrighted music.
- Text description and a list of tools used and how.

**Judging (five equally weighted criteria).** Technological Implementation, Design, Potential Impact, Innovation/Idea, Presentation. Stage one is a pass/fail check that the project fits the theme and really uses the required APIs. Tie-breaks compare criteria in that order. Judging may use panels and automated analysis.

**Prizes and the two-prize cap.** A project can win at most one overall prize or honorable mention, plus one sponsor prize. Special categories (Most Creative, Most Impactful, Best Demo Delivery, Best Use of PayPal + AI, Best Use of Agentic Commerce) count as honorable mentions. Overall 1st/2nd/3rd are $12,000 / $8,000 / $5,000; the special categories are $5,000 each.

**What we are targeting.**
- Sponsor prize: **Best Use of AG Grid** ($5,000 / $2,000 / $1,000 x3). The sponsor rewards polish beyond stock components: custom cell renderers, theming, layout.
- Honorable mention candidates: Best Use of PayPal + AI, Most Creative, Most Impactful, Best Demo Delivery.
- Free extras: use the **APIMatic Context Plugin** in the coding agent when building the PayPal integration, and host on **Render**. Only one sponsor prize can be won, so AG Grid is the focus.
- Not targeting: Best Use of Agentic Commerce, Channel3 and other sponsors (poor fit, extra scope).

Practical implication: one polished, end-to-end working flow beats many half-finished features.

## 4. Builder context and working agreement

- The author is a junior developer (a fresher). Comfortable with TypeScript, React and Node; has used Rust and Go but the project stack is TypeScript. Keep things simple.
- Prefer boring, well-known libraries. Do not add a library unless it removes real work. No Redis, no microservices, no Kubernetes.
- Explain non-obvious decisions in one or two sentences. Make small, reviewable commits with tests. Plan before large changes.
- Never commit secrets. Provide `.env.example`.
- When unsure about money, security or GitHub/PayPal behavior, stop and ask or mark it **VERIFY**.

## 5. Product specification

### 5.1 Core flow

1. A maintainer installs the GitHub App on a repo.
2. On an issue, a user with write access comments `/bounty 150`. The backend verifies the webhook signature and the commenter's permission, creates a bounty (`created`), adds a label, and the bot replies with a PayPal funding link.
3. The funder pays through PayPal checkout. The backend captures the payment, sets the bounty to `funded`, and the bot edits its comment and adds the label `bounty: $150`.
4. A contributor opens a PR containing `Closes #<issue>` (or links the issue in the PR's Development sidebar). The backend links the PR to the bounty (`claimed`) and posts a note. The contributor sets a payout address once through a private link; it is never posted on GitHub.
5. On PR open and on each push, the worker runs the AI review for the current head commit and posts a check run (named "Bountyline AI review") and a bot comment. Score at or above the threshold marks the PR eligible; below the threshold marks it `flagged` and a maintainer must decide.
6. When the PR is merged, the backend ensures the latest review matches the merged commit, then starts the review window and schedules the payout (merge time + window).
7. During the window a maintainer can veto. The bounty becomes `vetoed` and a refund job runs. A maintainer can also approve early, which pays immediately.
8. When the window closes with no veto, the worker re-checks state and pays the contributor through PayPal. The bounty becomes `paid`; the bot comments and labels it.
9. Bounties not claimed within the expiry period become `expired` and are refunded to the funder.

### 5.2 Slash commands (comments)

| Command | Who | Effect |
|---|---|---|
| `/bounty <amount>` | write access on issue | Create a bounty, reply with funding link |
| `/cancel-bounty` | funder or admin, before payout | Cancel and refund |
| `/veto-bounty` | write access, during the window | Veto the payout and refund |
| `/approve-bounty` | write access | Approve and pay now (also resolves `flagged`) |

The bot ignores commands from anyone without write access, replies briefly with why, and ignores comments edited after the fact unless it is a documented checkbox (see 5.4).

### 5.3 Labels (created by the bot)

`bounty: $<amount>`, `payout scheduled`, `bounty: paid`, `bounty: refunded`. Labels are applied to the issue and the PR.

### 5.4 Bot surfaces inside GitHub

1. Labels (above).
2. A single bot comment per bounty on the issue and per PR, **edited in place** as the status changes so threads stay short. Optionally includes a checkbox "Veto this payout (maintainers only)": the bot listens for the comment being edited and must verify the editor's permission.
3. A check run "Bountyline AI review" with the score and the rubric breakdown. Check runs can carry action buttons (Approve, Veto) that arrive as `check_run` webhook events with `requested_action`. **VERIFY** limits on number and length of actions.
4. Slash commands in the normal comment box.
5. Short timeline comments when the payout is scheduled and when it is paid.

Never post on GitHub: payout emails, PayPal IDs or order numbers, full model reasoning, secrets. Comments show amounts, status, score, and dates only.

### 5.5 Per-repo settings and defaults

| Setting | Default |
|---|---|
| Review window before payout | 48 hours |
| Auto-schedule score threshold | 75 |
| Bounty expiry if unclaimed | 30 days |
| Max changed lines per PR for AI review | 1,500 |
| Bounty amount limits | min $5, max $500 (sandbox) |
| Currency | USD only |
| Wait for contributor payout address after window | 14 days, then refund |

### 5.6 Rules and edge cases

- One bounty per issue. A bounty can have multiple PRs; each is reviewed. The first PR that merges and passes review is paid; other PRs are marked superseded.
- The funder cannot claim their own bounty (flag and reject in v1).
- If new commits are pushed after a review, the old review is stale. A review is valid only for its head SHA. If the PR merges with a stale review, run a fresh review before starting the window.
- Webhooks can be delivered more than once. Store the GitHub delivery ID and ignore duplicates.
- A veto after payout cannot undo the payment. The bot replies that the payout already happened. The README must state that payouts are final.
- A cancelled or expired bounty refunds the original funding payment. The platform fee may not be refundable (**VERIFY** in sandbox).
- PRs above the size cap are not reviewed automatically; the bot asks for a split or a manual maintainer decision.
- Contributors may open a PR before a bounty is funded: ignore until funded, then link.

## 6. Bounty state machine

States: `created`, `funded`, `claimed`, `reviewed`, `window`, `paid`, `flagged`, `vetoed`, `cancelled`, `expired`, `refunded`.

```
created -> funded -> claimed -> reviewed -> window -> paid
                          \-> flagged -> (approve) -> paid | (reject) -> refunded
created|funded|claimed -> cancelled -> refunded
funded|claimed -> expired -> refunded
window -> vetoed -> refunded
```

Every transition must be validated by one function, tested, written to `events`, and be idempotent. Illegal transitions throw. No code outside the state machine module may change `bounties.status`.

## 7. Architecture

Three parts, one backend:

- **GitHub**: where the code lives and where users see the bot.
- **Backend** (API + worker): all bot logic, PayPal calls, AI review. Deployed on Render.
- **Dashboard**: a separate React site using the same backend API.

```
GitHub --webhooks--> API (Hono) ----> Postgres <---- Worker (pg-boss)
   ^                  |   ^                              |     |
   |  REST as bot     |   | REST + GitHub login          |     |
   +------------------+   Dashboard (React + AG Grid)    v     v
                                                      Claude API  PayPal
```

Responsibilities:
- **API**: verify webhook signatures, parse commands, enforce permissions, write state, enqueue jobs, serve the dashboard REST API, handle PayPal return URLs and optional PayPal webhooks.
- **Worker**: run AI reviews, schedule and run payouts, expiry and refunds. All jobs are idempotent and retry-safe.
- **Dashboard**: read-only view plus approve/veto actions, calling the same backend functions as the GitHub-side actions.

At first, API and worker can be one process with a Postgres-backed queue (pg-boss). Split later only if needed.

## 8. Tech stack and repository layout

TypeScript everywhere. **Bun** is the package manager and runtime. Three repos: a parent repo with two git submodules (no pnpm workspaces, no shared `packages/` folder).

| Part | Choice |
|---|---|
| API | Bun runtime, Hono |
| GitHub App | `@octokit/app` and `@octokit/webhooks` (Probot is an alternative) |
| Database | Postgres, **Prisma ORM** and Prisma Migrate |
| Queue | pg-boss (no Redis) |
| AI | Anthropic SDK, JSON schema output, `zod` validation |
| PayPal | PayPal's official Node server SDK for orders; plain `fetch` for anything it does not cover (Payouts, **VERIFY**) |
| Dashboard | React, Vite, `ag-grid-react` (also serves the signed-out landing page at `/`) |
| Hosting | Render (web service, worker, Postgres, static site); each repo deploys on its own |

```
bountyline/                  parent repo (the Devpost submission link), no app code
  apps/
    api/                     submodule: lohit-dev/bountyline-api
      src/
        server/              webhooks, commands, REST API, PayPal return handlers
        worker/              review jobs, payout window jobs, expiry and refund jobs
        github/              Octokit helpers, command parser, comment and check templates
        paypal/              thin wrapper: create order, capture, payout, refund
        review/              diff filtering, chunking, prompts, verdict schema
        core/                state machine, types, config and env validation
      prisma/schema.prisma   data model and migrations
    dashboard/               submodule: lohit-dev/bountyline-dashboard (React + AG Grid, GitHub login)
  LICENSE  README.md  PROJECT_CONTEXT.md
```

The dashboard cannot import from the API repo. Duplicate the small shared types (`Bounty`, `Status`), or generate them from the API's OpenAPI or zod schemas. Pitfalls with Bun: run `bunx prisma generate` after installing; check that `pg-boss` and the Octokit packages run on Bun and fall back to Node for the API if one does not (**VERIFY**).

Use a permissive license (MIT or Apache-2.0); the license file must be visible in the repo About section.

## 9. Data model (Postgres)

All money stored as integer cents with a currency column. All timestamps UTC.

- `repos`: id, github_installation_id, owner, name, org_id, settings (review_window_hours, score_threshold, expiry_days, max_changed_lines).
- `bounties`: id, repo_id, issue_number, amount_cents, currency, status, funder_github_id, paypal_order_id, funded_at, expires_at, created_at.
- `submissions`: id, bounty_id, pr_number, author_github_id, head_sha, status (open, merged, superseded, closed), merged_at.
- `reviews`: id, submission_id, head_sha, score, rubric (JSON scores per criterion), summary, recommendation, model, created_at.
- `payout_accounts`: github_user_id, paypal_email (encrypted at rest), created_at.
- `payouts`: id, bounty_id, recipient_github_id, scheduled_at, state (scheduled, sent, failed, cancelled), paypal_batch_id, idempotency_key, sent_at.
- `refunds`: id, bounty_id, paypal_refund_id, reason (cancelled, expired, vetoed), state.
- `events`: id, bounty_id, type, actor, data (JSON), created_at. This is the audit log and feeds the dashboard activity list.
- `webhook_deliveries`: delivery_id (unique), received_at. Used for deduplication.

## 10. GitHub App configuration

Register the App once in GitHub's developer settings. The bot's name and avatar come from the App settings. Backend URL receives webhooks; a webhook secret is mandatory and every request is verified.

Repository permissions (**VERIFY** exact names when registering): Issues (read/write), Pull requests (read/write), Checks (read/write), Contents (read, for diffs if needed), Metadata (read).

Webhook events to subscribe to (**VERIFY**): `issue_comment`, `issues`, `pull_request`, `check_run` (for action buttons), `installation`, `installation_repositories`.

Local development: expose the local server with a webhook tunnel (smee.io or ngrok) and use a development GitHub App.

Dashboard login uses GitHub user authorization. Access is determined by the repos the signed-in user can reach through the App's installations (**VERIFY** the user-installations endpoint), and approve/veto actions re-check repo permission on the server.

## 11. PayPal integration

Use the sandbox only. Create sandbox business and personal accounts.

**Money flow.** PayPal has no true escrow. Funding is captured into the platform's PayPal account and held there; the payout is sent later with the Payouts API; refunds use the original captured payment. An authorize-then-capture flow is an alternative (void on veto) but authorizations expire, so it only fits short windows. **VERIFY on day one**: funding capture, payout to a sandbox recipient, refund of a captured order, whether fees are returned on refund, and Payouts availability in sandbox.

**Idempotency.** Every PayPal write uses an idempotency key (request ID header and a unique payout batch ID, **VERIFY** field names) stored in the `payouts` and `refunds` tables before the call. A retry must never pay twice.

**Finality.** A payout to a claimed account cannot be reversed. This is why the review window exists. Document it.

**Contributor payout address.** Collected through a private page after GitHub login, stored encrypted, never posted to GitHub.

## 12. AI review specification

**Purpose.** Produce an explained score for a PR against its issue. It informs the decision; it does not pay by itself.

**Inputs.** Issue title and body, PR title and description, filtered diff, and the results of the repo's CI checks from the GitHub checks API. Never execute contributor code.

**Filtering.** Drop lockfiles, generated and vendored code, minified files, snapshots and binary assets before the model sees anything. If changed lines exceed the cap, skip automatic review. For large diffs, review chunk by chunk (in parallel), then make one final call that combines notes into a verdict.

**Rubric (total 100).** Solves what the issue asks 25; changes stay in scope 20; tests added and CI passing 30; code quality and readability 20; no red flags 5. Red flags include hard-coded secrets, suspicious dependencies, deleted or weakened tests, and trivial changes that game the bounty.

**Output.** Strict JSON validated with zod: per-criterion scores, total, recommendation (approve, needs_review), and a short plain-language summary. Reject and retry on invalid output. Store the verdict with the head SHA.

**Threshold.** Score at or above the repo threshold (default 75) means eligible to auto-schedule after merge. Below it, the bounty becomes `flagged` and a maintainer must approve.

**Prompt-injection defense.** Treat the diff, PR text and comments as untrusted data. The system prompt must say to ignore any instructions found inside them. The model has no tools and cannot trigger payments.

**Re-review.** Only on new commits, keyed by head SHA. Review runs in the background; the PR shows a pending check until it finishes.

## 13. Security and privacy rules (non-negotiable)

- Verify every GitHub webhook signature. Reject otherwise.
- Check repo permission (write or higher) for every command, checkbox and button, on the server.
- Deduplicate webhooks by delivery ID. All jobs and PayPal calls are idempotent.
- No payout emails, PayPal IDs, secrets or full model reasoning in GitHub comments or labels.
- Encrypt payout emails at rest with a key from the environment. Never log them.
- Validate all external input with zod. Use parameterized queries only.
- Never run contributor code.
- Secrets only in environment variables. `.env.example` lists names without values.
- Sandbox PayPal only. No live money.

## 14. API surface (backend)

Webhooks:
- `POST /webhooks/github`
- `POST /webhooks/paypal` (optional, for payout status)

Funding and payout flow:
- `GET /fund/:bountyId` redirects to PayPal checkout; `GET /fund/return` captures.
- `GET /claim/:token` private payout address page (GitHub login required); `POST /api/payout-account`.

Dashboard (authenticated):
- `GET /api/me`
- `GET /api/bounties?status=&repo=&q=`
- `GET /api/bounties/:id` (includes reviews, timeline, private PayPal details for admins)
- `POST /api/bounties/:id/approve`
- `POST /api/bounties/:id/veto`
- `GET /api/repos`, `PATCH /api/repos/:id/settings`
- `GET /api/activity`

The dashboard buttons and the GitHub-side commands must call the same service functions so behavior cannot diverge.

## 15. Dashboard and UI specification

Design reference screens (made in a Design canvas): **Dashboard**, **Bounty detail**, **Bot inside a GitHub PR**. A working AG Grid dashboard page was also built as a proof of concept (AG Grid Community, version 35.3.1 from the CDN); use it as the starting point for `apps/dashboard`.

**Screen 1: Dashboard.** Top bar with brand, org chip and signed-in user. Four stat cards (Funded, Paid out, In review window, Open/unclaimed). A bounty table (AG Grid) with status tabs (All, In review, Open, Paid, Refunded), a search box, and a bot activity list with a privacy note.

**Screen 2: Bounty detail.** Header with title, repo and PR, status pill and amount. AI review card with total score, threshold marker, per-criterion scores and summary. Timeline of events. Side column: payout card with countdown, "Approve and pay now" and "Veto and refund"; private PayPal details (masked); repository settings.

**Screen 3: Bot inside a GitHub PR.** Annotated mock showing the five places the bot appears (labels, bot comment, check run, slash commands, timeline comments).

**AG Grid requirements (this is the sponsor prize).** Go beyond a stock grid:
- Custom cell renderers: two-line issue cell, status pill with due line, AI score bar, Approve/Veto buttons with a two-step veto confirmation.
- A custom theme through the Theming API (`themeQuartz.withParams`) wired to the app's CSS tokens so it works in light and dark.
- Sorting, an external status filter, and the quick-filter search; stats and tab counts update live after actions.
- AG Grid Community needs no license key. The sponsor offers a free 45-day trial license for Enterprise features; if used, load the key from an environment variable and never commit it.
- Optional: a small AG Charts panel for payouts over time.

**Design tokens.** Fonts: IBM Plex Sans (UI) and IBM Plex Mono (numbers, refs). Light palette: background `#F4F5F7`, surface `#FFFFFF`, ink `#16181D`, muted `#4A4F57`, line `#E3E5E8`, accent green `#0B6E4F`, danger `#B42318`. Status pills: review (amber) `#FFF4DB`/`#8A5300`, open (blue) `#E8EEF9`/`#1E4A99`, paid (green) `#DDF3E8`/`#0B5C40`, flagged (red) `#FDE8E6`/`#9C2A1E`, refunded (grey) `#ECEDEF`/`#4A4F57`. Provide a dark palette with the same tokens. Buttons and controls are at least 44px tall. The page must work at phone width, with the table scrolling inside its card.

## 16. Environment variables

`DATABASE_URL`, `BASE_URL`, `SESSION_SECRET`, `ENCRYPTION_KEY`, `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`, `GITHUB_WEBHOOK_SECRET`, `GITHUB_CLIENT_ID`, `GITHUB_CLIENT_SECRET`, `PAYPAL_CLIENT_ID`, `PAYPAL_CLIENT_SECRET`, `PAYPAL_ENV` (sandbox), `ANTHROPIC_API_KEY`, `AG_GRID_LICENSE_KEY` (optional).

## 17. Testing and local development

- Unit tests: state machine (every legal and illegal transition), command parser, permission checks, diff filter and chunker, review verdict schema.
- Webhook fixture tests: recorded GitHub payloads, including duplicate deliveries.
- Idempotency tests for payouts and refunds.
- A seed script that creates sample bounties for the dashboard.
- Local run: `bun install` in each submodule, `bun run dev` in `apps/api` (API and worker) and in `apps/dashboard`; a webhook tunnel forwards GitHub events to the local API.

## 18. Build plan

Dates are approximate. Submit a day early, by Nov 11.

- **Week 1 (Oct 15 to 21): the loop without AI.** PayPal sandbox spike first (day 1). Then the GitHub App with `/bounty`, funding link, capture, claim, merge, and a sandbox payout.
- **Week 2 (Oct 22 to 28): AI review.** Diff filtering and chunking, verdict schema, check run and PR comment, flagged path.
- **Week 3 (Oct 29 to Nov 4): window and dashboard.** Review window, veto and approve, expiry and refunds, GitHub login, AG Grid dashboard, bounty detail page.
- **Week 4 (Nov 5 to 11): ship.** Deploy on Render, README with run instructions and license, edge-case fixes, demo video, Devpost write-up.

## 19. Demo script and submission checklist

**Demo (under 3 minutes, filmed on a real repo).**
1. Comment `/bounty` on an issue, fund it through PayPal sandbox, see the label appear.
2. Open a PR that closes the issue.
3. Show the AI review comment and check run with the score.
4. Merge the PR, show the payout window and countdown.
5. Show the dashboard (AG Grid) and the bounty detail page.
6. Let the window close (use a short demo window) and show the payout landing in the sandbox recipient account. Briefly show a veto and refund.

**Submission checklist.**
- Public repo with LICENSE visible, run instructions, and a hosted demo URL.
- YouTube demo video under 3 minutes, public, no trademarks or copyrighted music.
- Devpost: name, 200-character pitch, thumbnail, project story, Built with tags, explanation of each tool used (PayPal, the Claude API, AG Grid, APIMatic, Render) and how.
- No secrets in the repo. Sandbox only.

## 20. Risks and open questions (VERIFY list)

1. PayPal: capture, payout, refund and fee behavior in sandbox; Payouts availability; idempotency field names.
2. GitHub: exact App permission and event names; check run action button limits; how to read PR-to-issue links from the Development sidebar; dashboard login and installation access.
3. Whether the AG Grid theme parameter names used in the prototype all exist in the pinned version.
4. Review quality on real PRs; tune the rubric and prompts with sample diffs.
5. Name availability for "Bountyline".

## 21. Non-goals

Live money, multiple currencies, cross-repo org analytics beyond the dashboard list, splitting a bounty across contributors, running contributor code, a mobile app, a marketplace listing. These are future work.

## 22. Glossary

- **Bounty**: money attached to a GitHub issue.
- **Funder**: the person who pays the bounty.
- **Contributor**: the PR author who is paid.
- **Review window**: time after merge during which maintainers may veto before payout.
- **Flagged**: AI score below the threshold; a maintainer must decide.
- **Check run**: a GitHub status entry created by the bot, shown on the PR.
