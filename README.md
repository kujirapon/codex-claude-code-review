Claude Code Read-Only Review
A Project workflow for using Claude Code as an independent, read-only second
reviewer for the current branch diff. The goal is to catch correctness,
security, and operational issues that a single reviewer may miss while keeping
the review local-git-only and preventing Claude Code from editing files or
touching remotes.

Last verified: 2026-06-16. The local CLI checked for this update was
claude 2.1.138. The command examples below use --tools to restrict the
available tool set; current Claude Code documentation describes
--allowedTools as a pre-approval mechanism, not as the primary availability
restriction.

When to Use
Use this workflow only when the user explicitly requests a Claude Code
read-only review, for example:

/claude-code-review
/claude-code-review --wip
/claude-code-review --base develop --include-untracked
Do not trigger this workflow from generic requests such as "review my changes."
If Claude Code is unavailable, stop and tell the user. Do not replace the
second review with a Codex-only review unless the user asks for that explicitly.

Privacy Notice
This workflow sends the assembled diff prompt to Claude Code through the local
claude CLI. Do not use it on code, secrets, logs, or customer data that cannot
be shared with the Claude Code service. The workflow must never fetch from,
push to, or query a remote repository.

Supported Flags
--base <branch>: Review against a specific local branch or ref.
--wip or --working-tree: Include uncommitted working-tree changes.
--include-untracked: Include the contents of untracked files. By default,
only untracked paths are listed.
If any other flag is supplied, list the supported flags and stop.

Hard Rules
Run Claude Code read-only:
claude -p
--permission-mode plan
--tools "Read,Grep,Glob"
--disallowedTools "Edit,Write,NotebookEdit,Bash,WebFetch,WebSearch"
--disable-slash-commands
--no-session-persistence
--output-format json
Do not let Claude Code edit files, run shell commands, or use tools for
outbound network access. The only intended network activity is the Claude
Code service call made by the claude CLI itself.
Do not apply suggested fixes during this workflow. Report findings only.
Use only local git commands. Never run git fetch, git pull, git push,
git ls-remote, git remote show, git remote update, forge CLIs/APIs, or
curl/wget against repository hosts.
Claude Code reviews only the material assembled locally. Do not ask it to run
git or infer the diff.
Preserve Claude Code's raw .result text in the final output.
Do not use claude ultrareview, --remote, --from-pr, --plugin-url,
--chrome, --allow-dangerously-skip-permissions, or
--dangerously-skip-permissions for this workflow.
Workflow
1. Preflight
Run:

claude --version
If the command is missing or exits non-zero, stop with:

`claude` CLI is not installed. Install Claude Code and re-run.
If the later Claude Code dispatch fails because authentication is missing, stop
and tell the user to run claude interactively to log in or configure
ANTHROPIC_API_KEY, then re-run.

If any command example fails because an installed CLI is older or newer than the
published documentation, verify the available flags with claude --help and
preserve the same security intent: print mode, plan permission mode, only
read/search tools available, no shell, no editing, no browser/network tools, and
JSON output.

2. Resolve the Review Base
If --base <branch> was supplied, validate it locally:

git rev-parse --verify <branch>
If it does not resolve, stop. Do not silently fall back.

Otherwise auto-detect the base without contacting remotes:

