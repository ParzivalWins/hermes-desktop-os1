---
name: github-linear-janitor
description: >
  Audit GitHub PRs against Linear issues to find and repair broken tracking links.
  Scans a target repo for PRs missing Linear issue references, matches them to
  existing Linear issues by title/branch, creates new issues when none exist,
  patches PR descriptions with the correct link, syncs Linear status to match
  PR state, and posts an audit summary. Use when the user says
  "Run GitHub-Linear janitor" or when resolving janitor-labelled Linear issues.
---

# GitHub-Linear Janitor

Bidirectional audit and repair agent for GitHub PR <-> Linear issue tracking.
Ensures every merged or open PR in a target repository has a corresponding
Linear issue and vice-versa.

## When to use

- Trigger phrase: **"Run GitHub-Linear janitor"**
- When a Linear issue is labelled `janitor` and describes missing PR-to-issue links
- After a batch of PRs are merged without Linear tracking (e.g. audio manifest,
  script, or content-pipeline PRs in `ParzivalWins/StoryRun`)

## Prerequisites

- **GitHub MCP** — list PRs, read PR descriptions, update PR body
- **Linear MCP** — search issues, create issues, update issue status, post comments
- Write access to the target GitHub repository
- Linear API access to the `Build_Value` team workspace

## Inputs

| Input | Description | Default |
|---|---|---|
| `REPO` | GitHub repo in `owner/repo` format | `ParzivalWins/StoryRun` |
| `TEAM` | Linear team key | `Build_Value` |
| `PROJECT` | Linear project name (optional) | `StoryRun-WatchOS and WearOS application` |
| `PR_NUMBERS` | Comma-separated PR numbers to audit (optional; omit to scan all) | scan all open + recently merged |
| `DRY_RUN` | If true, report findings without making changes | `false` |

## Procedure

### Step 1 — Collect PRs to audit

1. If `PR_NUMBERS` is provided, fetch only those PRs from GitHub.
2. Otherwise, list all **open** PRs and all PRs **merged in the last 30 days**
   from `REPO`.
3. For each PR, extract:
   - PR number, title, branch name, merge status, created/merged date
   - Full PR body text

### Step 2 — Check for existing Linear references

For each PR, search the PR body for patterns:
- `Fixes BUI-\d+`
- `Closes BUI-\d+`
- `Supports BUI-\d+`
- `Resolves BUI-\d+`
- A bare `BUI-\d+` reference
- A `linear.app/build-value/issue/BUI-\d+` URL

Mark PRs as **linked** or **unlinked**.

### Step 3 — Match unlinked PRs to existing Linear issues

For each unlinked PR:

1. **Title match**: Search Linear issues in `TEAM` with keywords extracted from
   the PR title (e.g. for `[Audio] Phantom Protocol — Eleven Labs production
   manifest`, search `"Phantom" AND "audio manifest"`).
2. **Branch match**: Check if any Linear issue's `gitBranchName` matches the
   PR's head branch.
3. **PR-link match**: Search Linear issue descriptions and comments for the
   PR URL (`github.com/{REPO}/pull/{number}`).

If a match is found, record the `BUI-XXX` identifier. If multiple candidates
are found, prefer the one whose title is most similar to the PR title.

### Step 4 — Create Linear issues for truly orphaned PRs

For each unlinked PR with **no matching Linear issue**:

1. Determine the issue type from the PR title prefix:
   - `[Audio]` -> labels: `Feature`, category: audio production
   - `[Script]` -> labels: `script`, category: mission script
   - `[Janitor]` -> labels: `janitor`, category: maintenance
   - Other -> labels: `Feature`
2. Create a new Linear issue:
   - **Team**: `TEAM`
   - **Project**: `PROJECT` (if provided)
   - **Title**: `[{Category}] {Mission name} — {short description}`
     - Example: `[Audio] Phantom Protocol — ElevenLabs production manifest`
   - **Description** (markdown):
     ```
     ## Tracking issue for PR #{number}

     **Repository:** {REPO}
     **PR:** [{PR title}](https://github.com/{REPO}/pull/{number})
     **Status:** {Merged | Open}
     **Merged date:** {date or N/A}

     ## Summary

     {First 3 lines of PR description body}

     ---
     Created by GitHub-Linear Janitor to close tracking gap.
     ```
   - **Labels**: as determined above
   - **State**: set based on PR status (see Step 5)
