---
description: Scan for secrets, push the project to a GitHub repo, set up GitHub Pages via Actions, and write the README + repo About
argument-hint: <github-repo-url-or-owner/repo>
---

You are publishing this project to GitHub end-to-end. The target repo is: $ARGUMENTS

If no repo was given in $ARGUMENTS, stop and ask the user for the GitHub repo URL (e.g. `https://github.com/owner/repo.git`) or `owner/repo` before doing anything else.

Follow these steps **in this order** — the secret scan runs first, before anything is committed or pushed, because once a secret is pushed to a public GitHub repo it is exposed in history immediately (removing it afterward does not undo the exposure).

## 1. Scan for sensitive data (before any commit/push)

Scan the entire working tree (tracked and untracked files) for things that should never be pushed:
- API keys, tokens, and secrets (AWS keys, private keys, `sk-...`/`gho_...`/`ghp_...` style tokens, JWTs, generic `password=`/`secret=`/`token=` assignments)
- `.env`, `.env.*`, credential files, `*.pem`, `*.key`, service-account JSON files
- Hardcoded internal URLs, IPs, or connection strings that look credential-bearing (e.g. `mongodb://user:pass@...`)
- Any file that looks like a personal data export or real (non-demo) user data

Use `git status`/`git ls-files -o` plus `grep -rniE` (or similar) across the project, excluding `.git`, `node_modules`, and other build/dependency directories.

**If anything suspicious is found:**
- Do NOT commit or push.
- Report exactly what was found and where (file + line, redact the actual secret value in your report).
- Ask the user how to proceed (delete it, move it to an untracked/`.gitignore`d file, replace with an env var placeholder, etc.) and wait for their answer before continuing.

**If nothing is found**, say so briefly and continue to step 2.

## 2. Initialize / update git and push to GitHub

- Check `git status`. If this isn't a git repo yet, run `git init`.
- Make sure a sensible `.gitignore` exists for this project type (OS junk files, editor folders, any secret-pattern files identified in step 1) — create or extend one if missing.
- Stage and commit any uncommitted changes with a concise, accurate commit message describing what's being published.
- If there's no `origin` remote, add it using the repo URL/slug from $ARGUMENTS (normalize `owner/repo` to `https://github.com/owner/repo.git`).
- Push to the default branch. If the remote has existing unrelated history (e.g. an auto-generated README from repo creation), merge with `--allow-unrelated-histories` rather than force-pushing.
- Never force-push. If push is rejected for a reason other than unrelated histories, stop and explain the situation to the user instead of overwriting remote history.

## 3. Set up GitHub Pages via GitHub Actions

- Determine the site entry point (e.g. a root `index.html`, or a `dist`/`build` output if this project has a build step — check for a build script before assuming static root).
- Create or update `.github/workflows/deploy-pages.yml` using the standard `actions/checkout` → `actions/configure-pages` → `actions/upload-pages-artifact` → `actions/deploy-pages` flow, triggered on push to the default branch plus `workflow_dispatch`. If a build step is required, add it before the upload step.
- Commit and push the workflow file.
- Enable Pages to build from GitHub Actions: `gh api -X POST repos/<owner>/<repo>/pages -f build_type=workflow` (skip if Pages is already configured that way).
- Trigger the workflow (`gh workflow run ...` if it didn't already run from the push) and watch it to completion with `gh run watch <run-id> --exit-status`.
- Verify the deployed page returns HTTP 200 (`curl -s -o /dev/null -w "%{http_code}"` against the Pages URL) and spot-check that linked assets/routes don't 404. Fix and redeploy if anything is broken.

## 4. Create/update the README

- Write or update `README.md` at the repo root with: a clear project title and one-paragraph description, how to run/build it, key architecture notes, and a link to the live GitHub Pages site.
- Base the content on the actual project (check for an existing `CLAUDE.md` or similar docs to ground the description — don't invent features that aren't there).
- Capture a screenshot of the live site for the README: use the Playwright MCP tool (`browser_navigate` to the Pages URL, `browser_resize` to a reasonable desktop viewport, `browser_take_screenshot`) to save an image, move it into `docs/screenshot.png` (create `docs/` if needed), and embed it in the README near the top (e.g. `![Screenshot](docs/screenshot.png)`). Clean up any transient Playwright output directory (e.g. `.playwright-mcp/`) afterward — it shouldn't be committed. Skip this sub-step only if the Playwright MCP tool isn't available.
- Commit and push.

## 5. Set the repo About (description + homepage link)

- Use `gh repo edit <owner>/<repo> --description "<one-line description>" --homepage "<pages-url>"` to set the repo's About section and link it to the live Pages site.

## 6. Report back

Summarize what was done and end with the live GitHub Pages URL and the repo URL. If step 1 halted the process, make that the headline of your report instead.
