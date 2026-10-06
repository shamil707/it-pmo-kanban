---
description: Security-scan, then publish this project to a GitHub repo with README, CI/CD, GitHub Pages and the About section
argument-hint: <github-repo-url>
---

Publish this project to GitHub, using the repo URL `$ARGUMENTS`.

If `$ARGUMENTS` is empty, ask the user for the repo URL and stop. Accept `https://github.com/<owner>/<repo>`, with or without `.git`, or `git@github.com:<owner>/<repo>.git`. Work out `OWNER` and `REPO` from it. Derive the Pages URL as follows:
- `https://<owner-lowercase>.github.io/<REPO>/`
- or `https://<owner-lowercase>.github.io/` if `REPO` is `<owner>.github.io`.

Work through the steps below in order. The security scan runs **before** anything is pushed, even though the user listed it last, because a secret that reaches GitHub has to be treated as leaked. Report progress briefly after each step.

## 0. Preflight

- `git rev-parse --is-inside-work-tree`. If this isn't a git repo, run `git init -b main`.
- `command -v gh && gh auth status`. With an authenticated `gh`, steps 6–7 can be automated. Without it, those steps become manual instructions for the user.
- **Never** read credentials from the keychain, credential helpers, `~/.git-credentials`, env files or similar to get around a missing `gh`. If something needs auth you don't have, tell the user exactly what to click or run.
- Check whether the repo exists and is public: `git ls-remote <url>`, or `curl -s https://api.github.com/repos/OWNER/REPO` for public repos. If it doesn't exist and `gh` is available, ask the user before running `gh repo create OWNER/REPO --public --source . --remote origin`. Otherwise ask the user to create it **empty** (no README, licence or .gitignore) and stop until they confirm.
- GitHub Pages on a free plan needs a **public** repo. If it's private, warn the user and continue only once they confirm.

## 1. Security scan (blocking)

Scan everything that would be pushed: the tracked files, the untracked files that aren't ignored (`git ls-files --cached --others --exclude-standard`), and **every commit in history** (`git log -p --all`).

1. If `gitleaks` is installed, run `gitleaks git --no-banner -v .` (or `gitleaks detect --source . --no-banner -v` on older versions). If `trufflehog` is installed, run `trufflehog git file://. --no-update`. Treat their findings as authoritative.
2. Always also run a grep pass with `git grep -nIE --untracked -e "$PATTERN"` on the working tree, and `git log -p --all | grep -nE -e "$PATTERN"` on the history. The `-e` is required, because the private-key pattern starts with `-` and would otherwise be read as an option. Pipe a known fake key through the patterns once, to prove they match. Look for:
   - private keys: `-----BEGIN [A-Z ]*PRIVATE KEY-----`
   - cloud and SaaS tokens: `AKIA[0-9A-Z]{16}`, `gh[pousr]_[A-Za-z0-9]{36,}`, `github_pat_[A-Za-z0-9_]{20,}`, `xox[abposr]-[A-Za-z0-9-]{10,}`, `AIza[0-9A-Za-z_-]{35}`, `sk_live_[0-9a-zA-Z]{20,}`, `sk-ant-[A-Za-z0-9_-]{20,}`, `sk-[A-Za-z0-9]{32,}`, and JWTs `eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}\.`
   - generic assignments: `(password|passwd|secret|api[_-]?key|access[_-]?token|auth[_-]?token|client[_-]?secret)["']?\s*[:=]\s*["'][^"']{8,}["']` (case-insensitive)
   - credentials in URLs: `[a-z]+://[^/\s:@]+:[^/\s@]+@`
3. Flag files that should never be published:
   - `.env*` (except `.env.example`), `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `*.keystore`
   - `credentials*.json`, `service-account*.json`, `.npmrc`/`.pypirc` containing tokens
   - `.claude/settings.local.json`, `*.log`, `.DS_Store`
4. **Email addresses and personal data:** list every email address found, e.g. one in `FORMSUBMIT_ENDPOINT`. Anything other than a placeholder like `YOUR_EMAIL@example.com` becomes public once pushed, so ask the user before continuing.
5. Make sure a `.gitignore` exists and covers the file types in point 3. Create it or add to it, and commit it with the other changes.

**If anything real is found:**
- Stop. Don't push.
- Show each finding as `file:line`, and mask the secret value; never print it in full.
- If the secret is only in the working tree, remove it, replace it with a config placeholder or environment variable, and get the user's OK.
- If it's in **git history**, tell the user that the history needs rewriting (e.g. `git filter-repo`) and that the secret must be rotated. Don't rewrite history without explicit approval.

Document false positives, such as placeholder values or docs that only mention a pattern, and continue.

## 2. README.md

Create or update `README.md` from what's actually in the repo (code, `CLAUDE.md`, existing README). Don't invent features. If a README already exists, keep the sections the user wrote and only update or add the following:
- project title and a one-paragraph description
- **Live demo** link (the Pages URL)
- features
- how to run it locally (for this project: open `index.html`, or `python3 -m http.server` when FormSubmit email is needed)
- configuration, e.g. `FORMSUBMIT_ENDPOINT` and the FormSubmit one-time activation step
- how deployment works (the CI/CD workflow)
- the tech constraints that matter to contributors (vanilla JS, single file, no persistence, no external resources)

## 2b. Screenshots for the README (Playwright MCP)

Capture fresh screenshots with the project's Playwright MCP server (`.mcp.json`), then embed them in the README.

1. Make sure the `mcp__playwright__*` tools are available.
   - If the server was only just added, its tools won't load until the next session. Tell the user to restart Claude Code and re-run this command, or skip this step and say so in the final report.
   - The server needs Node 20 or newer. If it fails to start on an older default Node, see the Playwright note in `CLAUDE.md`.
2. Serve the app locally so the screenshot matches the commit being pushed, rather than the old live site: run `python3 -m http.server 8765 --bind 127.0.0.1` in the background from the repo root.
3. Use the Playwright tools to:
   - `browser_resize` to 1440×900, then `browser_navigate` to `http://127.0.0.1:8765/index.html`
   - `browser_take_screenshot` with `filename: "docs/screenshots/board-desktop.png"`
   - `browser_resize` to 390×844, then `browser_take_screenshot` with `filename: "docs/screenshots/board-mobile.png"`

   Relative filenames are resolved against the server's working directory, which is the repo root. Check the files ended up in `docs/screenshots/` and not loose in the repo root; move them if needed.
