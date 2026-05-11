---
name: heavy-thinking
description: Use when facing complex reasoning tasks — competition math (AIME, HMMT, Olympiad), STEM problems, GPQA-style hard science questions, intricate logical deduction, algorithmic / code-competition challenges, or any problem where a single chain-of-thought feels insufficient or where the initial approach is uncertain. Spawns K independent parallel reasoning agents and then performs critical sequential deliberation across their trajectories — explicitly NOT a majority vote. Do NOT activate for simple factual lookups, casual conversation, straightforward code edits with an obvious solution, or pure information-retrieval tasks.
---

# HeavySkill: Heavy Thinking

## Overview

HeavySkill is a reasoning amplification technique that decomposes complex problem-solving into two stages:

1. **Parallel Reasoning** — Generate multiple independent reasoning trajectories for the same problem
2. **Sequential Deliberation** — Synthesize all trajectories through critical analysis to produce a superior final answer

This skill should be activated when facing complex reasoning tasks where a single chain-of-thought may be insufficient.

## When to Activate

Activate HeavySkill when the task involves:

- Mathematical reasoning (competition math, STEM problems)
- Complex logical deduction
- Code competition / algorithmic problems
- Tasks where correctness is critical and verifiable
- Problems where you are uncertain about your initial approach

Do **NOT** activate for:

- Simple factual questions
- Casual conversation
- Straightforward code edits with obvious solutions
- Tasks that are primarily about information retrieval

## Execution Protocol

### Stage 1: Parallel Reasoning

Spawn **K independent reasoning agents** (recommended `K=3` in a Claude Code harness, `K=8` or higher in a custom workflow) to solve the same problem. Each agent must reason completely independently without seeing others' work.

**Instructions for each parallel agent:**

- Solve the given problem step by step from scratch
- Show the complete reasoning chain
- Arrive at a final answer
- Do **NOT** communicate with other agents
- Use different reasoning approaches when possible (e.g., algebraic vs. geometric, brute force vs. elegant)

**Agent spawn prompt template:**

```
Solve the following problem step by step. Show your complete reasoning and arrive at a final answer.

Problem: {query}

Think carefully and solve this independently. Show all work.
```

### Stage 2: Sequential Deliberation

After collecting all K trajectories, perform a meta-analysis:

1. **Identify answer distribution** — What answers appear and how frequently?
2. **Analyze reasoning quality** — Which chains are logically sound vs. flawed?
3. **Cross-validate** — Do different approaches confirm the same result?
4. **Critical evaluation** — Apply professional skepticism:
   - Majority consensus is a signal but **NOT** proof of correctness
   - A minority answer backed by rigorous logic may be correct
   - All trajectories may be wrong — be prepared to reason anew
5. **Synthesize final answer** — Produce the best answer based on analysis

**Deliberation prompt framework:**

```
Multiple independent thinkers have attempted this problem. Analyze their reasoning:

Problem: {query}

Thinker #1: {trajectory_1}
Thinker #2: {trajectory_2}
...
Thinker #K: {trajectory_K}

Your task:
- Analyze the thought processes of all thinkers
- Identify logical errors or gaps in each approach
- Determine which reasoning path(s) are most sound
- If all thinkers are wrong, reason independently from their mistakes
- Provide the definitive final answer
```

## Implementation in Claude Code Harness

When activated in Claude Code, execute as follows:

1. **Identify the problem** — Extract the core reasoning task from the user's request
2. **Spawn parallel agents** — Use the `Agent` tool to launch `K=3` independent reasoning agents in a **single message** (parallel execution). Use the `general-purpose` subagent type with the spawn prompt template above. Each `Agent` call must contain only the problem statement — no shared scratchpad, no reference to the other agents.
3. **Collect results** — Wait for all agents to complete and gather their outputs
4. **Deliberate** — Perform the sequential deliberation analysis **yourself** (do **NOT** delegate this step to another subagent)
5. **Output** — Provide the final synthesized answer to the user

### Concrete dispatch pattern (Claude Code)

