# Usage examples

These examples show what activating HeavySkill looks like from a user's perspective and what happens inside the harness.

## Example 1 — Competition math (auto-activation)

**User:**

> Find the smallest positive integer $n$ such that $\lfloor \sqrt{n} \rfloor$ divides $n$ and $\lfloor \sqrt[3]{n} \rfloor$ divides $n$ and $\lfloor \sqrt[4]{n} \rfloor$ divides $n$, but $\lfloor \sqrt[5]{n} \rfloor$ does not divide $n$.

**What the harness does:**

1. The skill auto-activates because the query matches the "competition math" trigger in `SKILL.md`.
2. The main agent emits three `Agent` tool calls in a single message:
   - Thinker #1: tries a number-theoretic approach (lcm / divisibility lattice).
   - Thinker #2: brute-force enumeration with the four floor constraints.
   - Thinker #3: hybrid — narrows the search space algebraically, then enumerates.
3. The three trajectories return.
4. The main agent runs the deliberation step itself:
   - Identifies the answer distribution.
   - Checks each chain for arithmetic and logical errors.
   - Cross-validates the candidate answers against the original constraints.
5. The main agent replies to the user with `\boxed{n}` only — no meta-analysis exposed.

## Example 2 — Explicit invocation

**User:**

> Use heavy-thinking. Prove that the only solutions in positive integers to $a^3 + b^3 = c^3 + 1$ with $a \le b$ and $c \ge 1$ are $(a,b,c) = (1,1,1)$ and the family $(a, a, a-1)$ ... wait, double-check this claim too.

The explicit "Use heavy-thinking" makes activation unambiguous. The harness runs the K=3 protocol as in Example 1.

## Example 3 — When the skill should *not* fire

**User:**

> What's the capital of France?

This is a simple factual question. Per the "Do NOT activate" section of `SKILL.md`, the skill stays dormant and the main agent answers directly. Spawning three parallel agents for a one-word answer would be pure overhead.

## Example 4 — Iterative refinement (N=2)

For a problem where the first deliberation pass leaves the main agent unsure (e.g., trajectories split 1-1-1 with all reasoning chains plausible), the main agent can iterate:

1. Treat the first deliberation output as a fourth "expert thinker" trajectory.
2. Re-run Stage 2 deliberation over the four trajectories.
3. If a stable answer emerges, return it; otherwise spawn one more wave of K=3 parallel agents and repeat.

The paper's default is `N=1`. Iterate only when the first pass is genuinely inconclusive.

## Inspecting trajectories during a session

While developing or debugging, you can ask the main agent to expose the per-thinker trajectories after the fact:

> Show me the three trajectories you collected and the deliberation reasoning before giving me the final answer.

The skill's "Output Format" section says the *default* output is the final answer only — but the trajectories are still in the conversation context and can be surfaced on request.