3. Record the newly created `BUI-XXX` identifier.

### Step 5 — Sync Linear issue status to match PR state

For each PR-to-issue pair (both existing matches and newly created):

| PR state | Linear status action |
|---|---|
| Merged | Set issue state to **Done** |
| Open + approved reviews | Set issue state to **In Review** |
| Open + no reviews | Set issue state to **In Progress** |
| Draft | Set issue state to **In Progress** |
| Closed (not merged) | Set issue state to **Cancelled** |

Skip status sync if the Linear issue is already in the target state or a
later state (e.g. don't move Done back to In Review).

### Step 6 — Patch PR descriptions with Linear link

For each unlinked PR that now has a matched or newly created issue:

1. Prepend the following line to the PR body (before any existing content):
   ```
   Fixes BUI-{number}
   ```
2. If the PR body already contains a `## Related` or `## References` section,
   add the Linear URL there instead:
   ```
   - **Linear:** [BUI-{number}](https://linear.app/build-value/issue/BUI-{number})
   ```

### Step 7 — Post audit comment on Linear issues

For each Linear issue that was linked or created, post a comment:

```
GitHub-Linear Janitor audit completed.

- **PR:** #{number} ({Merged | Open})
- **Action taken:** {Created new issue | Linked existing issue}
- **PR description updated:** {Yes | No (dry run)}
- **Status synced to:** {Done | In Review | In Progress}
```

### Step 8 — Close the triggering janitor issue

If this skill was invoked from a specific janitor issue (e.g. `BUI-223`):

1. Post a summary comment on the janitor issue listing all repairs performed:
   ```
   ## Janitor Audit Complete

   | PR | Linear Issue | Action | Status |
   |---|---|---|---|
   | #147 | BUI-XXX | Created new issue | Done |
   | #146 | BUI-YYY | Created new issue | Done |
   ```
2. Set the janitor issue state to **Done**.

### Step 9 — Post summary

Compile and present a full audit report:

```
## GitHub-Linear Janitor Report

**Repo:** {REPO}
**PRs scanned:** {total}
**Already linked:** {count}
**Newly linked:** {count}
**Issues created:** {count}
**Status syncs performed:** {count}
**Errors:** {count or "none"}

### Repairs

| PR | Title | Linear Issue | Action |
|---|---|---|---|
| #147 | [Audio] The Last Witness ... | BUI-XXX | Created + linked |
| #146 | [Audio] Phantom Protocol ... | BUI-YYY | Created + linked |
```

## Error handling

- If a PR body update fails (permissions, branch protection), log the error
  and continue with the next PR. Do not abort the entire audit.
- If Linear API rate limits are hit, back off exponentially (1s, 2s, 4s, max 30s).
- If a Linear issue search returns ambiguous results, flag for manual review
  in the summary rather than guessing.

## StoryRun-specific conventions

These patterns are specific to `ParzivalWins/StoryRun` and the `Build_Value`
Linear team:

- **Audio manifest PRs** follow the title pattern:
  `[Audio] {Mission Name} — ElevenLabs production manifest`
- **Script PRs** follow: `[Script] {Mission Name} — 4-segment mission script`
- **Audio manifest files** live in `shared-assets/audio/{MissionName}/`
- **Audio production manifests** live in
  `shared-assets/scripts/{MissionName}/{slug}-audio-manifest.json`
- **Linear project**: `StoryRun-WatchOS and WearOS application`
- **Linear labels for audio work**: `Feature` or the mission-specific label
- When creating issues for audio PRs, reference the related script PR if
  the audio README mentions one (look for `Script PR:` in the file listing)

## Example invocation

User says: *"Run GitHub-Linear janitor on StoryRun PRs #147 and #146"*

Agent executes Steps 1-9 with:
- `REPO` = `ParzivalWins/StoryRun`
- `PR_NUMBERS` = `147,146`
- `TEAM` = `Build_Value`

Expected outcome:
- Two new Linear issues created (one for Phantom Protocol audio, one for
  The Last Witness audio)
- Both PR descriptions updated with `Fixes BUI-XXX`
- Both Linear issues set to **Done** (PRs are already merged)
- BUI-223 janitor issue closed with summary
