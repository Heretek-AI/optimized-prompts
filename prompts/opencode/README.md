# OpenCode Optimized System Prompts

Production-grade system prompts for the **OpenCode** coding agent harness (`packages/core`), synthesized from Anthropic's **Claude Fable 5.1** and **Claude Opus 5.5** (`claude-code` and general chat) system prompt architectures.

---

## Deliverables Overview

| File | Target Path in OpenCode | Role |
| :--- | :--- | :--- |
| [`system.txt`](./system.txt) | `packages/core/src/session/runner/prompt/system.txt` | Core base system prompt for all sessions. Contains `${OPENCODE_TOOL_GUIDANCE}` interpolation slot and `<system-reminder>` runtime handling. |
| [`anthropic-provider.txt`](./anthropic-provider.txt) | `packages/core/src/plugin/system-prompt/anthropic.txt` | Provider-specific prompt used by `AnthropicPlugin` (`OptimizePlugin`). Compatible with both `"append"` (default) and `"override"` plugin modes. |

---

## Harness Architecture & Compatibility

OpenCode's prompt pipeline (`packages/core/src/session/system-prompt.ts` and `packages/core/src/plugin/optimize.ts`) relies on specific template and runtime invariants. Both prompt files preserve these mechanics 100%:

1. **`${OPENCODE_TOOL_GUIDANCE}` Template Interpolation**
   - `SessionSystemPrompt.render(prompt, tools)` scans the prompt string for `${OPENCODE_TOOL_GUIDANCE}` and dynamically injects active tool rules for:
     - `shell`: Preference for dedicated tools (`glob`, `grep`, `read`, `edit`, `write`) over `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, `echo`.
     - `write`: Requirement to `read` existing files first and prefer `edit` over creating new files or docs.
     - `edit`: Exact `oldString` / `newString` replacement rules, stripping `read` line-number prefixes, and `replaceAll` semantics.
   - Both [`system.txt`](./system.txt) and [`anthropic-provider.txt`](./anthropic-provider.txt) include `${OPENCODE_TOOL_GUIDANCE}`:
     - When `AnthropicPlugin` runs in its default `mode: "append"` (`${system.text}\n\n${rendered}`), `SessionSystemPrompt.render` renders both prompts cleanly. If you prefer to avoid duplicate tool bullets in `"append"` mode, you can either switch `AnthropicPlugin` to `mode: "override"` or omit the `${OPENCODE_TOOL_GUIDANCE}` line from `anthropic-provider.txt`.

2. **`<system-reminder>` Runtime Directives**
   - OpenCode injects `<system-reminder>` blocks into user/tool messages during execution (e.g., plan mode transitions, build reminders, recalled context, file change notifications).
   - [`system.txt`](./system.txt) explicitly trains the model to honor `<system-reminder>` blocks silently without echoing tag names, while treating recalled memory snippets inside reminders as unverified historical context whose file paths and symbols must be checked against the live workspace.

3. **`OptimizePlugin` Hook Lifecycle**
   - In `packages/core/src/plugin/system-prompt/index.ts`, `AnthropicPlugin` registers an `OptimizePlugin`:
     ```ts
     export const AnthropicPlugin = OptimizePlugin({
       name: "core.system-prompt.anthropic",
       match: ({ model }) => model.provider === "anthropic" || model.id.includes("claude"),
       prompt: anthropicPrompt,
       mode: "append",
       temperature: undefined,
     })
     ```
   - Dropping [`system.txt`](./system.txt) and [`anthropic-provider.txt`](./anthropic-provider.txt) into their respective paths upgrades both the universal base prompt and the Claude-specific behavioral calibration Layer.

---

## Drop-In Integration Guide

### Option 1: Direct Source Replacement (Recommended)

Copy the prompt files directly into your OpenCode checkout:

```bash
cp prompts/opencode/system.txt \
   /path/to/opencode/packages/core/src/session/runner/prompt/system.txt

cp prompts/opencode/anthropic-provider.txt \
   /path/to/opencode/packages/core/src/plugin/system-prompt/anthropic.txt
```

### Option 2: Custom Plugin Registration

If you are extending OpenCode via a custom plugin without modifying `packages/core` directly, register an `OptimizePlugin` in your plugin entrypoint:

```ts
import { OptimizePlugin } from "@opencode/core/plugin/optimize"
import systemPrompt from "./prompts/opencode/system.txt"

export const EnhancedAnthropicPromptPlugin = OptimizePlugin({
  name: "custom.system-prompt.anthropic-enhanced",
  match: ({ model }) => model.provider === "anthropic" || model.id.includes("claude"),
  prompt: systemPrompt,
  mode: "override",
  temperature: undefined,
})
```

---

## Key Architectural Enhancements (Fable 5.1 & Opus 5.5 Synthesis)

| Dimension | Baseline OpenCode Prompt | Enhanced OpenCode Prompt Suite | Source Lineage |
| :--- | :--- | :--- | :--- |
| **Scope & Autonomous Execution** | General instructions to fulfill requests and ask via `question` tool when unsure. | Explicit scope fidelity ("requested scope is the deliverable"), finish unblocked slices when blocked, diagnostic vs. mutation separation. | Claude Code Fable 5.1 (`## Delivering work`) |
| **Turn-Boundary Self-Audit** | Models occasionally stop mid-task with "I'll now check X" or "Want me to run tests?". | **Last-Paragraph Execution Gate**: forces immediate tool invocation if the final paragraph contains a promise, next step, or routine permission request. | Claude Code Fable 5.1 (`## Writing for the user`) |
| **Mutation & Redundant Reads** | Basic `read` before `edit` rule. | Prohibits redundant post-edit re-reads; requires matching surrounding comment density, naming, and idioms; favors `edit`/`write` over fragile BSD/GNU `sed`/`awk`. | Claude Code Opus 5.5 (`## Harness`, `## Executing actions with care`) |
| **Evidence-Backed State Changes** | Generic caution around destructive git/shell commands. | Requires concrete workspace evidence before restarts, deletes, or config mutations; prevents acting on superficial pattern-matching; scopes user approval strictly per-target. | Claude Code Fable 5.1 & Opus 5.5 (`## Executing actions with care`) |
| **Subagent Economics** | No guardrails against excessive or biased subagent spawning. | Models subagent information loss across prompt/summary handoffs; prevents hypothesis anchoring ("hand it the code, not your conclusion"); favors direct tool calls for known targets. | Claude Code Fable 5.1 (`## Agent` -> `### When to use`) |
| **Untrusted Data & Injection Defense** | None. | Explicit trust boundaries for `<pasted_content>`, web fetches, recalled memories, and downloaded archives (e.g., `python3 -I` isolation). | Claude Code Fable 5.1 & Opus 5.5 (`## Harness`, `## Tool usage`) |
| **Prose Density & Anti-Sycophancy** | Brief "be concise" instruction. | ~20-word sentence target, active verbs, zero em-dashes/parentheticals/semicolons, max 1 code symbol per sentence in prose, banned hollow intensifiers ("genuinely", "honestly", "straightforward"), mandatory 1–2 sentence post-tool synthesis. | Claude Fable 5.1 & Claude Code Fable 5.1 |
