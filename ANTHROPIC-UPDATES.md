# Anthropic Updates - Digest for Milica

**Last refreshed:** 2026-05-12
**Next refresh due:** 2026-05-19
**Sources checked:** 6 of 8 (Reddit and TypeScript SDK skipped this run, will include next refresh)

---

## What's new

### Claude Sonnet 4.6 is now Anthropic's recommended balanced model
**Type:** Model
**Affects SALOS:** Yes - Milica currently uses `claude-sonnet-4-20250514` everywhere. Sonnet 4.6 has improved agentic search performance and uses fewer tokens for equivalent work. 1M token context window is now standard with no beta header.
**Source:** https://platform.claude.com/docs/en/release-notes/overview

The practical implication for SALOS: lower input token cost on long context calls (lead research with full org config, meeting prep with conversation history, proposal generation), and better agentic behavior for multi-step research.

**What to do:** Plan a single PR to change `claude-sonnet-4-20250514` to `claude-sonnet-4-6` in the `/api/ai/chat` proxy and in any other places the model string is referenced. Test with one full research call before merging.

### Claude Opus 4.7 released
**Type:** Model
**Affects SALOS:** Partially - too expensive to use as the default but worth considering as an "advisor" model for the hardest reasoning steps (proposal generation, deep prospect research).
**Source:** https://www.anthropic.com/news

Stronger performance on coding, agents, vision, and multi-step tasks. For SALOS, the natural fit is the new Advisor pattern below: Sonnet as executor, Opus consulted for hard sub-problems.

### Web search and programmatic tool calling are now GA
**Type:** API
**Affects SALOS:** Yes - any code using these features can drop beta headers.
**Source:** https://platform.claude.com/docs/en/release-notes/overview

Remove beta headers from API calls. Both tools are stable.

### API code execution is now free when used with web search or web fetch
**Type:** API / Pricing
**Affects SALOS:** Yes - direct cost reduction for any lead enrichment flow using web search + code execution to parse results.
**Source:** https://platform.claude.com/docs/en/release-notes/overview

Web search and web fetch now also support "dynamic filtering" - code execution filters results before they reach the context window, reducing token cost. Worth investigating for SALOS lead research.

### Advisor tool in beta
**Type:** API / Pattern
**Affects SALOS:** Worth piloting - Sonnet executor + Opus advisor in a single Messages API call. Lets the cheaper model consult the smarter one only for hard reasoning, then continue.
**Source:** https://platform.claude.com/docs/en/release-notes/overview

Beta header: `anthropic-beta: advisor-tool-2026-03-01`. Tool name: `advisor_20260301`. Anthropic published benchmarks showing meaningful gains on SWE-bench Multilingual. For SALOS, the cleanest first test is proposal generation: Sonnet drafts, Opus reviews logic on hard sections, Sonnet finalizes.

### Claude Code: `/goal` command (v2.1.x)
**Type:** Claude Code
**Affects SALOS:** Yes - directly improves Milica's primary editing workflow.
**Source:** https://github.com/anthropics/claude-code/releases

Set a completion condition and Claude keeps working across turns until met. Live overlay shows elapsed time, turns, and tokens. Works in interactive, `-p`, and Remote Control. This is the right tool for tasks like "fix all org_id filtering issues across the API endpoints" - one goal, multi-turn execution.

### Claude Code: Agent view (Research Preview)
**Type:** Claude Code
**Affects SALOS:** Yes - useful when Milica runs multiple Claude Code sessions in parallel.
**Source:** https://github.com/anthropics/claude-code/releases

Run `claude agents` to see every session (running, blocked, done) in one list. Useful when she's running parallel sessions on different SALOS features.

### Claude Code: `/recap` for returning to a session
**Type:** Claude Code
**Affects SALOS:** Yes - useful after multitasking gaps.
**Source:** https://code.claude.com/docs/en/changelog

Configurable in `/config`, manually invocable. Generates a self-contained summary when she returns to a session after time away. Fits her working style.

### MCP servers now auto-retry on startup failures
**Type:** MCP / Claude Code
**Affects SALOS:** Indirectly - improves reliability of her connected tools (Figma, Slack, Google Drive, etc).
**Source:** https://code.claude.com/docs/en/changelog

Transient startup errors now retry up to 3 times instead of leaving the server disconnected. Less manual reconnection.

### Built-in slash command discovery via Skill tool
**Type:** Claude Code
**Affects SALOS:** Yes - relevant for skill authoring.
**Source:** https://claudefa.st/blog/guide/changelog

