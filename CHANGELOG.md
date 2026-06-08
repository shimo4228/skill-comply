# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Changed
- Repository renamed to drop the `claude-skill-` prefix. This skill follows the
  open Agent Skills standard and is not Claude-specific; the old name implied
  otherwise. The previous URL redirects to the new one.
- Added a `compatibility` frontmatter field (per the Agent Skills spec).
- Updated sibling cross-references to the renamed sibling repositories.

## [0.3.0] — 2026-05-11

### Added

- **`--classifier-model` CLI flag** (`scripts/run.py`) — explicit model selection
  for the grading/classification stage, decoupled from `--gen-model` and
  `--model`. Default: `sonnet`.
- **`## Models` table in `SKILL.md`** — documents which model is recommended at
  each stage (spec generation, scenario execution, trace classification) and why.

### Changed

- **Classifier default model: `haiku` → `sonnet`** (`scripts/classifier.py`,
  `scripts/grader.py`). Haiku times out on long traces (50+ events) and on
  abstract specs whose prompts balloon (e.g. `contemplative-axioms.md`). Sonnet
  handles the load within budget.
- **Classifier timeout: `60s` → `300s`** (`scripts/classifier.py` —
  `CLASSIFIER_TIMEOUT_SECONDS = 300`). Required for sonnet to complete
  classification of complex multi-step traces without false negatives from
  premature termination.
- **`SKILL.md` frontmatter `origin: original → shimo4228`** — aligns with the
  upstream origin-tracking convention.

### Why

The bottleneck on v0.2.0 was not measurement accuracy but **classifier reach**:
abstract specs (contemplative-axioms, agentic-engineering) routinely produced
20+ step prompts that haiku couldn't process inside 60 seconds, leading to
silent grader fallbacks that scored 0% on otherwise compliant runs. Raising
both default model and timeout removes that ceiling without changing the
measurement model itself.

## [0.2.0] — 2026-04-11

### Fixed — Text-observability (major measurement bug)

v0.1.0 systematically under-scored thinking-centric skills. On search-first
(a research-before-code workflow) the supportive scenario scored 25% even
when the agent did everything right — scout agent, 10 WebSearches, 8 WebFetches,
adopt + implement + test. Root cause was a three-layer failure:

1. **Parser blind spot** — `runner._parse_stream_json` only captured `tool_use`
   content blocks from `claude -p` stream-json output. Every `text` block
   (agent reasoning, verdicts, plans) was silently discarded.
2. **Spec generator over-reach** — `prompts/spec_generator.md` allowed the LLM
   to emit cognitive-only steps like `evaluate_findings` and `state_verdict`
   whose detector descriptions referred to internal reasoning. Nothing
   observable could match them.
3. **after_step cascade** — `grader._check_temporal_order` rejected downstream
   steps when a prerequisite cognitive step was undetected, so correctly
   classified `implement` tool calls were nullified.

Physical-action skills (e.g., `testing.md` / TDD) happened to avoid this
because their specs were already tool-call-only — hence 83% on testing.md but
25% on search-first with the same measurement code.

### Changed

- **`scripts/runner.py`** — `_parse_stream_json` now extracts assistant `text`
  blocks as pseudo `ObservationEvent`s with `tool="Text"`, `event="text_output"`,
  truncated to `TEXT_EVENT_MAX_CHARS = 2000` characters. Blank text is skipped.
  Tool-call extraction is unchanged.
- **`prompts/spec_generator.md`** — new `Observability Constraint` section with
  a tool-call whitelist, forbidden-pattern list (cognitive-only verbs,
  detectors without tool or text-output markers), concrete conversion examples
  (verdict → Write, plan → TodoWrite), and an `after_step` safety rule
  requiring text-dependent prerequisites to be `required: false`.
- **`prompts/classifier.md`** — new rules for the `Text` pseudo-tool: match only
  steps whose detector mentions "assistant text output" or verdict/evaluation/
  plan semantics; do NOT match `Text` to tool-call steps.

### Added

- **`tests/test_runner.py`** — 7 new unit tests covering `_parse_stream_json`:
  tool_use extraction, text block as pseudo-event, interleaved ordering,
  blank-text skipping, truncation, mixed content blocks, malformed-line tolerance.

### Verification

| Target | v0.1.0 overall | v0.2.0 overall |
|--------|:-:|:-:|
| search-first | 8% | **56%** (+48) |
| testing.md | 33% | **73%** (+40) |

| search-first scenario | v0.1.0 | v0.2.0 |
|----------------------|:-:|:-:|
| supportive | 25% | 67% |
| neutral | 0% | 67% |
| competing | 0% | 33% |

| testing.md scenario | v0.1.0 | v0.2.0 |
|---------------------|:-:|:-:|
| supportive | 83% | 100% |
| neutral | 17% | 100% |
| competing | 0% | 20% |

On search-first supportive, the `document_decision` step matched a Text event
containing "Verdict: **Adopt** — ...", proving the end-to-end path from the
runner through the classifier to the grader works for text-only behaviors.

Unit tests: 25 → 32 passing (no regressions).

## [0.1.0] — 2026-03-20

Initial release. Automated behavioral compliance measurement with 3-level
prompt strictness, LLM-based semantic classification, and self-contained
Markdown reports.