4. Stop the HTTP server and `browser_close`.
5. Open each image with Read and check it shows the seeded board fully rendered, with no error page or half-loaded layout. Retake it if not.
6. Under the README's **Live demo** line, add or refresh a `## Screenshots` section. Use the desktop image as a Markdown image and the mobile image as `<img ... width="260">`, both with descriptive alt text, and keep the existing file names so the links stay valid.
7. Screenshots go through the same security scan as everything else. Make sure they show only demo data: no real names, emails, tokens or internal URLs.

## 3. GitHub Actions CI/CD

Create or update `.github/workflows/deploy-pages.yml`. If there's an existing workflow, edit it instead of adding a duplicate. It needs:
- **Triggers:** `push` and `pull_request` to `main`, plus `workflow_dispatch`.
- **`ci` job**, on every trigger. For this project it should:
  - syntax-check the inline script (extract it with `awk`, then `node --check` using `actions/setup-node`)
  - fail if the file contains `localStorage|sessionStorage|indexedDB|document\.cookie|alert\(|confirm\(|!important` or external `src="http`/`href="http` references
  - run a secret scan (`gitleaks/gitleaks-action` with `fetch-depth: 0`, plus `env: GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}`)
- **`deploy` job:**
  - `needs: ci`, and only runs on `push` to `main` or `workflow_dispatch`
  - copies only the publishable files (for this project, `index.html` plus an empty `.nojekyll`) into `_site`
  - uses `actions/configure-pages`, `actions/upload-pages-artifact`, `actions/deploy-pages` with `permissions: pages: write, id-token: write`, the `github-pages` environment, and a `concurrency: pages` group
- Use the latest major versions of the actions. If you're unsure which version is current, check the action's releases page rather than guessing.
- Validate the YAML locally (e.g. `ruby -ryaml -e 'YAML.load_file(ARGV[0])' <file>`) before committing.

## 4. Commit and push

- Commit the README, `docs/screenshots/`, the workflow, `.gitignore` and any scan fixes, with clear messages. Follow this session's commit attribution rules.
- Set `origin` to the provided URL. If `origin` already points somewhere else, ask before changing it.
- Push with `git push -u origin main`. **Never force-push.** If the remote has commits you don't have (e.g. a README created on GitHub), stop and ask how the user wants to combine them.

## 5. GitHub Pages

The Pages source must be **GitHub Actions**.
- **With `gh`:** run `gh api repos/OWNER/REPO/pages --jq .build_type`. If that 404s, run `gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`. If it returns `legacy`, run `gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow`.
- **Without `gh`:** tell the user to open `https://github.com/OWNER/REPO/settings/pages` and set **Build and deployment → Source → GitHub Actions**. Then wait for confirmation, or poll `https://api.github.com/repos/OWNER/REPO/pages` every 90s or more. Unauthenticated requests are limited to 60 an hour, and an empty or unparseable response is *not* a change.
- If the first workflow run failed because Pages wasn't enabled, re-run it (`gh run rerun <id>` or the **Re-run all jobs** button). Without `gh`, ask the user before pushing an empty commit to trigger it.

## 6. Repo About section

Set the description, the website (the Pages URL) and topics.
- **With `gh`:** `gh repo edit OWNER/REPO --description "<one-line summary from the README>" --homepage "<pages-url>" --add-topic <topic> ...`. Use relevant topics, e.g. `kanban`, `project-management`, `vanilla-javascript`, `github-pages`.
- **Without `gh`:** give the user the exact description, website URL and topics to paste in via the gear icon next to **About** on the repo page.

## 7. Verify and report

- Watch the latest Actions run to completion: `gh run watch`, or poll `https://api.github.com/repos/OWNER/REPO/actions/runs?per_page=1`. If it fails, fetch the job steps and annotations, explain the cause, and fix it if it's in the repo.
- Confirm the live site returns HTTP 200: `curl -s -o /dev/null -w '%{http_code}' <pages-url>`. Only give the user the link once it actually loads.
- Final report:
  - repo URL
  - live Pages URL (verified)
  - the Actions run link and its result
  - a security scan summary (tools used, findings, false positives)
  - files created or changed
  - any manual steps still needed
