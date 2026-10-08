---
name: validation-plan
description: Build a manual validation plan for an implemented feature or code change and publish it as a private shareable page — a short summary with links to the Jira ticket, PR, and design docs, copy-paste local setup commands, happy-path curl requests derived from the code with expected responses, UI click-through steps, and optional Playwright videos of the UI flow. Use when the user explicitly asks for a validation plan, a manual test plan, a QA walkthrough, or a way to "thumb through" / manually verify a PR, branch, or Jira ticket. Do not use proactively after finishing a change; this is only for higher-impact work the user asks to have validated.
---

# Validation plan

A validation plan lets someone manually walk the happy path of a finished change in a few
minutes. Edge cases and gotchas were already handled while the change was built; the plan is
a quick, trustworthy pass to confirm it works end to end. Everything in it should be
copy-pasteable and grounded in the code, because a plan with a guessed request body wastes
the reader's time the moment it 400s.

Only run this when the user asks for it.

## 1. Identify the change

The user may give a PR number or URL, a Jira key, or nothing (use the current branch). Gather
from whichever sources are available, and follow links between them:

- **PR**: `gh pr view <n> --json title,body,url,headRefName,baseRefName,files,mergedAt,mergeCommit`
  and `gh pr diff <n>`. Read the code from a detached `git worktree add` in the scratchpad
  (and run it there only once the user picks *Run and capture* in step 3) rather than switching branches in the user's clones, which often have work in
  progress. Check out the PR head if it's open, or `mergeCommit.oid` if it's merged, so the
  plan reflects that change and not whatever later landed on `origin/main`.
- **Jira**: use whatever Jira access the session has, such as `jira issue view <KEY> --plain`
  with `jira-cli` or an Atlassian connector, and follow the user's CLAUDE.md if it names
  one. Pull acceptance criteria and linked issues.
- **Current branch**: `git diff $(git merge-base HEAD origin/main)...HEAD` plus
  `git log --oneline` for the range. Look for a ticket key in the branch name or commits, and
  `gh pr list --head <branch>` for an open PR.

Then find preceding docs: PRDs, implementation plans, TDDs, ADRs. Look for links in the PR
body and Jira ticket, and search the repo's `docs/` (and similar) for the ticket key or feature
name. Collect every relevant ticket, PR, and doc link; the summary must list them.

## 2. Understand what to exercise

Read the changed code, not just the diff hunks, to learn:

- **Entry points**: new or changed routes (method, path, path/query params), message handlers,
  CLI commands, UI routes and components.
- **Request shape**: build bodies from the actual source of truth — DTOs/structs/schemas,
  validation rules (required fields, enums, formats, length limits), OpenAPI specs, and
  existing tests or fixtures. A test fixture that already passes validation is the best
  starting point. Use values that satisfy every validator.
- **Headers and auth**: which middleware runs on the route, what auth scheme it expects, and
  any required headers (content type, project/environment ids, idempotency keys).
- **Responses**: success status code and response body shape from the handler and its
  serializer. Note side effects the reader can observe (a DB row, a follow-up GET, an event).
- **Local environment**: how this repo runs locally — `README`, `Makefile`, `docker-compose*`,
  `.env.example`, `configs/dev*`, `package.json` scripts, `CLAUDE.md`, and any local-stack
  CLI or upstream stubs under `tools/` or `scripts/` (they usually beat hand-rolled docker
  commands). Check memory for environment conventions the user has already established
  (e.g. testing against a locally created shop rather than a shared staging project).
- **Data the change needs**: whether the local stack can actually produce the state the
  change renders or handles. Stubs and fixtures often predate the feature. If they lack it,
  plan a small patch to the stubs or fixtures (applied in the scratch worktree, never
  committed) and ship that patch in the plan's setup section so the reader gets the same
  state.
- **UI changes**: whether the diff touches frontend code the reader would click through. If
  so, map the happy-path flow and list the edge cases the code explicitly handles (empty
  states, validation messages, permission branches, error banners). Rank them by impact and
  keep the top 5.

