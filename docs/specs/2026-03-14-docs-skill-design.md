# `/build:docs` Skill — Design Spec

**Date:** 2026-03-14
**Status:** Draft
**Plugin:** build (claude-plugin-build)
**Pipeline position:** After `/build:implement`, closes the loop

---

## 1. Purpose

A fully automated documentation generator that runs after `/build:implement`. It scans all pipeline artifacts (PRD, design doc, plan, implementation report) plus the actual codebase to determine which doc types are needed, generates them, verifies accuracy against the code, and optionally captures screenshots for user-facing docs.

The skill produces two categories of documentation:
- **Developer docs** — written to `docs/` for developers who maintain or extend the code
- **User docs** — written to `public-docs/` for end users, served via GitHub Pages using raw Jekyll-rendered markdown

---

## 2. Pipeline Context

```
/build:prd → /build:design → /build:plan → /build:implement → /build:docs
```

Each upstream skill produces artifacts that serve as inputs:

| Artifact | What it provides to docs |
|----------|-------------------------|
| PRD (`docs/prd-<slug>.md`) | Product vision, user stories, features, acceptance criteria, target users |
| Design doc (`docs/design-<slug>.md`) | Architecture, API contracts, data model, tech stack, security model |
| Plan (`docs/plan-<slug>.md`) | Task breakdown, phasing (useful for changelog/release notes context) |
| Implementation report (`docs/implement-<slug>.md`) | What actually shipped, test results, issues filed, skipped items |
| Codebase | Ground truth — actual exports, types, routes, config files, test commands |

---

## 3. Doc Types and Detection

| Doc Type | Audience | Detected When | Output Path |
|----------|----------|--------------|-------------|
| README.md | Both | Always | `README.md` |
| User Guide | End user | PRD has user-facing features | `public-docs/guide.md` |
| Configuration Reference | End user | Config files or env vars detected (`.env.example`, config dir, CLI flags) | `public-docs/configuration.md` |
| API Reference | Developer | Design doc Section 5 has endpoints, or route/controller files exist | `docs/api-reference.md` |
| Architecture Overview | Developer | Always | `docs/architecture.md` |
| Contributing Guide | Developer | Always | `docs/contributing.md` |

### Detection Logic

The orchestrator determines which docs to generate by scanning:

1. **PRD features** — if user-facing features exist → User Guide
2. **Design doc Section 5** — if API endpoints are defined → API Reference
3. **Codebase scan** — look for:
   - `.env.example`, `config/`, CLI argument parsing → Configuration Reference
   - Route files, controller files, API directories → API Reference (even if design doc is missing)
   - `package.json` scripts, `Makefile`, test config → Contributing Guide content

---

## 4. Process

### Phase 0: Discovery & Doc Plan

1. **Locate pipeline artifacts.** Search for the most recent set of pipeline docs:
   - If `$ARGUMENTS` contains a slug, look for `docs/prd-<slug>.md`, `docs/design-<slug>.md`, `docs/plan-<slug>.md`, `docs/implement-<slug>.md`
   - Otherwise, find the most recently modified `implement-*.md` and derive the slug from it
   - If no implementation report is found, warn the user that docs generation works best after `/build:implement` but can proceed with whatever artifacts exist

2. **Read all available artifacts.** Extract:
   - Product name and description (from PRD)
   - Tech stack and architecture (from design doc)
   - API endpoints and data models (from design doc)
   - What was actually built vs. planned (from implementation report)

3. **Scan the codebase.** Identify:
   - Project structure and key directories
   - Package manager and scripts (from package.json, Cargo.toml, pyproject.toml, etc.)
   - Config files and environment variables
   - Test framework and commands
   - Route/controller files and their exports
   - Existing documentation (to avoid overwriting user-maintained docs)

4. **Check for existing docs and site configuration.** Scan for:
   - Existing doc files at any of the target output paths
   - Existing GitHub Pages configuration (`.github/workflows/` with pages deployment, `_config.yml` at repo root, `docs/` already configured as Pages source in repo settings)
   - Existing static site configurations (e.g., `docusaurus.config.js`, `mkdocs.yml`, `docs/.vitepress/`)
   - Auto-generated markers from previous `/build:docs` runs (see Section 9 — Re-run Behavior)

5. **Present the doc plan** with all decisions batched into a single confirmation:
   ```
   Documentation plan for <Product Name>:

   Developer docs (→ docs/):
     ✅ API Reference — 12 endpoints from design doc [new]
     ✅ Architecture Overview — component diagram + data flow [new]
     ⚠️  Contributing Guide — exists, will overwrite [overwrite/skip?]

   User docs (→ public-docs/):
     ✅ User Guide — 5 feature sections from PRD [new]
     ⏭  Configuration Reference — no config files detected [skip]

   README.md — exists, will merge (preserve custom content) [merge/overwrite/skip?]

   GitHub Pages: will generate .github/workflows/pages.yml to serve public-docs/
   ⚠️  Found existing _config.yml at repo root — will not overwrite

   Adjust any items, or confirm to proceed.
   ```

   The user can adjust individual items (e.g., "skip contributing guide, overwrite README") in a single response. The orchestrator applies all adjustments before proceeding.

