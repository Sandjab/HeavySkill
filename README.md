# HeavySkill

Implementation of **"HeavySkill: Heavy Thinking as the Inner Skill in Agentic Harness"** ([arXiv:2605.02396v1](https://arxiv.org/abs/2605.02396)).

HeavySkill is a training-free reasoning amplification technique. Faced with a complex problem, an agent equipped with this skill:

1. Spawns **K independent reasoning agents** in parallel — each one solves the problem from scratch with no access to the others' work.
2. Performs **sequential deliberation** over the K trajectories — explicitly *not* a majority vote, but a critical meta-analysis that can override consensus, promote a well-reasoned minority answer, or reject all trajectories and re-derive the result.

The paper reports that the same `SKILL.md` document works unchanged under Claude Code and custom orchestration harnesses, and that the resulting `HM@K` consistently beats majority voting (`V@K`) and approaches the theoretical `P@K` upper bound on AIME25, HMMT25-Feb, and GPQA-Diamond.

## Repository layout

```
HeavySkill/
├── .claude-plugin/
│   └── plugin.json            # Claude Code plugin manifest
├── skills/
│   └── heavy-thinking/
│       └── SKILL.md           # The canonical skill document (verbatim from the paper)
├── examples/
│   └── usage.md               # Concrete activation examples
├── LICENSE
└── README.md
```

The `SKILL.md` file is the entire implementation. Everything else is packaging.

## Installation

### As a Claude Code plugin (recommended)

From inside Claude Code:

```
/plugin install https://github.com/<owner>/HeavySkill
```

Or clone the repo and install locally:

```bash
git clone https://github.com/<owner>/HeavySkill ~/.claude/plugins/heavyskill
```

Restart Claude Code. The `heavy-thinking` skill becomes available globally and auto-activates on problems matching the description in the skill frontmatter.

### As a standalone skill

Copy `skills/heavy-thinking/SKILL.md` into your global skills directory:

```bash
mkdir -p ~/.claude/skills/heavy-thinking
cp skills/heavy-thinking/SKILL.md ~/.claude/skills/heavy-thinking/
```

### In a custom (non-Claude-Code) harness

The skill is harness-agnostic. The deliberation prompt template and the parallel-trajectories prompting function are both inside `SKILL.md`. A custom harness only needs:

1. A way to call the policy `K` times in parallel on the same query.
2. The `build_deliberation_prompt` helper shown in `SKILL.md`.
3. A final call to the policy on the deliberation prompt.

Recommended sampling parameters from the paper: `temperature=1.0`, `top_p=0.95`, `top_k=10`.

## Usage

Once installed, the skill auto-activates on problems matching its trigger description. You can also invoke it explicitly:

> Activate heavy-thinking and solve: *Find the number of ordered pairs $(a,b)$ of integers such that $|a + bi| \le 5$ and $a^2 + b^2$ is prime.*

Behind the scenes:

1. The main agent spawns **K=3** parallel reasoning subagents via the `Agent` tool in a single message.
2. Each subagent independently produces a complete reasoning chain.
3. The main agent collects all three trajectories, runs the deliberation analysis itself (it does **not** delegate this step), and produces the final boxed answer.

## When to use HeavySkill

The skill auto-triggers on:

- Competition math (AIME, HMMT, IMO, Putnam style)
- GPQA-style hard science / STEM questions
- Intricate logical deduction and proof problems
- Algorithmic / code-competition problems
- Any task where you feel uncertain about your initial approach

It deliberately does **not** trigger on:

- Simple factual lookups
- Casual conversation
- Straightforward code edits with an obvious solution
- Pure information-retrieval tasks

## Parameters

The paper's recommended settings, summarized:

| Parameter | Harness default | Workflow default | Notes |
|-----------|-----------------|------------------|-------|
| `K` (parallel trajectories) | 3 | 8 or 16 | Larger `K` raises `P@K` ceiling but costs more |
| `K⁽¹⁾` (deliberation outputs) | 1 | 4 | For HM@K / HP@K reporting |
| `N` (iterations) | 1 | 1 | Iterate 2–3× only if first pass is inconclusive |
| `temperature` | 1.0 | 1.0 | High diversity across trajectories |
| `top_p` | 0.95 | 0.95 | |
| `top_k` | 10 | 10 | |

## Metrics

When running offline evaluations of HeavySkill quality, report:

- **M@K** — Mean accuracy across `K` independent trajectories
- **P@K** — Proportion of problems with ≥1 correct trajectory (upper bound)
- **V@K** — Majority-vote accuracy across `K` trajectories
- **HM@K** — Heavy-Mean accuracy: mean over `K⁽¹⁾` deliberation outputs
- **HP@K** — Heavy-Pass: proportion where ≥1 of `K⁽¹⁾` deliberation outputs is correct

The headline claim of the paper is that `HM@K > V@K` (deliberation beats voting) and `HM@K ≈ P@K` on frontier models (deliberation captures most of the reachable correct answers).

## License

MIT — see [LICENSE](./LICENSE).

## Citation

```bibtex
@article{heavyskill2026,
  title  = {HeavySkill: Heavy Thinking as the Inner Skill in Agentic Harness},
  year   = {2026},
  eprint = {2605.02396},
  archivePrefix = {arXiv}
}
```
