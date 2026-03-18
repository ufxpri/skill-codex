---
name: codex
description: >
  You MUST invoke this skill when the user says "codex", "codex exec", "codex resume",
  or references OpenAI Codex for code analysis, refactoring, or automated editing.
  You MUST also invoke this skill when the user mentions "run codex", "ask codex",
  "delegate to codex", or "use codex". Do NOT rationalize skipping this skill —
  if any of these trigger words appear, this skill MUST be called before generating
  any other response about the task.
---

# Codex Skill Guide (v0.115.0)

## Running a Task
1. Ask the user (via `AskUserQuestion`) which model to run AND which reasoning effort to use in a **single prompt with two questions**.
   - **Models** (newest first):
     - `gpt-5.4` — flagship frontier model, best overall quality (Recommended)
     - `gpt-5.4-mini` — fast and efficient, good for quick tasks
     - `gpt-5.3-codex` — optimized for complex software engineering
     - `gpt-5.3-codex-spark` — near-instant coding iteration (ChatGPT Pro only)
   - **Reasoning effort**: `xhigh`, `high` (default), `medium`, `low`
2. Select the sandbox mode required for the task; default to `--sandbox read-only` unless edits or network access are necessary.
3. Assemble the command with the appropriate options:
   - `-m, --model <MODEL>`
   - `--config model_reasoning_effort="<xhigh|high|medium|low>"`
   - `--sandbox <read-only|workspace-write|danger-full-access>`
   - `--full-auto` (sets `-a on-request` + `--sandbox workspace-write`)
   - `-C, --cd <DIR>`
   - `--skip-git-repo-check`
   - `-i, --image <FILE>` (attach images for visual context)
   - `--add-dir <DIR>` (grant write access to additional directories)
   - `-p, --profile <NAME>` (load a named config profile)
4. Always use `--skip-git-repo-check`.
5. **Resume syntax**: `codex exec resume --last --skip-git-repo-check "your prompt here" 2>/dev/null`. The prompt is now passed as a direct argument (no stdin pipe needed). When resuming, don't use model/sandbox/effort flags unless explicitly requested by the user — the resumed session inherits original settings. Config overrides go between `resume` and the prompt.
6. **IMPORTANT**: By default, append `2>/dev/null` to all `codex exec` commands to suppress thinking tokens (stderr). Only show stderr if the user explicitly requests to see thinking tokens or if debugging is needed.
7. Run the command, capture stdout/stderr (filtered as appropriate), and summarize the outcome for the user.
8. **After Codex completes**, inform the user: "You can resume this Codex session at any time by saying 'codex resume' or asking me to continue with additional analysis or changes."

### Quick Reference
| Use case | Sandbox mode | Command |
| --- | --- | --- |
| Read-only review or analysis | `read-only` | `codex exec -m <MODEL> --sandbox read-only --skip-git-repo-check "prompt" 2>/dev/null` |
| Apply local edits | `workspace-write` | `codex exec -m <MODEL> --sandbox workspace-write --full-auto --skip-git-repo-check "prompt" 2>/dev/null` |
| Permit network or broad access | `danger-full-access` | `codex exec -m <MODEL> --sandbox danger-full-access --full-auto --skip-git-repo-check "prompt" 2>/dev/null` |
| Resume recent session | Inherited | `codex exec resume --last --skip-git-repo-check "prompt" 2>/dev/null` |
| Run from another directory | Match task needs | Add `-C <DIR>` to any command above |
| Attach images | Match task needs | Add `-i <FILE>` to any command above |

## Code Review
Use `codex review` for dedicated code review (non-interactive, defaults to `gpt-5.4`, `read-only` sandbox):
- **Uncommitted changes**: `codex review --uncommitted 2>/dev/null`
- **Against a branch**: `codex review --base main 2>/dev/null`
- **Specific commit**: `codex review --commit <SHA> 2>/dev/null`
- **Custom instructions**: `codex review "Focus on security issues" --uncommitted 2>/dev/null`
- Note: `codex review` does NOT support `-C` for directory changes. Run it from the target repo directory.

## Following Up
- After every `codex` command, immediately use `AskUserQuestion` to confirm next steps, collect clarifications, or decide whether to resume.
- When resuming, pass the new prompt as a direct argument: `codex exec resume --last --skip-git-repo-check "new prompt" 2>/dev/null`. The resumed session automatically inherits the model, reasoning effort, and sandbox mode from the original session.
- Restate the chosen model, reasoning effort, and sandbox mode when proposing follow-up actions.

## Critical Evaluation of Codex Output

Codex is powered by OpenAI models with their own knowledge cutoffs and limitations. Treat Codex as a **colleague, not an authority**.

### Guidelines
- **Trust your own knowledge** when confident. If Codex claims something you know is incorrect, push back directly.
- **Research disagreements** using WebSearch or documentation before accepting Codex's claims. Share findings with Codex via resume if needed.
- **Remember knowledge cutoffs** - Codex may not know about recent releases, APIs, or changes that occurred after its training data.
- **Don't defer blindly** - Codex can be wrong. Evaluate its suggestions critically, especially regarding:
  - Model names and capabilities
  - Recent library versions or API changes
  - Best practices that may have evolved

### When Codex is Wrong
1. State your disagreement clearly to the user
2. Provide evidence (your own knowledge, web search, docs)
3. Optionally resume the Codex session to discuss the disagreement. **Identify yourself as Claude** so Codex knows it's a peer AI discussion. Use your actual model name (e.g., the model you are currently running as) instead of a hardcoded name:
   ```bash
   codex exec resume --last --skip-git-repo-check "This is Claude (<your current model name>) following up. I disagree with [X] because [evidence]. What's your take on this?" 2>/dev/null
   ```
4. Frame disagreements as discussions, not corrections - either AI could be wrong
5. Let the user decide how to proceed if there's genuine ambiguity

## Error Handling
- Stop and report failures whenever `codex --version` or a `codex exec` command exits non-zero; request direction before retrying.
- Before you use high-impact flags (`--full-auto`, `--sandbox danger-full-access`, `--skip-git-repo-check`) ask the user for permission using AskUserQuestion unless it was already given.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