6. Wait for user confirmation before generating.

### Phase 1: Generation

Spawn parallel doc-writer agents — one per doc type. Each agent receives only the source material relevant to its doc type.

```
Agent({
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Generate <doc type>",
  prompt: <doc writer prompt from references/doc-writer-profiles.md>,
  run_in_background: true
})
```

All doc-writer agents run in parallel since they write to different files.

#### Agent Failure Handling

If a doc-writer agent fails (crashes, times out, or produces empty output):
- Note which doc type failed and the error.
- Ask the user whether to retry or skip that doc type.
- If retried, dispatch a fresh agent for that doc type only.
- Other successfully generated docs are not affected.

#### Doc Writer Prompt Structure

Each doc writer receives:
- **Role:** "You are a technical writer generating a `<doc type>` for `<product name>`."
- **Tone:** User-facing docs use approachable, task-oriented language. Developer docs use precise technical language.
- **Source material:** The specific artifacts relevant to this doc type (not all artifacts — only what's needed)
- **Codebase context:** Relevant file contents scanned by the orchestrator
- **Output path:** Where to save the file
- **Template:** A structural template from `references/doc-templates.md`

#### README.md Generation

The README is special — it bridges both audiences. Structure:
1. Project name and one-line description (from PRD)
2. Key features (from PRD, confirmed against implementation report)
3. Quick start / installation (from codebase — package.json scripts, install commands)
4. Usage overview (brief, links to user guide for details)
5. Documentation links (links to both `docs/` and `public-docs/` or the GitHub Pages URL)
6. Contributing (brief, links to contributing guide)
7. License

#### GitHub Pages Workflow

Generate `.github/workflows/pages.yml` if it doesn't exist:

```yaml
name: Deploy docs to GitHub Pages
on:
  push:
    branches: [main]
    paths: ['public-docs/**']

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: 'public-docs'
      - id: deployment
        uses: actions/deploy-pages@v4
```

Also generate `public-docs/index.md` as the landing page (links to guide, config reference, etc.) and `public-docs/_config.yml` for minimal Jekyll configuration (theme, title, description).

### Phase 2: Verification

After all doc writers complete, run two verification passes.

#### 2.1 — Accuracy Verification

Spawn a single accuracy-verification agent that cross-checks ALL generated docs against the codebase:

```
Agent({
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Verify documentation accuracy",
  prompt: <accuracy verifier prompt>,
  run_in_background: false
})
```

The accuracy verifier checks:
- **File paths** mentioned in docs actually exist
- **Code examples** compile/parse correctly (syntax check, not execution)
- **API signatures** (method, path, parameters) match actual route definitions
- **Commands** (install, test, build, run) match actual package.json scripts or Makefile targets
- **Import paths** and module names are correct
- **Config option names** match actual config file keys
- **Version numbers** and dependency names are current

For each issue found, report:
- Which doc file
- Which line/section
- What the doc says vs. what the code says
- Suggested fix

If issues are found, the orchestrator fixes them directly by editing the doc files inline (these are factual corrections — wrong paths, outdated signatures, incorrect commands — not judgment calls). After applying fixes, re-run the accuracy verifier. Max 3 iterations. If issues persist after 3 iterations, present the remaining issues to the user with the verifier's suggested fixes and let them decide.

#### 2.2 — Visual Capture (frontend projects only)

For projects with a frontend (detected by presence of UI components, `public/`, `src/pages/`, etc.), attempt to capture screenshots for the user guide.

**Before attempting visual capture, check for browser tools:**
1. Check if Chrome DevTools MCP tools are available
2. Check if Playwright is installed

**If browser tools are NOT available:**
```
⚠️  Browser tools not detected — screenshots cannot be captured for user docs.
    To enable visual capture:
    - Run /chrome to start Chrome DevTools MCP, or
    - Install Playwright: npx playwright install chromium

    Options:
    1. Fix browser tools now, then retry visual capture
    2. Proceed without screenshots (can re-run /build:docs later)
```

Wait for the user's decision. Do NOT silently skip.

**If browser tools ARE available:**
1. Start the dev server. If the dev server fails to start within 30 seconds:
   - Warn the user: "Dev server failed to start. Screenshots cannot be captured."
   - Show the error output.
   - Offer: (1) fix the issue and retry, or (2) proceed without screenshots.
2. For each major feature/flow described in the user guide:
   - Navigate to the relevant page
   - Wait for the page to be ready (network idle, no loading spinners). Timeout after 15 seconds per page.
   - Take a screenshot
   - If the screenshot appears blank or shows only a loading state, flag it: "Screenshot for `<feature>` may be incomplete — page didn't fully render."
   - Save to `public-docs/images/<feature-slug>.png`
   - Insert the image reference into the user guide markdown
3. For responsive features, also capture at mobile viewport (375x667)
4. If any pages require authentication or setup data to render, skip those screenshots and note them: "Skipped screenshot for `<feature>` — requires authentication/data setup."

### Phase 3: Review & Commit

After verification passes:

1. **Present the summary first** (do not commit yet):
   ```
   Documentation generated for <Product Name>:

   Developer docs:
     ✅ docs/api-reference.md — 12 endpoints
     ✅ docs/architecture.md — component overview with Mermaid diagram
     ✅ docs/contributing.md — dev setup, testing, conventions

   User docs:
     ✅ public-docs/index.md — landing page
     ✅ public-docs/guide.md — 5 feature sections, 3 screenshots
     ⏭  public-docs/configuration.md — skipped (no config files)

   Other:
     ✅ README.md — project overview + quick start
     ✅ .github/workflows/pages.yml — GitHub Pages deployment

   Accuracy verification: ✅ All paths, commands, and API signatures verified

   Next steps:
   - Enable GitHub Pages in repo Settings → Pages → Source: GitHub Actions
   - User docs will be served at: https://<owner>.github.io/<repo>/
   ```

2. **Ask the user to review and confirm.** The user can:
   - Review any generated doc by asking to see it
   - Request changes to specific docs
   - Confirm they're ready to commit

3. **Commit all generated docs** after user confirms:
   ```
   docs: generate project documentation [/build:docs]
   ```

---

## 5. SKILL.md Frontmatter

```yaml
---
name: docs
description: Generates project documentation from pipeline artifacts (PRD, design doc, implementation report) and the codebase. Produces developer docs (API reference, architecture, contributing guide) in docs/ and user docs (user guide, configuration reference) in public-docs/ for GitHub Pages. Run after /build:implement to close the documentation loop.
disable-model-invocation: true
---
```

---

## 6. File Structure

```
skills/docs/
├── SKILL.md                          # Orchestration logic (Phases 0-3)
└── references/
    ├── doc-writer-profiles.md        # Prompts for each doc type writer (one named section per doc type)
    ├── doc-templates.md              # Structural templates for each doc type
    └── accuracy-verifier-prompt.md   # Prompt for the accuracy verification agent
```

### doc-writer-profiles.md Structure

The file contains one named section per doc type, selected by the orchestrator based on the doc plan:

| Section Name | Used For |
|-------------|----------|
| `## README Writer` | README.md |
| `## User Guide Writer` | public-docs/guide.md |
| `## Config Reference Writer` | public-docs/configuration.md |
| `## API Reference Writer` | docs/api-reference.md |
| `## Architecture Writer` | docs/architecture.md |
| `## Contributing Guide Writer` | docs/contributing.md |

The orchestrator reads `doc-writer-profiles.md`, finds the section matching the doc type being generated, and uses that section's prompt template for the agent dispatch.

---

## 7. Conversation Style

- Be direct and operational. This is a generation phase, not a discussion phase.
- Present the doc plan clearly in Phase 0 — the user should see exactly what will be generated and where.
- During generation (Phase 1), show progress as agents complete.
- When accuracy verification catches errors, fix them silently unless the fix requires judgment (e.g., ambiguous API behavior). Only surface issues the skill can't resolve.
- For the visual capture warning, be explicit and actionable — tell the user exactly how to fix it.

---

## 8. Scope Boundaries

**In scope:**
- Generating docs from pipeline artifacts + codebase
- Accuracy verification against actual code
- Screenshot capture for user-facing docs (with browser tools)
- GitHub Pages workflow generation
- README.md generation

**Out of scope (future):**
- Static site generator setup (VitePress, Docusaurus, MkDocs)
- Changelog / release notes generation
- API doc generation from OpenAPI/Swagger specs
- Hosting on platforms other than GitHub Pages
- Internationalization of docs
- Doc versioning (multiple versions for different releases)

---

## 9. Re-run Behavior

When `/build:docs` is run again on a project that already has generated docs:

All generated doc files include a header comment marking them as auto-generated:
```markdown
<!-- build:docs:auto-generated — do not edit between markers -->
```

For README.md, generated sections are wrapped in markers:
```markdown
<!-- build:docs:start:features -->
... generated content ...
<!-- build:docs:end:features -->
```

On re-run:
- Files with auto-generated markers are updated in place (no overwrite prompt)
- README.md sections between markers are regenerated; content outside markers is preserved
- Files without markers are treated as user-maintained and get the overwrite/skip prompt in the doc plan
- New doc types that weren't generated before get the `[new]` label in the doc plan

---

## 10. Edge Cases

- **No PRD exists:** Skip user guide generation. Warn user. Still generate developer docs from codebase alone.
- **No design doc exists:** Generate architecture overview from codebase scan instead of design doc. API reference scans actual route files.
- **No implementation report exists:** Generate from whatever artifacts + codebase are available. Warn user that docs may not reflect what was actually shipped vs. planned.
- **README.md already exists and has custom content:** Offer to merge (generated sections wrapped in HTML comments for future updates) or overwrite. Never silently overwrite.
- **Project has no tests:** Contributing guide notes this and suggests adding tests rather than documenting nonexistent test commands.
- **Monorepo / multiple packages:** Detect monorepo signals (workspace config in `package.json`, multiple `Cargo.toml` files, `packages/` or `apps/` directories, Nx/Turborepo/Lerna config). Ask user which package to document, or generate docs for each with a top-level index.
