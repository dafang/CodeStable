---
name: cs-roadmap-impl-goal
description: CodeStable end-to-end roadmap execution planner. Use when the user gives a large software request and wants it turned into a CodeStable roadmap, decomposed into feature designs/checklists, then executed feature-by-feature through cs-feat-design, cs-feat-impl, and cs-feat-accept under one ready-to-paste /goal command. Trigger on requests like "做完整个 roadmap", "把这个大需求拆完并自动推进", "生成 goal 指令跑完整个 CodeStable roadmap", "像 supergoal 一样规划并执行这些 features", or when the user wants autonomous follow-through after one roadmap/design review gate.
---

# cs-roadmap-impl-goal

## Intent

Create a single CodeStable execution package for a large request:

1. Use `cs-roadmap` to create or update one roadmap.
2. Generate all planned feature designs and checklists up front with `cs-feat-design`.
3. Wait for one explicit user confirmation over the roadmap + all designs.
4. Print one ready-to-paste `/goal` command that runs the approved feature chain: for each feature, run `cs-feat-impl`, then `cs-feat-accept`, update roadmap state, continue to the next feature, and finish with a final roadmap audit.

Do not put the full task body into the `/goal` argument. Long instructions live in files under `.codestable/roadmap-goals/{slug}/`; the `/goal` condition only names those files and the terminal transcript markers.

## Required Inputs

Before writing artifacts, collect only true gaps:

- The large request / outcome the user wants.
- Scope cut-line and explicit non-goals.
- Any product priority that cannot be inferred from dependencies.
- External constraints that code recon cannot discover.

For brownfield repos, prefer repo recon over questions. Read `.codestable/attention.md` first; if missing, stop and ask the user to run `cs-onboard` or repair the CodeStable skeleton.

## Artifact Layout

Create a per-run directory:

```text
.codestable/roadmap-goals/{roadmap-slug}/
├── goal-plan.md          # user-facing summary + assumptions + risks
├── goal-state.yaml       # live execution state for the /goal session
├── protocol.md           # copied/adapted from this skill's references/protocol.md
└── features/
    └── {feature-slug}.md # per-feature execution spec pointing at design/checklist paths
```

The normal CodeStable artifacts remain in their standard homes:

- `.codestable/roadmap/{slug}/{slug}-roadmap.md`
- `.codestable/roadmap/{slug}/{slug}-items.yaml`
- `.codestable/features/{YYYY-MM-DD}-{feature-slug}/{feature-slug}-design.md`
- `.codestable/features/{YYYY-MM-DD}-{feature-slug}/{feature-slug}-checklist.yaml`

## Planning Workflow

### 1. Recon

Read:

- `.codestable/attention.md`
- relevant requirements / architecture / compound docs
- existing roadmap docs to avoid duplicates
- existing features with related terms
- code and validation commands needed to assess baseline risk

Summarize to the user in a few lines: relevant modules, validation commands, likely risks, and any unresolved input gaps.

### 2. Create Or Update Roadmap

Run the `cs-roadmap` workflow in spirit:

- Write the full roadmap doc and items.yaml.
- Include module split, interface contracts, sub-feature DAG, minimal loop, safety-net/polish/harden needs, risks, validation entrypoints, deliverables, and knowledge writeback candidates.
- Validate items.yaml.

Do not proceed with feature designs until the roadmap is internally coherent: no circular dependencies, no vague contracts, no unmeasurable feature descriptions.

### 3. Generate All Feature Designs Up Front

For every `planned` roadmap item whose dependencies are satisfiable in roadmap order:

- Create the normal feature directory.
- Write `{slug}-design.md` using `cs-feat-design` rules.
- Write `{slug}-checklist.yaml`.
- Set design frontmatter `roadmap` and `roadmap_item`.
- Update roadmap items.yaml using the existing `cs-feat-design` convention: set the item to `in-progress` and fill `feature`. Execution order is then tracked by `goal-state.yaml`; do not infer execution completion from `in-progress`.

Each design must include:

- mandatory validation commands / baseline risk
- deliverables
- acceptance scenarios with evidence type
- cleanup/cleanliness expectations
- implementation steps with independent exit signals

### 4. Write Goal Specs

Copy `references/protocol.md` into `.codestable/roadmap-goals/{slug}/protocol.md`, then write:

- `goal-plan.md`: roadmap path, feature order, risks, assumptions, mandatory command set, user review notes.
- `goal-state.yaml`: `status: ready-to-dispatch`, current feature index, baseline ref, feature list with `pending` status.
- `features/{feature-slug}.md`: one execution spec per feature with paths, dependencies, commands, acceptance evidence, deliverables, and cleanup rules.

### 5. Self-Critique Before Review

Before showing the user the plan, run one self-critique pass and fix issues in place:

1. Are all feature acceptance criteria measurable?
2. Is any feature secretly two features?
3. Does any weak dependency threaten many later features?
4. Are baseline risks and mandatory commands explicit?
5. Can final audit verify deliverables from repository facts?

### 6. User Review Gate

Show one concise review:

- roadmap slug and paths
- feature order with one-line deliverables
- assumptions to correct
- top risks and mitigations
- baseline/preflight commands
- self-critique findings
- artifact paths

Wait for explicit approval. If the user changes scope, update roadmap, designs, goal specs, and state before generating the `/goal`.

### 7. Print Ready-To-Paste Goal

After approval, print a single fenced `/goal` command:

```text
/goal "Execute the CodeStable roadmap-goal package at .codestable/roadmap-goals/{slug}. Read protocol.md and goal-state.yaml first. For each feature spec in order, run implementation and acceptance using the referenced CodeStable design/checklist files, update state, and print CS_ROADMAP_GOAL_FEATURE_DONE. After all features, run the final roadmap audit in protocol.md. Done only when CS_ROADMAP_GOAL_COMPLETE appears with all features accepted, final audit passed, and no CS_ROADMAP_GOAL_HANDOFF this run."
```

Then stop. Slash commands only run from user input; the user must paste it.

## Execution Markers

The generated protocol must require these transcript markers:

- `CS_ROADMAP_GOAL_START`
- `CS_ROADMAP_GOAL_FEATURE_START`
- `CS_ROADMAP_GOAL_FEATURE_VERIFY`
- `CS_ROADMAP_GOAL_FEATURE_DONE`
- `CS_ROADMAP_GOAL_AUDIT_START`
- `CS_ROADMAP_GOAL_AUDIT_VERIFY`
- `CS_ROADMAP_GOAL_AUDIT_COMPLETE`
- `CS_ROADMAP_GOAL_COMPLETE`
- `CS_ROADMAP_GOAL_HANDOFF`

Use CodeStable wording in artifacts; do not mention external workflow names or copy external marker names.

## Hard Boundaries

- One roadmap per run.
- One approval gate before autonomous execution.
- Do not execute the `/goal` yourself.
- Do not skip `cs-feat-accept`; roadmap state is only complete after acceptance updates architecture/requirement/roadmap docs.
- Do not mark the roadmap complete until final audit verifies every item is `done` or intentionally `dropped`.
- Do not commit unless the user explicitly asks during the goal session.

## Resources

Read `references/protocol.md` when writing `.codestable/roadmap-goals/{slug}/protocol.md`.