## 3. Ask before doing anything expensive

Use the AskUserQuestion tool for every question in this skill — never ask in plain text. Ask
these together in one call once you know what the change involves:

1. **Dry run or derive?** Offer: *Run and capture* (spin up the local env, run setup and each
   request, and paste real responses as the expected output), or *Derive from code only*. Note
   in the option description roughly what running involves (e.g. "starts docker compose and
   creates a local shop").
2. **Videos** — only if there are UI changes. Offer: *Happy path + edge cases* (list the up to
   5 edge cases you'd record in the option description), *Happy path only*, or *No videos*.

If the change has no UI component, skip the second question.

### If running

Run only against local services. Never send mutating requests to shared staging or
production; if the only way to exercise something is a shared environment, say so in the plan
and derive that step instead. When a command or request fails, fix the plan (wrong field, missing
header, missing setup step) and rerun rather than writing down something you know is broken.
If it still fails because of the change itself, stop and tell the user — that's a bug, not a
plan problem. Scrub tokens, secrets, and personal data from captured responses before they go
in the plan.

### If deriving

Write expected responses from the handler code, and show the shape with representative values.
Mark generated ids and timestamps as such (e.g. `"id": "<uuid>"`).

## 4. Record videos (if requested)

Read `references/playwright-recording.md` for the recording setup and script template. Record
one video for the happy path and one per chosen edge case (5 at most), each as its own short
clip with a caption naming the scenario. Run the UI against the local environment from the
setup section.

## 5. Write and publish the page

Publish the plan as a private Artifact page. Load `artifact-design` first. If this session
has no Artifact tool, write the same page as a local HTML file with the videos beside it,
and give the user its path instead of a link. Publish videos as
supporting files next to the page (the Artifact `files` map, e.g. `videos/happy-path.webm`)
and embed each one in its scenario with a relative `src`. This needs no runtime capability,
so the page stays shareable. Long text such as a setup patch goes in a collapsed
`<details>` block, HTML-escaped.

**Title**: the Jira ticket followed by a plain title, e.g.
`ULO-12057: Validate recording the webshop visit`. With no ticket, lead with the PR, e.g.
`PR #482: Validate media uploads`. No invented or clever titles.

Write the page as current information: what the change is and how to verify it, with no
history of the plan itself.

### Page structure

**Summary** — a short intro, not the focus. Two to four sentences: the problem, and how the
change solves it. Then a links list: Jira ticket(s), PR(s), and every preceding doc (PRD,
implementation plan, TDD, ADR). If a request flows through several components, add one small
labeled diagram showing the path the reader is about to exercise; skip it if a sentence says
it faster.

**Setup** — everything needed to get a local environment ready, in order, as copy-pasteable
blocks: checkout, dependency install, docker/compose commands, migrations or seed data,
starting services, and exporting environment variables. Use env var placeholders for tokens
and secrets (`$ACCESS_TOKEN`, `$PROJECT_ID`) and show how to obtain each one locally right
where it's exported. End with a quick health check that proves the stack is up.

**Validation** — the happy path as numbered steps the reader follows in order. For each API
step:

- a one-line purpose
- a curl command with method, URL, headers, and JSON body inline, using the env vars from
  setup
- the expected status code and response body
- what else to check, if anything (a follow-up GET, a log line, a DB row)

Chain steps when one response feeds the next, and show how to capture the value, e.g.
`export SHOP_ID=$(curl ... | jq -r .id)`.

For UI steps, give the URL to open and the clicks and inputs in order, with what the reader
should see after each meaningful action. Embed the happy-path video beside these steps.

**Edge case videos** — only if recorded. One labeled video per edge case with a single line on
what it shows. Keep edge cases out of the written validation steps; the written plan stays on
the happy path.

**Teardown** — commands to stop services and clean up anything the plan created, if needed.

After publishing, give the user the link and one line on what was verified (run and captured,
or derived from code), plus anything that couldn't be exercised locally.
