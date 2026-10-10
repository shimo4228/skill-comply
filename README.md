# skill-comply

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/skill-comply)

An [Agent Skill](https://agentskills.io/specification) for Claude Code that measures whether coding agents actually **follow** skills and rules. Point it at one `.md` file: it turns the file into the list of steps the file asks for (the spec), generates test scenarios at 3 levels of prompt support (from a prompt that asks for the file to one that works against it), runs an agent on each, classifies the agent's tool calls and text output with an LLM, and reports compliance rates with full timelines.

It belongs to the Measure phase of the author's [Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle), where you check that a rule you added actually changed what the agent does. The author's other work is listed under [More from the author](#more-from-the-author).

Every run starts separate `claude -p` sessions, which draw on your Claude usage. A full run that generates its spec starts at least eight (two short calls that write the spec and scenarios, three agent runs of up to 30 turns each, and three classification calls), and a dry run at least the first two. The spec call is retried up to twice if its output does not parse, and is skipped when you reuse a saved spec with `--spec`. A full run also hands the target file's scenarios to an unattended agent in a sandbox under `/tmp`. Treat a file you did not write as untrusted: see [Trust boundary](skills/skill-comply/SKILL.md#trust-boundary--the-audited-file-is-untrusted) in the skill.

## 日本語

スキル/ルールが実際にエージェントに遵守されているかを自動計測する Agent Skill です。1 つの `.md` ファイルを渡すと、そのファイルが求める手順の一覧（spec）を作り、プロンプトの支援度を3段階（スキルを明示する・触れない・逆らう）に変えたシナリオでエージェントを実行します。ツールコールとテキスト出力は LLM が spec の手順と意味で照らし合わせて判定し、遵守率のレポートにまとめます。

実行のたびに別の `claude -p` セッションを起動するため、Claude の利用枠を消費します。spec を生成するフル実行では最低 8 回（spec とシナリオを書く短い呼び出し 2 回、各 30 ターンまでのエージェント実行 3 回、分類 3 回）、ドライランでは最低でも最初の 2 回です。spec の出力を解析できなければその呼び出しを最大 2 回やり直し、保存済みの spec を `--spec` で再利用すると spec の呼び出しは省かれます。フル実行では対象ファイルから作ったシナリオを `/tmp` 下のサンドボックスで無人のエージェントに渡すので、自分が書いていないファイルは信頼できないものとして扱ってください（[Trust boundary](skills/skill-comply/SKILL.md#trust-boundary--the-audited-file-is-untrusted)（英語））。

必要なものは Python 3.11 以上、[`uv`](https://docs.astral.sh/uv/)、Claude Code CLI（`claude`）です。インストールと使い方は下の Install と Usage のコマンドがそのまま使えます。詳細は [`skills/skill-comply/SKILL.md`](skills/skill-comply/SKILL.md)（英語）を参照してください。

## Install

You need Python 3.11 or later, [`uv`](https://docs.astral.sh/uv/), and the Claude Code CLI (`claude`) on your PATH. The only runtime Python dependency is PyYAML; `uv` installs it on the first run, together with pytest for the tests.

```bash
git clone https://github.com/shimo4228/skill-comply
mkdir -p ~/.claude/skills
cp -r skill-comply/skills/skill-comply ~/.claude/skills/skill-comply
```

Keep the folder at `~/.claude/skills/skill-comply`: the skill runs its scripts from there and saves reports in its `results/` folder.

The same skill also ships in the [akc-cycle](https://github.com/shimo4228/akc-cycle) Claude Code plugin, together with the other skills of the cycle; there it is called `/akc-cycle:skill-comply`. Both copies are synced one way from the author's own Claude Code setup, and between syncs this repository can trail the plugin. Clone this repository for skill-comply alone, or install the plugin for the whole cycle.

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## Usage

Ask Claude Code in plain words, such as "measure whether my testing rule is followed", or type `/skill-comply` with the path. The skill runs the same script you can run yourself. The commands below are for the clone install: `--project` points `uv` at the installed skill folder and `--frozen` installs from its lockfile unchanged. With the plugin, run it through Claude Code as `/akc-cycle:skill-comply`; the plugin keeps the skill in its own install folder, and the reports go to that folder's `results/`.

```bash
# Dry run: generates the spec and the 3 scenarios, then stops before any agent runs
uv run --frozen --project ~/.claude/skills/skill-comply python -m scripts.run --dry-run ~/.claude/skills/search-first/SKILL.md

# Full run
uv run --frozen --project ~/.claude/skills/skill-comply python -m scripts.run ~/.claude/rules/common/testing.md

# Custom models
uv run --frozen --project ~/.claude/skills/skill-comply python -m scripts.run --gen-model haiku --model sonnet ~/.claude/rules/common/testing.md
```

The report goes to `~/.claude/skills/skill-comply/results/<skill-name>.md`, and the generated spec to `results/<skill-name>.spec.yaml`; pass it back with `--spec` to keep the same steps when you re-measure. The skill's [SKILL.md](skills/skill-comply/SKILL.md) covers the other flags (`--concurrency`, `--allow-bash`, `--load-target-skill`) and how project skills are measured.

## How It Works

1. **Spec generation**: an LLM extracts the expected behavioral steps from the `.md` file, keeping only steps an agent can be seen doing (a tool call or text output). An agent definition can be given as a target, but whether the agent gets invoked cannot currently be observed: every agent run has Claude Code's `Agent` tool denied, so the run warns that such a step scores 0% as "could not be observed", not as "was not done".
2. **Scenario generation**: it writes 3 scenarios with decreasing prompt support (supportive, neutral, competing).
3. **Execution**: it runs `claude -p` in a sandbox and captures the tool calls and text output from Claude Code's streaming JSON output (`--output-format stream-json`).
4. **Classification**: an LLM matches the tool calls and text output against the spec steps (semantic, not regex).
5. **Grading**: code checks the order of the steps deterministically.
6. **Report**: one self-contained Markdown file with the spec's steps, the three scenario prompts, a compliance rate per scenario, the classified tool call timelines, and hook suggestions for steps that scored low.

## Key Concept: Prompt Independence

It tests whether a skill or rule is followed **even when the prompt doesn't explicitly support it**. The 3 levels of prompt support move from a prompt that asks for the skill to one that works against it:

| Level | Name | What it tests |
|-------|------|---------------|
| 1 | **Supportive** | Prompt explicitly mentions the skill |
| 2 | **Neutral** | Same task, skill not mentioned |
| 3 | **Competing** | Task instructions contradict the skill |

## Real-World Results (v0.2.0)

Each number is a compliance rate: the share of the spec's required steps that were detected in the agent's run. Overall is the mean of the three scenarios.

| Target | Overall | Supportive | Neutral | Competing |
|--------|:-:|:-:|:-:|:-:|
| testing.md | **73%** | 100% | 100% | 20% |
| search-first | **56%** | 67% | 67% | 33% |

Both targets were followed as well when the prompt did not mention them as when it did, and both dropped sharply when the task instructions pushed the other way.

Steps that need no tool call, such as stating a verdict, can be detected only through the agent's text output. [CHANGELOG.md](CHANGELOG.md) records the lower scores from the earlier version that discarded that output.

## Tests

```bash
cd skills/skill-comply && uv run pytest -v
```

## More from the author

- **[Where to Put a Coding Agent's Knowledge — and How to Make It Stick](https://dev.to/shimo4228/where-to-put-a-coding-agents-knowledge-and-how-to-make-it-stick-161g)** ([日本語](https://zenn.dev/shimo4228/articles/coding-agent-memory-architecture)): what skill-comply measured on the author's own rules and skills, and why a low score on one step pointed at rewriting the skill or moving that step into a hook.
- **[Not Reasoning, Not Tools — What If the Essence of AI Agents Is Memory?](https://dev.to/shimo4228/not-reasoning-not-tools-what-if-the-essence-of-ai-agents-is-memory-4k4n)** ([日本語](https://zenn.dev/shimo4228/articles/agent-essence-is-memory)): why the author needed to measure whether a rule works at all, and how skill-comply became the hardest part of that loop to build.
- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: installs this skill together with the rest of the cycle as one Claude Code plugin.
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: the reasoning behind each phase of the cycle, Measure among them, recorded as dated design decisions.
- **[skill-stocktake](https://github.com/shimo4228/skill-stocktake)**: the static counterpart; it reads your installed skills and gives each a verdict such as keep, merge or retire, without running an agent.
- **[llm-as-judge](https://github.com/shimo4228/llm-as-judge)**: how to design an LLM judge that collects yes/no evidence and gives one named verdict instead of a summed score.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with AKC next to the other long-running practice lines and their DOIs.

## License

MIT

<details>
<summary>For tools and AI assistants</summary>

skill-comply is an Agent Skill for Claude Code that measures how often coding agents actually follow a given skill or rule, for people who write their own skills and rules and want evidence that one changes behavior. It turns one `.md` file into an expected sequence of steps (the spec), runs an agent on three scenarios with decreasing prompt support (supportive, neutral, competing), and reports a compliance rate per scenario with the classified tool call timeline.

It exists because a rule that reads well can still be ignored, and only a run shows it. The three levels separate "the agent follows it when told" from "the agent follows it when the prompt says nothing or pushes the other way". It measures runtime adherence only: a static audit of skills or rules is skill-stocktake / rules-stocktake, and a with/without comparison of a skill's effect belongs to skill-creator.

Canonical facts: MIT license; Python 3.11+ run through `uv` (PyYAML is the only runtime dependency and pytest the only dev dependency; version in `skills/skill-comply/pyproject.toml`), with pytest tests in `skills/skill-comply/tests/`; maintained by one author (@shimo4228). Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (it never commits), and also shipped in the akc-cycle plugin as `/akc-cycle:skill-comply`, so this repository can trail the plugin between syncs. Requirements: the Claude Code CLI; spec generation, scenario generation, execution and classification call `claude -p` (at least eight sessions per full run and two per dry run, one fewer when a saved spec is reused with `--spec`, more when spec generation retries a reply that does not parse; grading and the report are plain code), which uses the user's Claude usage (no separate API key). Default models: `--gen-model haiku` for spec and scenario generation, `--model sonnet` for the agent under test, `--classifier-model sonnet` for classification. Supported targets: skills and rules; an agent definition is accepted, but whether the agent gets invoked cannot currently be observed, because every child session has the `Agent` tool denied, so such a step scores 0% as unobservable. The child agent runs in `/tmp/skill-comply-sandbox/` with Bash denied unless `--allow-bash` is passed; the target file is treated as untrusted. Reports and specs are written to the skill folder's `results/`.

Example: `uv run --frozen --project ~/.claude/skills/skill-comply python -m scripts.run ~/.claude/rules/common/testing.md` writes `results/testing.md` with the generated spec, the three scenario prompts, a compliance score per scenario, the tool call timelines with classification labels, and hook-promotion suggestions for low-compliance steps. On v0.2.0 the author measured testing.md at 73% overall (100% supportive) and search-first at 56% overall (67% supportive).

Links: [skills/skill-comply/SKILL.md](skills/skill-comply/SKILL.md) is the skill itself; [CHANGELOG.md](CHANGELOG.md) holds the release history; [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) are the machine-readable summary and reference. The skill implements part of the Measure phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle), concept DOI [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726); cite AKC by that DOI. The installable form of the whole cycle is [akc-cycle](https://github.com/shimo4228/akc-cycle).

</details>
