# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file IT PMO Kanban board (`index.html`) for a fictitious bank, used as an internal demo/training tool. All markup, CSS (`<style>`) and JS (`<script>`) live in that one file.

## Hard constraints (from the original spec — do not break)

- **Vanilla HTML/CSS/JS only.** No frameworks, libraries, build step, bundler or npm.
- **Must run from `file://`** (double-click). No server required for the board itself.
- **No external resources:** no CDN scripts, web fonts, or image files. Use the system font stack and inline SVG/Unicode for icons.
- **No persistence:** no localStorage, sessionStorage, IndexedDB or cookies. A refresh resets to the seed data on purpose, and the header's demo note says so.
- **No `alert()` / `confirm()`:** validation errors are inline, and delete uses an in-card "Delete? Yes / No" toggle.
- **No `!important`;** colours and spacing come from CSS custom properties on `:root`.
- **Branding:** a neutral "IT PMO" text wordmark and a corporate blue palette only. Don't imitate any real bank's logo or systems. "Sham" appears only in task IDs (`Sham-ITPM-####`) and the email subject.
- **The only backend is FormSubmit's AJAX endpoint.** The notification email must never be sent anywhere else.

## Commands

There is no build, lint or test tooling. To run the app:

```sh
open index.html                 # board works fully from file://
python3 -m http.server 8000     # needed for real FormSubmit delivery (it tends to reject file:// requests, which send no page address)
```

Deployment: `.github/workflows/deploy-pages.yml` has two jobs. `ci` runs on every push and PR: the syntax check, the constraint grep below, and a Gitleaks secret scan. `deploy` runs only on `main`, after `ci` passes, and publishes `index.html` to GitHub Pages. In the repo settings, Pages → Source must be set to "GitHub Actions". The live site is served over HTTPS, so FormSubmit works there. `/publish-github <repo-url>` (in `.claude/commands/`) runs the whole publish process: secret scan, README, CI/CD, Pages and the About section.

Quick checks after editing:

```sh
# JS syntax check of the inline script
awk '/<script>/{f=1;next}/<\/script>/{f=0}f' index.html > /tmp/app.js && node --check /tmp/app.js

# Constraint check: should print nothing (one CSS comment mentions a confirm "dialog", which doesn't match)
grep -nE 'localStorage|sessionStorage|indexedDB|document\.cookie|alert\(|confirm\(|!important|src="http|href="http|@import' index.html
```

The Playwright MCP server (`.mcp.json`, project scope) can drive the page in a real browser; open it at `file:///…/index.html` or through the local HTTP server. It needs Node 20 or newer: on Node 18.13 it crashes on startup with `getDefaultAutoSelectFamilyAttemptTimeout is not a function`. If your default Node is older, add a local-scope override that points at a newer `npx`.

To test behaviour without editing `index.html`, copy it to a scratch file, add a `<script>` before `</body>` that drives the UI (`.click()`, `form.requestSubmit()`, dispatching `DragEvent`s with a `DataTransfer`) and writes PASS/FAIL into a `<pre>`, then run:

```sh
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --virtual-time-budget=3000 --dump-dom file:///path/to/test.html
```

Headless Chrome enforces a minimum window width of about 500px. To check the layout below 768px, screenshot the page inside a fixed-width `<iframe>`.

## Project skills (`.claude/skills/`)

Three third-party skills were installed with `npx skills add … -a claude-code --copy` and reviewed on 2026-10-06. The sources are recorded in `skills-lock.json`. Each `SKILL.md` starts with a **"Project overrides: IT PMO Kanban"** section that takes priority over the upstream text below it:
- **`frontend-design`** (anthropics/skills, Apache-2.0): visual and UI changes within the blue tokens, system fonts and single-file rules.
- **`ui-ux-pro-max`** (nextlevelbuilder, MIT): UX and accessibility checks.
  - The offline search script is `python3 .claude/skills/ui-ux-pro-max/scripts/search.py …`, run from the repo root. The upstream `${CLAUDE_PLUGIN_ROOT}` path was rewritten.
  - Never use `--stack`, and discard web-font, icon-package and GSAP suggestions.
- **`cybersecurity-analyst`** (rysweet/amplihack): a short STRIDE pass plus the project's security invariants.
  - The upstream repo has **no licence**, so this skill is listed in `.gitignore` and kept local. Don't commit it unless the author grants permission.

`npx skills update` re-downloads the upstream files and **overwrites the overrides**. Before updating, review the new version (instructions, scripts and data), then put the override sections back.

## Architecture

**State → render loop.** A single `state` object is the source of truth: `tasks`, `filters`, `nextIdNumber`, plus `ui.pendingDeleteId` / `ui.openMoveId` for per-card transient UI. Each action (`addTask`, `moveTask`, `deleteTask`) changes `state` and then calls `renderBoard()`. Don't change card contents directly in the DOM.

- `renderBoard()` is the only function that writes card markup. The four column `<section>`s are static HTML keyed by `data-status`. Each render rewrites only each column's `.card-list` (via `innerHTML` built from `renderCard()` strings), the count badges, the filter status line, and the header summary (`renderSummary()`).
- `renderCard()` returns an HTML string. **Every task value must go through `escapeHtml()`**, including IDs and the enum values used in class names.
- Re-rendering destroys focus, so keyboard flows restore it afterwards with `focusInCard(id, selector)` or `focusColumnHeading(status)`.
- The header summary counts **all** tasks. Column badges and the cards follow the filters (shown as "shown / total" when a filter is active). `applyFilters()` is pure.
- All board interaction uses event delegation on `#board` (`data-action` buttons, plus the HTML5 drag events), so it survives re-renders. The `.dragging` / `.drag-over` class toggles are the only DOM changes made outside `renderBoard()`, because re-rendering during a drag would cancel it.

**Add Task flow.** The form lives in a left sidebar panel, not a modal. A modal would hide the "Sending…" state and the toasts, and could stop screen readers announcing them. `handleFormSubmit` runs `validateForm()` (returns `{field: message}`), then `addTask()` (optimistic, so the card appears immediately), then `resetForm()` and a success toast. Only after that does it `await notifyNewTask()` with the submit button disabled. Any failure becomes a warning toast and never touches the board. The form-field-to-element mapping is `FORM_FIELDS`, and error elements are `#err-<key>`.

**Dates** are local-calendar `YYYY-MM-DD` strings (`toISODate`/`todayISO`, never `toISOString()`), compared as strings. A task is overdue when `dueDate < todayISO()` and its status isn't Done. Seed due dates are offsets from today (`addDaysISO`), so the Overdue examples show up on any day.

**FormSubmit:** set the address in the `FORMSUBMIT_ENDPOINT` constant (CONFIG section at the top of the script). While it still contains the `YOUR_EMAIL@example.com` placeholder, `notifyNewTask` throws without sending anything. FormSubmit also needs a one-time activation: the first submission emails a confirmation link to that address, and nothing is delivered until it's clicked. FormSubmit can return HTTP 200 with `success: "false"`, which is treated as a failure.
