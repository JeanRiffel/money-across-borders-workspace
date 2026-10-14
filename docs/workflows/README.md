# Engineering workflows

A **workflow** is a repeatable procedure for one kind of work: its phases, who owns each phase,
when a phase may start and end, and what evidence it leaves behind. Workflows let several agents
(and humans) hand work to each other without losing context or rigor.

## How workflows relate to AGENTS.md

- [`AGENTS.md`](../../AGENTS.md) is the entry point. It holds the workspace's global rules (safety,
  approvals, Git, conventions) and says **when** to follow each workflow.
- The files in this folder hold the **procedures**. They don't repeat the global rules; they link to
  them.
- If a workflow and `AGENTS.md` disagree, `AGENTS.md` wins. Fix the workflow, or ask a human.
- Each code repository's own `AGENTS.md` still decides commands, architecture and invariants inside
  that repository.

## How agents use them

1. Read `AGENTS.md`. Its "Workflows" section names the workflow for the work at hand.
2. Read that workflow file in full before starting any of its phases.
3. Record the evidence where the workflow says, usually in the task file.

## Available workflows

| Workflow | File | Applies to |
|---|---|---|
| Test-Driven Development | [`tdd.md`](tdd.md) | Every implementation task in `features/<feature-slug>/03-tasks/` |

The feature lifecycle (Discovery → Specifications → Tasks) is still described in
[`AGENTS.md`](../../AGENTS.md#feature-lifecycle). It may move here later.
