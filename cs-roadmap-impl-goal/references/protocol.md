# CodeStable Roadmap Goal Protocol

This file is copied into `.codestable/roadmap-goals/{slug}/protocol.md` and read by the `/goal` session. It is the operating manual for executing one approved CodeStable roadmap package.

## State Files

Read first:

- `goal-state.yaml`
- `goal-plan.md`
- `features/*.md` in the order listed by `goal-state.yaml`
- the referenced roadmap doc and items.yaml

`goal-state.yaml` must contain:

```yaml
roadmap: "{slug}"
status: ready-to-dispatch  # ready-to-dispatch | in-progress | blocked | complete
baseline_ref: "{git rev-parse HEAD or no-git}"
current_feature: 1
features:
  - slug: "{feature-slug}"
    feature_dir: ".codestable/features/YYYY-MM-DD-{feature-slug}"
    design: ".codestable/features/YYYY-MM-DD-{feature-slug}/{feature-slug}-design.md"
    checklist: ".codestable/features/YYYY-MM-DD-{feature-slug}/{feature-slug}-checklist.yaml"
    status: pending        # pending | implementing | accepted | blocked
```

Update this file after every feature boundary.

## Start

Print:

```text
CS_ROADMAP_GOAL_START
Roadmap: <slug>
Features: <count>
Baseline ref: <sha|no-git>
Plan: <goal-plan.md path>
Protocol: <protocol.md path>
```

Run preflight before changing code:

1. Union mandatory commands from every feature spec.
2. Run each once if cost is reasonable.
3. Record green/red in state and transcript.
4. If a command is red, classify as pre-existing, blocking, or unknown. Blocking/unknown requires user decision or a first safety-net fix if the roadmap explicitly includes one.

## Feature Loop

For each feature in state order:

1. Read the feature spec, design doc, checklist, roadmap item, and current code context.
2. Mark the feature `implementing`.
3. Print:

   ```text
   CS_ROADMAP_GOAL_FEATURE_START
   Feature: <N>/<total> <slug>
   Design: <path>
   Checklist: <path>
   Depends on: <list|none>
   Mandatory commands: <list>
   Evidence required: <list>
   ```

4. Run the `cs-feat-impl` workflow for this feature:
   - perform baseline precheck for relevant commands
   - execute checklist steps in order
   - update checklist step status immediately
   - collect step evidence
   - run step cleanliness checks
   - use the three-attempt failure recovery below for failed criteria
5. Run the `cs-feat-accept` workflow for this feature:
   - fill acceptance report
   - update checks to `passed`
   - update architecture / requirement / roadmap as required
   - run final feature audit against the original design/checklist
6. Print:

   ```text
   CS_ROADMAP_GOAL_FEATURE_VERIFY
   Feature: <slug>
   Implementation: pass|fail
   Acceptance: pass|fail
   Commands: <summary with exit codes>
   Deliverables: <present|missing summary>
   Cleanliness: <pass|fail summary>
   Roadmap item: <done|not-done>
   Knowledge candidates: <summary|none>
   ```

7. If pass, mark feature `accepted`, update `goal-state.yaml`, and print:

   ```text
   CS_ROADMAP_GOAL_FEATURE_DONE
   Feature <slug> accepted. State updated.
   ```

8. If the user has sent a new message, pause at the feature boundary and ask whether to resume, revise future feature specs, or stop.

## Failure Recovery

Apply this to implementation criteria, acceptance checks, mandatory commands, deliverables, and cleanliness failures.

### First failure: diagnose and retry

Print:

```text
CS_ROADMAP_GOAL_FAILURE_DIAGNOSE
Feature: <slug>
Failed item: <criterion/check/command>
Tried: <summary>
Hypothesis: <root cause>
Next: retry same item once
```

Retry only the failing item. Do not advance.

### Second failure: focused repair note

Write `.codestable/roadmap-goals/{slug}/features/{feature-slug}-repair.md`:

- failed item
- root cause
- allowed files / docs to change
- verification commands
- explicit no-scope-creep note

Execute only that repair, then re-run the original check.

### Third failure: handoff

Print:

```text
CS_ROADMAP_GOAL_HANDOFF
Feature: <slug>
Failed item: <item>
Attempts: <three summaries>
Suggested next move: <one line>
State: blocked
```

Update `goal-state.yaml` to `blocked` and stop. Do not print completion.

## Cleanliness Checks

For each feature, inspect the complete working tree since `baseline_ref` when available, otherwise inspect current diff:

- added debug output (`console.log`, `print`, `fmt.Println`, temporary logger)
- added temporary `TODO` / `FIXME` / `XXX`
- commented-out code
- unused imports
- files changed outside the feature scope

Any unexplained hit is a failed check. If the feature intentionally ships logging/debug behavior, the design or feature spec must declare the exception.

## Knowledge Writeback

After each accepted feature, list candidates but do not silently write them:

- environment / command facts likely needed by every future feature → `cs-note`
- reusable pitfalls or debugging patterns → `cs-learn`
- stable conventions / architectural decisions → `cs-decide`
- user/developer guide changes → `cs-guide`
- public API / command / component references → `cs-libdoc`

Acceptance may write required architecture/requirement/roadmap docs directly because those are part of the workflow. Other knowledge writes require user confirmation unless the active goal explicitly granted permission.

## Final Roadmap Audit

After the last feature is accepted, do not print completion yet.

Print:

```text
CS_ROADMAP_GOAL_AUDIT_START
Roadmap: <slug>
Features to verify: <count>
Commands to re-run: <deduplicated list>
```

Audit steps:

1. Re-read roadmap doc and items.yaml.
2. Verify every item is `done` or explicitly `dropped` with reason.
3. Re-read every feature design/checklist/acceptance report.
4. Re-run deduplicated mandatory commands once where practical.
5. Verify deliverables from repository facts: files, config keys, schema changes, routes, docs, roadmap state.
6. Check complete working tree: tracked, staged, unstaged, and untracked files.
7. Check cleanliness across the run.
8. Count `re-verified` vs `trust-prior-verify` items; mark screenshots/manual checks as trust-prior unless re-run.
9. Verify architecture / requirement / roadmap writebacks are present.
10. Verify knowledge candidates are either handled, explicitly deferred, or listed for user decision.

Print:

```text
CS_ROADMAP_GOAL_AUDIT_VERIFY
Roadmap items: <done>/<total>
Commands: <summary>
Deliverables: <present>/<total> present
Cleanliness: pass|fail
Writebacks: pass|fail
Knowledge exits: handled|deferred|none
Coverage: <re-verified> re-verified / <trust-prior> trust-prior
```

If gaps are found, write `.codestable/roadmap-goals/{slug}/audit-repair-<round>.md`, repair only the failing audit items, and rerun the audit. After three failed audit rounds, print `CS_ROADMAP_GOAL_HANDOFF`, mark state `blocked`, and stop.

If clean, mark state `complete` and print:

```text
CS_ROADMAP_GOAL_AUDIT_COMPLETE
Roadmap: <slug>
Audit rounds: <count>
Coverage: <re-verified> re-verified / <trust-prior> trust-prior
```

Then print:

```text
CS_ROADMAP_GOAL_COMPLETE
Roadmap <slug> complete.
Features accepted: <count>
Final audit passed.
Manual follow-up: <items|none>
```

Only `CS_ROADMAP_GOAL_COMPLETE` satisfies the `/goal` condition.