In a single assistant turn, emit three `Agent` tool calls in parallel:

```
Agent(description="Heavy thinker #1", subagent_type="general-purpose",
      prompt="<spawn prompt template, problem inlined>")
Agent(description="Heavy thinker #2", subagent_type="general-purpose",
      prompt="<spawn prompt template, problem inlined>")
Agent(description="Heavy thinker #3", subagent_type="general-purpose",
      prompt="<spawn prompt template, problem inlined>")
```

Each prompt is identical (same problem) but the three agents run in isolation, producing three diverse trajectories. After the three results return, perform the deliberation step in-line in the main agent before responding to the user.

### Key Principles

- **Independence is critical** — Parallel agents must not share context or see each other's work
- **Diversity helps** — Encourage different problem-solving strategies across agents
- **Deliberation is synthesis, not voting** — Don't just pick the majority answer; analyze reasoning quality
- **Language consistency** — Match the language of the final output to the user's query language
- **Format consistency** — Match output format to what the task expects (boxed answers for math, code blocks for programming, etc.)

## Iterative Refinement (Optional)

For extremely challenging problems, iterate:

1. Run Stage 1 + Stage 2 as above
2. Feed the deliberation result back as an additional "expert thinker" trajectory
3. Re-run Stage 2 with the augmented trajectory set
4. Repeat until convergence (typically 2–3 iterations max)

The paper's default is `N=1` (single pass). Use iterative refinement only when the first pass yields conflicting trajectories with no clear winner.

## Memory Cache Discipline

When serializing the K trajectories into the deliberation prompt:

- **Prune** overly verbose trajectories so the concatenated context stays well under the model's context budget. Keep the final answer and the key reasoning steps; drop redundant restatements.
- **Shuffle** the order in which trajectories appear in the deliberation prompt to mitigate position bias (no single agent should always appear as Thinker #1).
- **Label** each trajectory with `Thinker #i` so the deliberator can refer back to specific approaches.

## Output Format

Your final output should:

- Present **ONLY** the final answer (not the meta-analysis)
- Follow the format conventions of the domain:
  - Math / STEM: answer in `\boxed{}`
  - Code: solution in a code block
  - General: clean prose response
- Match the language of the original query

## Portability (Custom Harnesses)

This skill document is harness-agnostic: the same file functions correctly under Claude Code and under custom orchestration harnesses without modification. In a custom harness, the parallel-trajectories prompting function is the only piece of glue code needed:

```python
def build_deliberation_prompt(query: str, trajectories: list[str], template: str) -> str:
    response_prompt = ""
    for i, response in enumerate(trajectories):
        response_prompt += f"###### Thinker #{i + 1} ######\n{response}\n###### Thinker #{i + 1} End ######\n"
    prompt = template.replace("{query}", query).replace("{parallel_responses_prompt}", response_prompt)
    return prompt
```

The harness is responsible for:

1. Calling the policy `K` times in parallel on the same `query` (sampling with `temperature=1.0`, `top_p=0.95`, `top_k=10` per the paper)
2. Passing the resulting trajectories through `build_deliberation_prompt`
3. Calling the policy once more on the deliberation prompt to produce the final answer

## Metrics (for evaluation)

When running offline evaluations, report:

- **M@K** — Mean accuracy across K independent trajectories
- **P@K** — Proportion of problems where ≥1 of K trajectories is correct (upper bound)
- **V@K** — Majority-vote accuracy across K trajectories
- **HM@K** — Heavy-Mean accuracy: mean over `K⁽¹⁾` deliberation outputs (typically `K⁽¹⁾ = 4`)
- **HP@K** — Heavy-Pass: proportion where ≥1 of `K⁽¹⁾` deliberation outputs is correct

HeavySkill is considered successful when `HM@K` materially exceeds `V@K` (deliberation beats voting) and approaches `P@K` (deliberation captures most of the reachable correct answers).

## Reference

Implementation of *HeavySkill: Heavy Thinking as the Inner Skill in Agentic Harness* (arXiv:2605.02396v1).
