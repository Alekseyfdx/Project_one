# Building a browser-only AI prompt builder in 2026

A static HTML/CSS/JS prompt-engineering app is genuinely buildable today with **zero backend** because OpenAI's API now serves permissive CORS, supports Structured Outputs with strict JSON schemas, and exposes a built-in `web_search` tool through the Responses API. The right architecture is a three-call pipeline — **GPT-4.1-mini for clarifying questions, GPT-4.1 with `web_search` for version verification, and GPT-5.4 (or GPT-4.1) for final spec generation** — all streamed via raw `fetch` + `ReadableStream`, with a BYOK (bring-your-own-key) UI. Total per-session cost lands around **$0.07–$0.08**. The hardest design problem isn't the API plumbing; it's the meta-prompt engineering and the post-output guidance that tells the user *how* to feed the result into Claude Code, Cursor, or ChatGPT without hitting "lost in the middle." The recommendations below are concrete enough to paste directly into your codebase.

## The three-step pipeline and model routing

Your app is fundamentally a **summarized, artifact-driven pipeline** — not a long conversation. GitHub's Spec Kit (~92k stars on GitHub) proved that compaction matters: each phase reads only the prior artifact, never the whole transcript. Drift, role-bleed, and cost all explode when you pass full conversation arrays between steps. Persist three artifacts in `localStorage` (raw description, Q/A pairs JSON, final spec) and make every API call stateless against those.

Use this routing for May 2026 model pricing:

| Step | Model | Why | Cost |
|---|---|---|---|
| **A. Clarifying questions** | `gpt-4.1-mini` ($0.40/$1.60 per M tokens) | Cheap, fast, follows JSON schema reliably; reasoning models are overkill | ~$0.0015 |
| **B. Version/best-practice verification** (optional) | `gpt-4.1` with Responses API + `web_search` tool | Built-in citations, no separate Bing key needed, server-side conversation state | ~$0.04 |
| **C. Final spec generation** | `gpt-5.4` for quality, `gpt-4.1` for cheaper output | Synthesis quality matters most here; pin fallback to `gpt-4.1` | ~$0.025–$0.04 |

Model names drift quarterly. **Pin to `gpt-4.1`, `gpt-4.1-mini`, `gpt-4.1-nano`, `gpt-4o`, `gpt-4o-mini`, `o3`, and `o4-mini`** — these have been stable since early 2025. The `gpt-5.x` flagship suffix has changed several times (5 → 5.2 → 5.4 → 5.5) and exact availability varies by region. Read the live pricing page (`developers.openai.com/api/docs/pricing`) on first load if you want to expose the current flagship to users; otherwise default safely to `gpt-4.1`.

**Avoid o-series reasoning models for this app.** They don't accept `temperature`, charge for hidden reasoning tokens (often 5-10× the visible output), and frequently take 30–120s per call — hostile UX for a wizard with three sequential calls. `gpt-4.1` matches their quality on synthesis tasks at a fraction of the cost.

## Streaming with raw fetch (no SDK)

The official `openai` npm SDK adds ~hundreds of KB, requires `dangerouslyAllowBrowser: true`, and injects `x-stainless-*` headers some proxies reject. **Use raw `fetch()` with manual SSE parsing** — it's leaner, dependency-free, and avoids the warning entirely. The Chat Completions endpoint (`/v1/chat/completions`) is the right default; reserve `/v1/responses` for the web-search step where typed events (`response.web_search_call.searching`, `response.output_text.delta`) make UX easier.

Here's the canonical streaming function — paste this verbatim:

```javascript
export async function streamChat({
  apiKey, model = "gpt-4.1-mini", messages, signal,
  onDelta = () => {}, onEvent = () => {},
  response_format, temperature = 0.7, max_tokens
}) {
  const body = {
    model, messages, stream: true, temperature,
    ...(max_tokens && { max_tokens }),
    ...(response_format && { response_format }),
    stream_options: { include_usage: true }
  };
  const res = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: { "Content-Type": "application/json", "Authorization": `Bearer ${apiKey}` },
    body: JSON.stringify(body), signal
  });
  if (!res.ok) {
    const errText = await res.text().catch(() => "");
    const err = new Error(`OpenAI ${res.status}: ${errText}`);
    err.status = res.status;
    err.retryAfter = Number(res.headers.get("retry-after")) || null;
    throw err;
  }
  const reader = res.body.getReader();
  const decoder = new TextDecoder("utf-8");
  let buffer = "", full = "";
  while (true) {
    const { value, done } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });
    const parts = buffer.split("\n\n");
    buffer = parts.pop() ?? "";
    for (const part of parts) {
      for (const line of part.split("\n")) {
        if (!line.startsWith("data:")) continue;
        const data = line.slice(5).trim();
        if (data === "" || data === "[DONE]") { if (data === "[DONE]") return full; continue; }
        try {
          const json = JSON.parse(data);
          onEvent(json);
          const delta = json.choices?.[0]?.delta?.content;
          if (delta) { full += delta; onDelta(delta); }
        } catch (e) { console.warn("Bad SSE chunk", e); }
      }
    }
  }
  return full;
}
```

