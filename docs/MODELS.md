# Models

## Global routing (hermes/config.yaml)

```yaml
model:
  provider: minimax
  default: MiniMax-M2.7

fallback_providers:
  - provider: deepseek
    model: deepseek-chat
  - provider: anthropic
    model: claude-opus-4-6
```

Every Hermes process starts with this chain: try MiniMax direct, fall
back to DeepSeek direct, finally Anthropic. Per-agent overrides in the
manifest (e.g. Heero pinned to `anthropic/claude-opus-4-6`) short-circuit
the chain for that agent only.

## Per-agent assignments and rationale

| Agent      | Model                           | Why                                                                                        |
|------------|---------------------------------|--------------------------------------------------------------------------------------------|
| Treize     | `MiniMax-M2.7`                  | CEO: lots of short decisions, no long reasoning chains. Cost-optimized.                    |
| Misato     | `MiniMax-M2.7`                  | PM: task decomposition, status tracking. Pattern-heavy, model-agnostic.                    |
| **Heero**  | `anthropic/claude-opus-4-6`     | Lead Engineer: architecture decisions + code review need Opus-class reasoning.             |
| Ritsuko    | `MiniMax-M2.7`                  | Backend engineer — straight-line implementation, MiniMax handles 95% of it.                |
| Quatre     | `MiniMax-M2.7`                  | Full-stack — same logic as Ritsuko.                                                         |
| Asuka      | `MiniMax-M2.7`                  | Frontend — UI work is template-heavy.                                                       |
| Wufei      | `MiniMax-M2.7`                  | QA — test generation is pattern-heavy.                                                      |
| Cid        | `MiniMax-M2.7`                  | DevOps — deploy scripts, systemd, monitoring. Rote.                                         |
| Ed         | `MiniMax-M2.7`                  | Research — web scraping + summarization. Cheap model wins.                                 |
| Toji       | `MiniMax-M2.7`                  | Financial analysis — spreadsheet-style work.                                               |
| **Jet**    | `anthropic/claude-sonnet-4-6`   | Content Lead: voice review + editorial judgment. MiniMax produced shallow drafts (D-148).  |
| **Vicious**| `anthropic/claude-sonnet-4-6`   | Content ideation: needs to generate *non-obvious* takes, not reformulate prompts (D-148).  |
| Faye       | `MiniMax-M2.7`                  | X drafter — 280-char format, voice rules, pattern-heavy.                                   |
| Spike      | `MiniMax-M2.7`                  | Outreach — research + email drafts. Heuristic, fast, cheap.                                |
| tf-frasier | `MiniMax-M2.7`                  | Router — one-shot classification. Tiniest model would work.                                 |

## The Sonnet carve-out for content (D-148, 2026-04-22)

We ran Vicious + Jet on MiniMax for two weeks. Output was technically
coherent but consistently shallow — generic LinkedIn posts, recycled
angles, no sharp takes. Moving them to `claude-sonnet-4-6` was a clear
quality lift at ~15× the token cost. Since content accounts for <5% of
total fleet tokens, total spend barely moved.

Heuristic we now use:

> If the agent's job is to generate *quality* (editorial judgment,
> architectural decisions, sharp-edged writing), use Opus or Sonnet. If
> the job is to execute (code, summarize, classify, transform), MiniMax.

This is a cost-quality frontier trade-off. If you're content-first,
budget for Sonnet-class on your content seats. If you're
ops-automation-first, MiniMax covers 90% of the fleet at 5% of the cost.

## MiniMax-M2.7 notes

- Access via direct API (not through OpenRouter) — Hermes ships with a
  first-class MiniMax provider.
- Fast enough for synchronous heartbeat-driven flows.
- Struggles with: long multi-step reasoning (5+ tool calls), nuanced
  tone-matching, adversarial/debate content.
- Handles well: code generation, structured data extraction, JSON/YAML
  manipulation, templated content.

## Fallback behavior

Hermes tries each provider in `fallback_providers` in order on
provider-level errors (rate limit, 500, network). It does **not** fall
back on empty/low-quality completions — an agent silently producing
garbage looks identical to working output to the router. Catch quality
regressions via content review, not the router.

## Model bump playbook

1. Pick the agent whose output quality is blocking.
2. PATCH that single agent's `adapterConfig.model` to the bigger model.
3. Run for a week. Compare output quality (manual, not auto — routers
   don't read).
4. If better: leave it, update `MODELS.md`.
5. If no change: revert. The smaller model was fine; the bottleneck was
   prompt or context, not model capability.
