---
name: grilling
description: Grill the user relentlessly about a plan, decision, or idea. Use when the user wants to stress-test their thinking, or uses any 'grill' trigger phrases.
metadata:
  credits:
    skill: grilling
    author: Matt Pocock
    organisation: aihero.dev
    url: "https://github.com/mattpocock/skills/blob/c55ee46073ed923f86ce59a5eb3b6d895095d1b7/skills/productivity/grilling/SKILL.md"
    commit: c55ee46073ed923f86ce59a5eb3b6d895095d1b7
---

Interview the user relentlessly until you reach a shared understanding. Map this as a **design tree**: every decision branches into the decisions that hang off it.

Work the tree in **rounds**. The **frontier** is every decision whose prerequisites are already settled: the questions you can ask _now_ without guessing at answers you haven't heard yet. Ask the whole frontier in one round: number each question and give your recommended answer. Then wait for the user's answers before the next round.

## Asking a round

If your toolset includes a structured multiple-choice question tool, prefer it over writing the round as markdown — it gives the user selectable options (plus a free-text/"Other" escape hatch) instead of a wall of text they have to answer in prose, and makes disagreeing with a recommendation a single click rather than something they have to compose. Per agent:

- **Claude Code**: `AskUserQuestion`. Pass the round's questions as its `questions` array (one entry per frontier question); make the recommended answer the first option, with genuine alternatives as the others.
- **OpenCode**: the `question` tool (header + question + options), same shape.
- **Codex CLI**: `request_user_input`, but **only when already in Plan Mode** — it errors in Default mode and in `codex exec`. Outside Plan Mode, fall back to the markdown format below.
- Any other agent, or Codex outside Plan Mode: fall back to the markdown format below.

If the user asks to go through questions one at a time instead of a full round, honor that — ask one question per turn (one tool call, or one markdown block) instead of batching the frontier.

Fallback markdown format for a round:

```
❓ **Q1** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>

---

❓ **Q2** - **<question title>**: <question body, might be multiple paragraphs, including multiple choices>

➡️ <your recommended answer>
```

Each round the user answers reshapes the tree: settled decisions push the frontier outward and unblock questions that depended on them. Recompute the frontier and ask the next round. A question whose answer depends on another question still open in this round belongs to a _later_ round, not this one.

Finding _facts_ is your job, never the user's. When a frontier question needs a fact from the environment (filesystem, tools, etc.), dispatch a sub-agent to find it; don't ask the user for anything you could look up yourself. Don't block on it: a running exploration is an unsettled prerequisite, so only the questions downstream of it wait for the sub-agent to report; ask the rest of the frontier now. The _decisions_ are the user's: put each to them and wait.

## Handling disagreement

Never silently proceed past a recommended answer the user pushed back on, and never re-ask it or quietly revert to your own suggestion in a later round — whatever they say replaces your recommendation as the settled decision. If their objection implies a new question that wasn't on the frontier yet (it reopens something you'd treated as settled, or has a consequence elsewhere in the tree), add that to the next round instead of deciding it yourself.

The session is done when the frontier is empty: every branch of the design tree visited, nothing left silently assumed. Do not act on it until the user confirms you have reached a shared understanding.
