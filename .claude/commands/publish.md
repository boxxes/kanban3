---
description: Security-scan, push to GitHub, deploy GitHub Pages via Actions, and update README/About
argument-hint: [github-repo-url]
allowed-tools: Bash, Read, Grep, Glob, Write, Edit, AskUserQuestion
---

## Goal

Take the current project and publish it end to end: security scan first, then push to the
GitHub repo the user names, deploy it to GitHub Pages through a GitHub Actions workflow, and
finish by writing/updating the README and the repo's "About" section (description + homepage
link pointing at the live Pages URL).

Repo URL argument: `$1` (fall back to `$ARGUMENTS` if `$1` is empty). If no repo URL was given
at all, stop and ask the user for it with AskUserQuestion — do not guess a repo name or owner.

Do the steps below **in order**. Do not skip the security scan and do not push anything before
it passes. If a step fails, stop, explain what failed in plain terms, and ask before continuing
rather than working around it silently.

## Step 1 — Security scan (must pass before anything is pushed)

Scan every file that would be added/committed (`git status --porcelain`, plus everything if this
is a fresh `git init`) for secrets and sensitive data. Do not scan `.git/` internals.

**Filenames that are automatic red flags** (flag even if content looks empty/placeholder):
`.env`, `.env.*`, `*.pem`, `*.key`, `id_rsa*`, `*.ppk`, `*.pfx`, `*.p12`, `credentials.json`,
`secrets.*`, `.git-credentials`, `*.pkcs12`.

**Content patterns to grep for across all text files about to be committed:**
- AWS access key: `AKIA[0-9A-Z]{16}`
- Private key headers: `-----BEGIN (RSA |EC |OPENSSH |DSA )?PRIVATE KEY-----`
- GitHub tokens: `gh[pousr]_[A-Za-z0-9]{20,}`, `github_pat_[A-Za-z0-9_]{20,}`
- Slack tokens: `xox[baprs]-[A-Za-z0-9-]+`
- Generic API key/secret assignment: `(api[_-]?key|secret|password|passwd|token)\s*[:=]\s*['"][^'"\s]{6,}['"]`
- Google API key: `AIza[0-9A-Za-z\-_]{35}`
- OpenAI-style key: `sk-[A-Za-z0-9]{20,}`
- Connection strings with embedded credentials: `(mongodb(\+srv)?|postgres|mysql|redis)://[^:\s]+:[^@\s]+@`
- Any hardcoded personal email address that is not clearly meant to be public (e.g. not a
  placeholder like `YOUR_EMAIL@example.com`), especially inside JS/config that talks to an
  external service (e.g. a `FORMSUBMIT_ENDPOINT`-style constant).

Use Grep across the working tree (respecting what's actually staged/trackable — skip
`node_modules`, `.git`). For each hit, report the file and line number but **never print the
matched secret value itself** in your output.

If anything is found:
- Stop. Do not run any `git push` or any GitHub API call.
- Report each finding (file:line, what kind of pattern matched) to the user.
- Ask whether they want to remove/rotate it, add it to `.gitignore`, or explicitly override and
  proceed anyway. Only continue past this step on explicit confirmation.

If nothing is found, say so briefly and continue.

## Step 2 — Push to GitHub

1. Parse `owner` and `repo` out of the provided URL (handle both
   `https://github.com/OWNER/REPO(.git)` and `git@github.com:OWNER/REPO.git` forms).
2. If this directory is not yet a git repo, `git init` and set the default branch to `main`.
3. Ensure `origin` points at the given URL (`git remote add origin <url>` or
   `git remote set-url origin <url>` if it already exists but points elsewhere — confirm with
   the user before repointing an existing origin to a different URL).
4. Stage and commit any uncommitted changes with a concise, descriptive message (skip if the
   tree is already clean and there's nothing new to commit).
5. Push with `-u origin main`.
6. Auth: if the push is rejected for authentication, prefer whatever credential helper is
   already configured (e.g. Git Credential Manager) and let it prompt interactively rather than
   asking the user to paste a token into the chat. Only fall back to asking for a token if no
   credential helper is available.

## Step 3 — GitHub Pages via GitHub Actions

1. If `.github/workflows/pages.yml` (or an equivalent Pages-deploying workflow) doesn't already
   exist, create one using the standard `actions/checkout` → `actions/configure-pages` →
   `actions/upload-pages-artifact` (path `.` unless the project has an obvious build/output dir
   — check for a package.json build step first; if this is a static-file project with no build,
   upload the repo root) → `actions/deploy-pages` pipeline, triggered on push to `main` plus
   `workflow_dispatch`, with `permissions: pages: write, id-token: write, contents: read`.
2. Commit and push the workflow file if it's new or changed.
3. Check whether the repo's Pages site is already configured
   (`GET /repos/{owner}/{repo}/pages`). If it 404s, create it with
   `POST /repos/{owner}/{repo}/pages` and `{"build_type":"workflow"}`. Use the `gh` CLI if it's
   installed and authenticated; otherwise get a token via `git credential fill` (protocol=https,
   host=github.com) for an authenticated `curl` call — never print the token, and never write it
   to a file that gets committed.
4. Wait for the triggered workflow run to complete by polling the Actions API
   (`GET /repos/{owner}/{repo}/actions/runs`) rather than guessing a fixed sleep. If it fails,
   report the failure and stop rather than proceeding to README/About updates with a broken
   deploy.
5. Once it succeeds, curl the resulting Pages URL (`https://{owner}.github.io/{repo}/`, or the
   custom domain if one is configured) and confirm it returns HTTP 200, not a 404 — a successful
   workflow run doesn't always mean the URL is actually serving yet, so verify directly and retry
   a few times with short waits if needed.

## Step 4 — README

Create `README.md` if missing, or update the existing one, so it includes at minimum:
- A title and one/two-sentence description of what the project is.
- A "Live demo" (or similarly named) section linking the verified Pages URL from Step 3.
- Anything else already true of the project worth keeping (don't invent features it doesn't
  have). If a README already exists with real content, edit it in place rather than replacing it
  wholesale — add/update the live-demo link and leave the rest intact unless it's stale.

Commit and push this change.

## Step 5 — GitHub "About" section

Update the repo's description and homepage via
`PATCH /repos/{owner}/{repo}` (`gh repo edit --description "..." --homepage "<pages-url>"` if
`gh` is available, otherwise the same token-via-`git credential fill` + `curl` approach as Step
3) so the About panel on the repo page shows a short description and links to the live Pages
site.

## Step 6 — Final report to the user

Summarize plainly:
- Security scan result (clean, or what was found and how it was resolved).
- What was pushed (branch, commit(s)).
- The live GitHub Pages URL, confirmed non-404.
- That the README and About section were updated.

Do not include any secret values, tokens, or credentials anywhere in this report.
