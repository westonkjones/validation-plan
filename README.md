# validation-plan

A [Claude Code](https://claude.com/claude-code) plugin that turns a finished PR, branch or Jira ticket into a manual validation plan: a quick, copy-pasteable walk through the happy path of the change, published as a private page you can share.

Each plan has:

- **Summary**: the problem, how the change solves it, and links to the ticket, PRs and preceding docs (PRDs, implementation plans, TDDs, ADRs).
- **Setup**: the commands that bring up a local environment, ending in a health check.
- **Validation**: numbered happy-path steps. API steps are curl commands with request bodies and expected responses taken from the code. UI steps say what to click and what you should see.
- **Videos**: for UI changes, Playwright recordings of the happy path and up to five edge cases.

It runs only when you ask for it. Before doing anything expensive, it asks whether to run the plan against a local stack or derive it from code, and whether to record videos.

## Install

```
claude plugin marketplace add westonkjones/wes-skills
claude plugin install validation-plan@wes-skills
```

It's listed in the [wes-skills](https://github.com/westonkjones/wes-skills) marketplace.

## Use

Run the command from inside a clone of the repo the change belongs to, and pass it the change to validate:

```
/validation-plan:validation-plan 1249                                     # PR #1249 in the repo you're in
/validation-plan:validation-plan https://github.com/org/repo/pull/1249    # a PR by URL, in any repo
/validation-plan:validation-plan PROJ-123                                 # a Jira ticket; its linked PRs and docs are pulled in
/validation-plan:validation-plan                                          # the branch you have checked out, diffed against main
```

A bare number is a pull request number, read with `gh pr view` against the current repo. A Jira key needs Jira access (see Requirements). You can also ask in plain words, like "write a validation plan for this PR".

Before it runs anything, it asks two questions:

1. Whether to run the plan against a local stack and capture real responses, or derive it from the code only.
2. For UI changes, whether to record videos: the happy path plus up to five edge cases, the happy path only, or none.

It finishes with a link to the published plan and a note on anything it couldn't exercise locally.

## Requirements

- `gh`, authenticated.
- Optionally, Jira access ([`jira-cli`](https://github.com/ankitpokhrel/jira-cli) or an Atlassian connector) to pull the ticket and its linked docs.
- Node, for Playwright videos. The skill uses the repo's own Playwright when it has one.
- A Claude Code session that can publish Artifacts, to get a shareable link. Without one, the plan is written as a local HTML file.
