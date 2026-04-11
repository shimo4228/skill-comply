# claude-skill-comply

An [Agent Skill](https://agentskills.io/specification) that measures whether coding agents actually **follow** skills, rules, and agent definitions. Auto-generates test scenarios at 3 prompt strictness levels, runs agents, classifies tool call sequences via LLM, and reports compliance rates with full timelines.

## Install

### Claude Code

```bash
# Copy skill into your global skills directory
cp -r skills/skill-comply ~/.claude/skills/skill-comply
cd ~/.claude/skills/skill-comply && uv sync
```

### SkillsMP

```bash
/skills add shimo4228/claude-skill-comply
```

## How It Works

1. **Spec Generation** — LLM extracts expected behavioral steps from any `.md` file
2. **Scenario Generation** — Creates 3 scenarios with decreasing prompt support (supportive -> neutral -> competing)
3. **Execution** — Runs `claude -p` in sandbox, captures tool call traces via stream-json
4. **Classification** — LLM classifies tool calls against spec steps (semantic, not regex)
5. **Grading** — Deterministic temporal ordering validation
6. **Report** — Self-contained Markdown with compliance rates and full tool call timelines

## Key Concept: Prompt Independence

Tests whether a skill/rule is followed **even when the prompt doesn't explicitly support it**. The 3-level scenario structure covers the full spectrum:

| Level | Name | What it tests |
|-------|------|---------------|
| 1 | **Supportive** | Prompt explicitly mentions the skill |
| 2 | **Neutral** | Same task, skill not mentioned |
| 3 | **Competing** | Task instructions contradict the skill |

## Usage

```bash
cd ~/.claude/skills/skill-comply

# Full run
uv run python -m scripts.run ~/.claude/rules/common/testing.md

# Dry run (no cost, spec + scenarios only)
uv run python -m scripts.run --dry-run ~/.claude/skills/search-first/SKILL.md

# Custom models
uv run python -m scripts.run --gen-model haiku --model sonnet <path>
```

## Real-World Results (v0.2.0, post text-observability fix)

| Target | Overall | Supportive | Insight |
|--------|:-:|:-:|---------|
| testing.md | **73%** | **100%** | Observable 6-step TDD spec fully matches sonnet when explicitly instructed |
| search-first | **56%** | **67%** | Text-based verdicts (Adopt/Extend/Build) now captured via Text pseudo-events |
| security.md | dry-run OK | — | Spec + scenarios generated successfully |
| git-workflow.md | dry-run OK | — | Spec + scenarios generated successfully |

### v0.1.0 → v0.2.0 improvement

| Target | v0.1.0 overall | v0.2.0 overall | Δ |
|--------|:-:|:-:|:-:|
| search-first | 8% | 56% | **+48** |
| testing.md | 33% | 73% | **+40** |

v0.1.0 systematically under-scored thinking-centric skills because the runner
discarded assistant text blocks and the spec generator was free to emit
cognitive-only steps (`evaluate_findings`, `state_verdict`) that no tool call
could satisfy. After a downstream `after_step` dependency, cascading failures
nullified observable steps as well. v0.2.0 fixes both layers — see
[CHANGELOG.md](CHANGELOG.md).

## Requirements

- Python >= 3.11
- `uv` (or `pip install pyyaml`)
- Claude Code CLI (`claude`)

## Tests

```bash
cd skills/skill-comply && uv run pytest -v  # 32 tests
```

## License

MIT

---

## 日本語

スキル/ルール/エージェント定義が実際にエージェントに遵守されているかを自動計測する Agent Skill です。3段階のプロンプト厳格度でシナリオを生成し、ツールコールを LLM で意味的に分類、コンプライアンスレポートを出力します。

詳細は [`skills/skill-comply/SKILL.md`](skills/skill-comply/SKILL.md) を参照してください。
