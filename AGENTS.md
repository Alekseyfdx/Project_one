# AGENTS.md

## Project Overview

This repository contains the design specification for **Project_one: Program Prompt Builder** — a browser-only AI prompt-engineering wizard that generates high-quality software specs through a 3-step pipeline using OpenAI's API (BYOK).

The spec document is `compass_artifact_wf-669a41f1-1e71-4b73-8d4f-b380edff8339_text_markdown (1).md`.

**Current state:** Spec/design document only — no application code, build system, tests, or dependencies exist yet.

**Target architecture (per spec):**
- Zero-build static HTML/CSS/JS app (~1,500 LOC, ~30KB bundle)
- Vanilla JS ES modules + Pico.css (CDN)
- No backend — all API calls from browser via raw `fetch()` + SSE streaming
- State persisted in `localStorage`/`sessionStorage`
- External dependency: OpenAI API (user-provided key via BYOK)

## Cursor Cloud specific instructions

- **No dependencies to install.** This is a spec-only repository with no `package.json`, `requirements.txt`, or any other dependency manifest. The update script is a no-op (`echo "No dependencies to install"`).
- **No build step.** The spec targets a zero-build architecture with pure ES modules served statically.
- **No tests exist.** There are no test files or test frameworks configured.
- **No application to run.** The application has not been built yet — only the design specification document exists.
- **When the app is eventually built**, it will be a static site servable with `python3 -m http.server` or any static file server. No Node.js or npm is required per the spec's design.
- **The spec file has a long auto-generated name.** Reference it as the "spec document" or by its full filename with proper quoting due to spaces and parentheses.