Three details matter for production. First, **never write to `innerHTML` per token** — it's an O(n²) reflow disaster. Append to `textContent` of a `<pre>`, or buffer deltas and flush on `requestAnimationFrame`. Second, **abort with `AbortController`** so a Stop button cancels both the HTTP request and the reader; track a `gotAnyToken` flag so you only auto-retry streams that died before producing visible output. Third, **wrap the streaming target region with `<output role="status" aria-live="polite" aria-atomic="false" aria-relevant="additions" aria-busy="true">`** so screen readers announce additions without spamming, and flip `aria-busy="false"` on completion. The live region must exist on page load (empty) — adding it dynamically is unreliable across screen readers.

For 429s, honor the `retry-after` header when present, otherwise use exponential backoff with jitter (base 500ms, cap 30s, max 5 retries). Distinguish `insufficient_quota` from rate limits by regex on the error body — quota errors won't recover from backoff and need a clear "top up at platform.openai.com/billing" message.

## BYOK key handling done responsibly

OpenAI officially warns against client-side keys, but the BYOK pattern is widely accepted for personal tools — every Vercel AI playground, OpenRouter client, and JSFiddle-style demo relies on it. The risk is borne by the key's owner who is also the user. **Default to `sessionStorage`** (cleared on tab close); only persist to `localStorage` with explicit "Remember on this device" opt-in. Use `<input type="password" autocomplete="off" spellcheck="false">`, never log the key, validate format with a cheap probe to `/v1/models`, and ship a `<meta http-equiv="Content-Security-Policy" content="connect-src https://api.openai.com">` to limit exfiltration vectors. The most important user-facing message: **"Create a project-scoped key with a $5 monthly cap dedicated to this app, and rotate it after use."**

## Generating clarifying questions with strict JSON