Claude can now discover and invoke built-in commands like `/init`, `/review`, `/security-review` via the Skill tool. This means custom skills can chain into built-in commands.

### Claude Managed Agents: Outcomes, Dreaming, Multi-agent orchestration (public beta)
**Type:** Platform
**Affects SALOS:** Future-relevant. Not yet a fit but worth tracking - "Dreaming" lets agents review past sessions overnight and create new memory files. This pattern maps directly onto how the NEBOS Cradle is supposed to work.
**Source:** https://simonwillison.net/2026/May/6/code-w-claude-2026/

Beta header: `managed-agents-2026-04-01`. Outcomes lets you set what success looks like so Claude iterates until it gets there. Multi-agent orchestration for fleets of cooperating agents. For Milica: keep an eye on Dreaming as a potential implementation pattern for the NEBOS Knowledge Agent.

### Max tokens cap raised to 300k on Message Batches API
**Type:** API
**Affects SALOS:** Possibly - useful for long-form proposal generation in batch.
**Source:** https://platform.claude.com/docs/en/release-notes/overview

Beta header: `output-300k-2026-03-24`. Only on Opus 4.6 and Sonnet 4.6 batches.

### Claude Design (Anthropic Labs)
**Type:** Product
**Affects SALOS:** No, but relevant for TR3I marketing work.
**Source:** https://www.anthropic.com/news (Apr 17, 2026)

A new product for collaborative visual work: designs, prototypes, slides, one-pagers. Worth a 30-minute look for her TR3I marketing role.

---

## Active patterns

### Pattern: Assistant prefill for JSON extraction
**Use when:** Any API call that needs to return structured JSON.
**Implementation:** Send `{role: "assistant", content: "{"}` as the last message. Parse with `"{" + rawText` and trim with `lastIndexOf("}") + 1`.
**Last confirmed current:** 2026-05-12

This is Milica's existing pattern. Still the most reliable approach for JSON output.

### Pattern: Prompt caching for repeated context
**Use when:** Same context block appears at the start of many calls (e.g. SALOS org config, knowledge base extracts).
**Implementation:** Mark the cacheable block with `cache_control: {type: "ephemeral"}` in the API request. Cached blocks persist for 5 minutes.
**Last confirmed current:** 2026-05-12

Milica has not yet applied this to SALOS. The org config block (`getOrgProfilePromptBlock()`) prepended to every prompt is a perfect candidate. Estimated impact: significant input token reduction on high-volume operations (research, meeting prep, sequence message generation).

### Pattern: Server-side API proxy
**Use when:** Any production API call from a browser-based app.
**Implementation:** Route all API calls through a backend proxy (`/api/ai/chat`). Never expose `ANTHROPIC_API_KEY` to the client.
**Last confirmed current:** 2026-05-12

Already enforced in SALOS. No changes needed.

### Pattern: `/goal` for persistent multi-turn Claude Code tasks
**Use when:** A task that needs Claude to keep working across multiple turns until a clear completion condition is met (e.g. "fix all 27 endpoints lacking org_id filtering").
**Implementation:** In Claude Code, `/goal "completion condition here"`. Works in `-p` mode too.
**Last confirmed current:** 2026-05-12

### Pattern: Plan mode before build mode
**Use when:** Any non-trivial change in Claude Code.
**Implementation:** Press `Shift+Tab` to enter plan mode. Claude proposes the plan, Milica reviews, then approves to execute.
**Last confirmed current:** 2026-05-12

### Pattern: Skills as portable workflow definitions
**Use when:** Any repeatable Claude workflow that should travel between Claude.ai, Claude Code, and Cowork.
**Implementation:** Single `SKILL.md` with YAML frontmatter (`name`, `description`). Drop into `.claude/skills/` in repos, project knowledge in Claude.ai.
**Last confirmed current:** 2026-05-12

---

## Deprecated / flag on sight

### Old Sonnet 4 model string in SALOS
- **Old way:** `claude-sonnet-4-20250514`
- **New way:** `claude-sonnet-4-6`
- **Why it changed:** Sonnet 4.6 has better agentic search performance and uses fewer tokens. 1M context is standard.
- **Migration:** Single string replacement across the codebase. Test one research call before merging.
- **Hard deadline:** No forced retirement yet, but recommend migrating soon to capture token savings.