Try git symbolic-ref refs/remotes/origin/HEAD --short and strip origin/.
If that fails, use the first local ref that resolves from:
origin/main, origin/master, origin/develop, origin/trunk, main,
master, develop, trunk.
If none resolve, ask the user to provide --base <branch>.
Do not add production to the fallback list. If local origin/* refs look
stale, mention that the user may want to run git fetch themselves, but do not
fetch.

3. Build the Diff
Default mode reviews committed changes only:

git diff --stat <base>...HEAD
git diff <base>...HEAD
WIP mode includes working-tree changes from the merge base:

git merge-base <base> HEAD
git diff --stat <merge-base>
git diff <merge-base>
Always collect untracked paths:

git ls-files --others --exclude-standard
Include untracked file contents only when --include-untracked is supplied.

4. Exit Early on Empty Diffs
Use the selected diffstat as the emptiness check. If it is empty, stop with:

No changes between `<current-branch>` and `<base>` (mode: `<default|wip>`).
Nothing to review.
5. Assemble the Prompt Locally
Collect:

current branch: git rev-parse --abbrev-ref HEAD
base branch/ref
mode: default or wip
diffstat
diff body
untracked path list
untracked file contents, only if requested
Write the assembled prompt to a temporary file such as
$env:TEMP\cc-review-prompt.md. Omit sections that do not apply.

Use this prompt template:

Review the diff below for branch `<current-branch>` against base `<base>`
(mode: <mode>).

You are a read-only second reviewer. Do not edit files, do not run shell
commands, do not access the network. Only return findings.

Focus areas:

Correctness & robustness
- correctness bugs
- race conditions / concurrency issues
- edge cases (boundary values, empty/null, error paths)
- data loss risks
- missing or insufficient tests

Security & data handling
- security issues (injection, deserialization, path traversal, etc.)
- auth / authz mistakes (missing checks, IDOR, privilege escalation)
- secrets or credentials exposure
- PII leakage in logs, analytics, or error messages

Operations & integration
- migration / backward-compatibility risks
- config / env handling mistakes (defaults, validation, fail-open vs fail-closed)
- external API misuse (wrong method, missing retries/timeouts, ignored errors)

Ignore:
- cosmetic refactors
- broad architecture rewrites
- pure style preferences that do not cause real problems

For each finding, return:
- severity: critical | high | medium | low | noise
- file and line, function, or symbol
- issue: one-line summary
- why it matters: 1-2 sentences
- suggested fix: concrete change, but do not apply it
- confidence: high | medium | low

--- DIFFSTAT ---
<stat>

--- DIFF ---
<diff>

--- UNTRACKED (not included in diff above) ---
<untracked-list>

--- UNTRACKED FILE CONTENTS ---
<untracked-contents>
6. Run Claude Code Read-Only
PowerShell:

Get-Content "$env:TEMP\cc-review-prompt.md" -Raw | claude -p `
  --permission-mode plan `
  --tools "Read,Grep,Glob" `
  --disallowedTools "Edit,Write,NotebookEdit,Bash,WebFetch,WebSearch" `
  --disable-slash-commands `
  --no-session-persistence `
  --output-format json
bash/zsh:

cat /tmp/cc-review-prompt.md | claude -p \
  --permission-mode plan \
  --tools "Read,Grep,Glob" \
  --disallowedTools "Edit,Write,NotebookEdit,Bash,WebFetch,WebSearch" \
  --disable-slash-commands \
  --no-session-persistence \
  --output-format json
Capture the full raw JSON. Use the .result field as the review text. The
--no-session-persistence flag reduces local retention of the diff in Claude
Code session history, but it does not change the fact that the prompt is sent to
the Claude Code service. If the CLI treats stdin unexpectedly, verify with a
tiny prompt before using this workflow on a real diff.

Final Output Format
Group findings by severity. Drop nothing silently. Move low-confidence or likely
false-positive findings into "Probably noise / Needs verification."

## Critical
- **<file>:<line>** (`<symbol>` if relevant) - <one-line issue>
  - Why: <1-2 sentences>
  - Fix: <concrete suggestion>
  - Confidence: <high|medium|low>

## High
- ...

## Medium
- ...

## Low
- ...

## Probably noise / Needs verification
- ...

## Raw Claude Code output
```text
<full unedited `.result` field>
```
If Claude Code found nothing actionable, say exactly:

Claude Code found no significant issues.
Still include the raw output for transparency.

Notes for Adjacent Workflows
This Project is review-only. Keep write-capable Claude Code delegation and
multi-agent collaboration setup as separate workflows so a read-only review
cannot accidentally become an edit session.