The single highest-leverage prompt in your app is the question-generator system prompt. The pattern that works in production tools (Spec Kit's `/speckit.clarify`, Anthropic's prompt generator, Lovable's Chat Mode) is **a structured ambiguity-and-coverage scan across a fixed taxonomy**, then prioritization by Impact × Uncertainty, then 5–10 mixed-type questions. Critically, **the model must mirror the user's input language for questions** but generate the final spec in English (downstream coding agents like Claude Code and Cursor perform measurably better on English specs).

Force the output through Structured Outputs with `response_format: {type: "json_schema", json_schema: {strict: true, ...}}`. Strict mode requires `additionalProperties: false` at every object level and every property listed in `required` — make fields "optional" by allowing `null` (`type: ["string", "null"]`). Wrap arrays in an outer object since the root must be an object. Here's a battle-tested system prompt skeleton:

```
You are a senior product/engineering analyst. Surface the smallest set of high-impact clarifying questions about a software project before another AI generates a buildable spec.

PROCESS (silent):
1. Detect the user's input language; ask all questions in that language.
2. Run a coverage scan across: functional_scope, target_users_platform, domain_data_model,
   interaction_ux, tech_preferences, integrations, non_functional, security_privacy_compliance,
   accessibility_i18n, deployment_devops, edge_cases_failure, constraints_tradeoffs.
   Mark each Clear/Partial/Missing.
3. DROP questions where a reasonable industry default exists — note them in assumed_defaults.
4. Pick 5–10 questions, prioritized by Impact × Uncertainty.

CONSTRAINTS:
- ~60% multiple-choice (2–5 options + "Other"), ~25% yes/no, ~15% short open text.
- Each multiple-choice MUST include one option marked "recommended" with rationale.
- Never ask >1 question per category unless critical.
- Each question includes a "why_it_matters" field ≤20 words for inline help UI.

OUTPUT: strict JSON matching the schema (questions[], assumed_defaults[], detected_language,
project_type_guess, coverage_map). Total ≤1500 tokens.
```

Categories come from Spec Kit's published clarification taxonomy — this isn't theoretical, it's the actual checklist GitHub uses. The question schema should expose `id`, `category`, `type` (`single_choice`/`multi_choice`/`yes_no`/`short_text`), `question`, `why_it_matters`, and `options[]` with `label`, `implication`, and a boolean `recommended` flag. Always include a hardcoded fallback set of 7 generic questions (purpose, users, key features, constraints, success criteria, tech preferences, examples) so the wizard never dead-ends if the model fails twice.

## The compiler prompt and final spec template

Step C synthesizes raw description + Q/A pairs + (optional) version-audit results into the final spec. The empirically best structure for downstream consumption by **either** Claude or GPT is a **hybrid: XML tags as section spines for content blocks, Markdown headings for procedural sections.** Claude was specifically trained on XML tags; GPT/o-series prefer Markdown but tolerate XML. The hybrid wins for both.

The non-negotiable sections, in order: `<role>` → `<project_overview>` → `<glossary>` → `# Functional Requirements` (RFC-2119 numbered as FR-001, with GIVEN/WHEN/THEN acceptance lines) → `# Non-Functional Requirements` → `# Tech Stack (pinned to exact major.minor)` → `<file_structure>` (a literal tree) → `<data_model>` → `<api_contracts>` → `# Implementation Plan` (Spec Kit's `T001 [P] [US1]` task format with `[P]` marking parallelizable tasks) → `# Acceptance Criteria` → `# Out of Scope` → `<assumed_defaults>` → `<output_format>` → `<refusal_prevention>`.

Three details that disproportionately improve quality: **pin every dependency to exact major.minor versions** ("Next.js 15.x" not "latest Next"), since unpinned versions cause hallucination of outdated APIs; **explicitly mark out-of-scope items** to stop the "I'll also add an admin panel" drift documented in Spec Kit's `spec-driven.md`; and **end with a brutal `<output_format>` block** — "respond with a single message, full file contents in fenced blocks preceded by `// FILE: <path>`, no diffs, no truncation, no `...`" — because Cursor and Claude Code agents truncate aggressively without explicit instruction.

For the optional version-verification step (B), use the Responses API with `tools: [{type: "web_search", search_context_size: "medium", filters: {allowed_domains: ["nodejs.org", "react.dev", ...]}}]`. The built-in tool returns `url_citation` annotations and a `sources` array, costs $10–25 per 1k searches plus normal token costs, and avoids the impossibility of holding a Bing/Google key in client JS. Cap to 12 search calls per session and reject pre-release tags (alpha, beta, rc, canary, nightly).

## Wizard UX that doesn't dead-end

The dominant pattern in production prompt builders (Anthropic Console, OpenAI Playground, jourde/prompt-builder, Lovable Chat Mode) is a **horizontal numbered stepper with free back-navigation**. Allow clicking any completed step to edit; block forward jumps until the current step validates. Show "Step 3 of 6" plus a textual label always — pure dot-steppers without numbers measurably increase abandonment. Validate on blur (not keystroke) and use **soft validation** for short inputs ("This seems brief — most users describe X in 1–2 sentences. Continue anyway?"). Persist to `localStorage` keyed by a session UUID with a versioned schema (`promptBuilder.v1`); on reload, show a "Resume previous session?" modal rather than auto-loading silently.

Each clarifying question renders as a **card with consistent anatomy**: number, title, optional "Why we ask" collapsible (not a tooltip — tooltips fail on touch), the input control, and three actions on the bottom edge — **Skip**, **Not sure / let AI decide**, and **Show example**. The Skip vs Not-sure distinction matters: Skip leaves the field blank (AI infers silently), while Not-sure explicitly tells the AI to set a sensible default and surface it in `<assumed_defaults>` so the user can review. Track these separately. The Show-example button cycles through 3 realistic examples to avoid anchoring bias.

Render dynamic question types from the JSON: textarea for open description, large card-style radio groups (not tiny circles) for mutually-exclusive choice, checkbox groups for multi-select, segmented controls for ratings, and chip-input for tech preferences with autocomplete. **One question per screen on mobile, 2–3 grouped on desktop** — this comes from Brilliant's onboarding research and applies directly here.

For zero-build architecture, **Pico.css (10kb CDN) plus 150 lines of custom CSS** beats Tailwind Play CDN (~250kb runtime, slow first paint). State management can be a simple object + render function with subscribers; Alpine.js (15kb) is a reasonable upgrade if conditional rendering gets verbose. Skip HTMX — it requires server endpoints. Organize as `index.html`, `state.js`, `render.js`, `api.js`, `storage.js`, `prompts.js`, `i18n.js` — pure ES modules, no build step.

## Output presentation and the "how to send" decision tree

Present the final spec as a **collapsible section view by default** (one chevron per `<role>`, `<project_overview>`, etc.) with a toggle to "View as single block." Add per-section copy buttons plus copy-all. The copy confirmation pattern that wins both accessibility and visibility: **morph the button label** ("Copy" → "✓ Copied" for 2s) **plus** an `aria-live="polite"` toast — toasts alone are passive and easily missed; in-button feedback alone fails screen readers. Offer downloads as `.md`, `.txt`, `.json` (full session), and a "Copy as Cursor rule" preset that wraps the spec in MDC frontmatter (`---\ndescription:\nglobs:\nalwaysApply: true\n---`).

For token counting in the browser, use **`gpt-tokenizer`** (smallest pure-JS, supports both `cl100k_base` and `o200k_base`) over `js-tiktoken` (3× the bundle). Display per-section and total counts alongside **percentages of common context windows**: "≈2,400 tokens — 1.2% of Claude 200k, 1.9% of GPT-4o 128k." For Claude, OpenAI's tokenizer is an approximation; flag counts as "approximate" or hit Anthropic's `count_tokens` endpoint. Color-code: green <50% of smallest target, amber 50–80%, red >80%.

The "how to send" guidance is where most prompt builders fail. **Bake this exact decision tree** into the UI as a dynamic recommendation based on the live token count:

| Spec size | Recommendation |
|---|---|
| **<8K tokens** | Send all at once; any model works |
| **8K–30K** | Single message with TOC + sandwich critical instructions at start AND end |
| **30K–100K** | Upload as file to Claude Projects or ChatGPT Projects; or split by phase |
| **100K–200K** | Required: Claude Opus 4.6/4.7, GPT-5.5, or Gemini 2.5 Pro; upload as file |
| **>200K** | Mandatory chunking even on 1M models — quality drops; use "Chunk N of M" envelopes |

The "lost in the middle" research (Liu et al. 2024 TACL, Hsieh et al. RULER 2024) is unambiguous: attention is U-shaped, with mid-context recall degrading sharply, and **about half of models claiming 32K+ fail effective-quality thresholds at 32K**. This is why "sandwich" placement — most critical instructions at the start *and* end — outperforms single-position placement on every model tested. For chunked delivery, each chunk needs an envelope: `=== CHUNK N of M: TITLE ===` header, 2–4 sentence context recap, the section, and an explicit footer telling the model whether to act now or wait.

Ship five **target-tool export presets** as one-click buttons: **Claude.ai** (use Projects for >30K specs; type `/extra-usage` on Pro accounts to enable 1M context on Sonnet 4.6), **ChatGPT** (Projects for persistent files, switch to GPT-5 Thinking for complex specs), **Cursor** (saves as `.cursor/rules/*.mdc` with frontmatter), **Claude Code** (saves as `CLAUDE.md` at repo root), and **v0/Bolt/Lovable** (single-prompt format, strips non-UI sections). Each preset can subtly tweak formatting — XML-heavy for Claude, Markdown-heavy for GPT, MDC frontmatter for Cursor — without changing the underlying spec.

Teach prompt caching in the guidance: **place the entire spec at the top of the conversation in a system or first user message and never edit it on subsequent turns** — only append. This converts a 50K-token spec from costing $0.15 every turn to ~$0.015 every turn after the first (Anthropic explicit cache_control + 90% read discount; OpenAI automatic caching at ≥1024 tokens with `prompt_cache_key`). For your app's own internal calls, keep the system prompts byte-stable so OpenAI auto-caches the prefix at 50–75% discount.

## Conclusion

Build it as three streamed API calls (questions → optional web-search verification → spec compilation) wrapped in a six-step wizard with localStorage persistence. The model routing matters less than three architectural decisions: **stateless artifact-driven steps** (not a conversation array), **strict JSON Structured Outputs for the questions step**, and **a hybrid XML+Markdown spec template** with RFC-2119 numbered requirements, pinned tech-stack versions, explicit out-of-scope items, and a brutal `<output_format>` block. The differentiating UX feature isn't the prompt builder itself — every tool has one — it's the **token-aware delivery guidance** at the end that tells the user exactly which model and which mechanism (paste, file upload, Projects, chunked delivery) fits their spec size, with one-click formatted exports for Claude Code, Cursor, ChatGPT, and v0. Ship a hardcoded fallback question set and graceful degradation paths for every API failure mode; the apps that win in this category are the ones that never dead-end the user. Total cost per session sits around $0.07–$0.08, total bundle around 30KB without dependencies, and the entire app fits in roughly 1,500 lines of vanilla JS plus Pico.css.