### Haiku 3 model string
- **Old way:** `claude-3-haiku-20240307`
- **New way:** `claude-haiku-4-5` (or the current Haiku 4.5 dated alias)
- **Why it changed:** Haiku 3 was retired on April 20, 2026.
- **Migration:** Replace any reference. SALOS does not appear to use Haiku 3 currently, but check.
- **Hard deadline:** Already retired. Any remaining usage will fail.

### Opus 4 and 4.1 model strings
- **Old way:** `claude-opus-4-...`, `claude-opus-4-1-...`
- **New way:** `claude-opus-4-7` (current latest) or `claude-opus-4-6` (still supported)
- **Why it changed:** Both were removed from the Claude model selector and Claude Code.
- **Migration:** Replace if used anywhere.
- **Hard deadline:** Removed.

### Beta headers no longer needed
- **Old way:** Sending beta headers for `web_search` or programmatic tool calling
- **New way:** Drop the beta headers, both features are GA
- **Why it changed:** Promoted out of beta.
- **Migration:** Remove headers, no other changes required.
- **Hard deadline:** No forced deadline, but headers may eventually error.

### 1M context beta header for Sonnet 4.5 / Sonnet 4
- **Old way:** `context-1m-2025-08-07` beta header with Sonnet 4.5 or Sonnet 4
- **New way:** Migrate to Sonnet 4.6 or Opus 4.6 where 1M context is standard
- **Why it changed:** Beta retired on April 30, 2026.
- **Migration:** Update model string. No header needed afterward.
- **Hard deadline:** Already past. Requests now error if they exceed 200k on the old models.

---

## Watch list

### Dreaming for Managed Agents
**Status:** Research preview
**Source:** https://simonwillison.net/2026/May/6/code-w-claude-2026/

Agents review past sessions overnight and create new memory files (e.g. `descent-playbook.md`). This is the most relevant upcoming pattern for the NEBOS Cradle architecture. Track closely.

### Multi-agent orchestration
**Status:** Public beta (`managed-agents-2026-04-01`)
**Source:** https://platform.claude.com/docs/en/release-notes/overview

Fleets of cooperating agents with a Commander pattern. Relevant for NEBOS Pearl-to-Pearl coordination once SALOS is the second Pearl.

### Outcomes for Managed Agents
**Status:** Public beta
**Source:** https://platform.claude.com/docs/en/release-notes/overview

Set success criteria, agent iterates until met. Conceptually similar to Claude Code's `/goal` but at the API/agent level. Watch for fit with SALOS sequence execution.

### Agent view in Claude Code
**Status:** Research preview
**Source:** https://github.com/anthropics/claude-code/releases

Already usable via `claude agents`. Promote to active pattern once Milica has used it for a week.

### Claude Mythos Preview
**Status:** Gated research preview
**Source:** https://platform.claude.com/docs/en/release-notes/overview

Defensive cybersecurity work. Not directly relevant to SALOS/NEBOS but worth knowing about.

### Claude Security public beta
**Status:** Public beta (Enterprise only)
**Source:** https://releasebot.io/updates/anthropic/claude

Code vulnerability scanning with Opus 4.7. Not relevant unless TR3I or NEBOS adopt Enterprise plans.

---

## Sources checked this refresh

| Source | URL | Status |
|--------|-----|--------|
| Anthropic news | https://www.anthropic.com/news | Fetched |
| Claude API release notes | https://platform.claude.com/docs/en/release-notes/overview | Fetched |
| Claude Code changelog | https://code.claude.com/docs/en/changelog | Fetched |
| Claude Code GitHub releases | https://github.com/anthropics/claude-code/releases | Fetched |
| Claude apps release notes | https://support.claude.com/en/articles/12138966-release-notes | Fetched |
| Simon Willison Anthropic tag | https://simonwillison.net/tags/anthropic/ | Fetched |
| Reddit r/ClaudeAI top weekly | https://www.reddit.com/r/ClaudeAI/top/?t=week | Skipped this run |
| TypeScript SDK releases | https://github.com/anthropics/anthropic-sdk-typescript/releases | Skipped this run |

---

## Refresh summary

- **New items this refresh:** 13
- **Most important for SALOS right now:**
  1. Migrate to Sonnet 4.6 - lower cost, better agentic performance, 1M context standard
  2. Add prompt caching to the org config block - significant input token reduction
  3. Try the Advisor tool for proposal generation - Sonnet + Opus in one call
- **Deprecated, action now:** Plan the Sonnet 4 to 4.6 migration as a small standalone PR
- **Sources failed:** None (2 skipped, will include next run)
- **Next refresh due:** 2026-05-19
