---
name: bypass-pdca
description: |
  Plan file-driven autonomous PDCA orchestrator. Extracts phases/steps from plan files, sequences bkit agents, and executes the full PDCA cycle.
  Supports multi-plan sequential execution, session resume, and wave-based parallelism.
  Not for simple bug fixes, planless work, or single file edits.
triggers:
  - bypass-pdca
  - run plan
  - plan execute
  - execute plans
argument-hint: "[plan-file1] [plan-file2] ... or [feature-name]"
disable-model-invocation: true
allowed-tools:
  - Read       # plan/status file read
  - Write      # status.json, analysis docs
  - Edit       # implementation edit
  - Grep       # code pattern search
  - Glob       # file discovery
  - Bash       # build, test, git
  - Agent      # cto-lead team delegation
  - Skill      # /simplify invocation
  - WebSearch  # tech reference lookup
effort: max
---

# bypass-pdca: Smart Orchestrator

> Composite workflow orchestrator. A composite skill exempt from single-responsibility and minimum allowed-tools rules.

## Prerequisites

- **bkit plugin required**: Invokes the `bkit:cto-lead` agent (which internally operates a team of gap-detector, report-generator, etc.) and the `/simplify` skill.
- **Runs autonomously to completion without user confirmation.**
- **Main Session execution**: CC blocks nested agent spawning. Running as a Main Session Skill keeps Agent calls at 1-level.

```
Main Session + bypass-pdca SKILL
  └── Agent(bkit:cto-lead)         -- 1-level OK
        ├── Task(gap-detector)     -- internal to cto-lead
        └── Task(report-generator) -- internal to cto-lead
```

## Terminology

| Term | Meaning | Example |
|------|---------|---------|
| **Stage** | Execution unit of this skill (fixed) | parse, cto, ralph-loop, commit, archive |
| **Plan Phase** | User-defined Phase inside the plan file | Phase A: Static Analysis |
| **Plan Step** | Detailed task item inside a Plan Phase | Step 1.1: Create Table |

## Input Interpretation

`$ARGUMENTS` is interpreted by the following rules:

| Pattern | Example | Behavior |
|---------|---------|----------|
| Multiple files | `plan-a.md plan-b.md` | Each executed sequentially as a separate plan |
| Glob | `docs/01-plan/features/*.plan.md` | Collect all matching files |
| Single file | `my-feature.plan.md` | Read directly |
| Feature name | `my-feature` | `docs/01-plan/features/$NAME.plan.md` -> `docs/01-plan/$NAME.md` -> Glob `**/*$NAME*plan*` |
| No arguments | | Print error and exit |

## Execution Flow

### Single Plan

```
1. Plan Parse: extract Phases + dependency graph + initialize status.json
2. CTO delegation: Agent(cto-lead) — design + do + analyze + iterate(100%) + report
3. ralph-loop: impact-scan + Skill(/simplify)
4. Commit
5. Archive: /pdca archive
```

### Multi-Plan

Read all plans first, run dependency analysis (apply the common rules below), then execute sequentially in priority order. Each plan goes through an independent cycle (1-5), with independent commits + independent archives. No parallel execution across plans.

After all plans complete, emit a consolidated report.

### Dependency Analysis (Common Rules)

The same rules apply both across Phases and across plans:
- **Explicit**: direct references such as "after XX is complete," "requires XX"
- **Implicit**: relationships that reference/modify artifacts
- **Independent**: no dependency relation (eligible for Wave parallelism)
- **Circular dependency**: resolve directly after analysis (split/merge/re-sequence)

## Stage Details

### 1. Plan Parse

1. Read the entire plan file.
2. Extract Phase/Step using these patterns:
   - `## Phase N:` / `## Phase N.` / `## Phase N -`
   - `### Step N.M:` / `### Step N.M.`
   - `## Stage N:` / `## N Stage`
   - YAML `phases:`/`steps:` keys
   - JSON `phases`/`steps` arrays
   - `## N. ` numbered section form
   - Checklist: `- [ ] Phase N:`
3. Extract from each Phase/Step: ID, name, description, dependencies, artifacts
4. Build the dependency graph (apply the common rules).
5. Create `.ww-w-ai/devtools/bypass-pdca/status.json` (see the Progress Tracking section for structure).
   - If `.ww-w-ai/devtools/bypass-pdca/` does not exist, create it via `mkdir -p`
   - If `.gitignore` does not contain `.ww-w-ai/`, add it
   - `currentStage` = `"cto"`

### 2. CTO Delegation (design + do + analyze + iterate + report)

Delegate the full PDCA cycle to the cto-lead team. bypass-pdca only passes the plan and context and receives the result.

| Passed Item | Content |
|-------------|---------|
| Plan file | Full content |
| Phase/Step list | Extraction result + dependency graph |
| Project context | Reference CLAUDE.md |
| Document output paths | `docs/02-design/`, `docs/03-analysis/`, `docs/04-report/` |
| Parallelism hint | Recommend Wave-based parallel execution for independent Phases |
| matchRate target | == 100% (see Iterate Gate below) |
| Execution order | Generate design docs -> do (implement) -> gap analysis -> iterate -> generate analysis docs -> generate report docs. Generate each stage's documents before proceeding to the next stage. |
| Immediate status.json recording | Update the `documents` field in status.json immediately when each document is generated. Record **upon creation**, not batched after completion. |

#### Iterate Gate (Mandatory)

After gap analysis, verify the matchRate. **Do not apply the bkit default 90% threshold.**
This skill's matchRate target is 100%, and this gate takes precedence over bkit's default rule.

| Condition | Action |
|-----------|--------|
| matchRate == 100% | Skip iterate, proceed to the next stage (analysis document) |
| matchRate < 100% | Run pdca-iterator to close the gap, then re-analyze |
| Still < 100% after 5 iterations | Record the current matchRate and proceed to the next stage (force termination) |

Do not proceed to analysis document generation until matchRate == 100% or 5 iterations are exhausted.

**cto-lead team internal flow:**
```
Generate design docs (design) -> record status.json documents.design
  -> Implement (do) — Wave-based parallel
  -> Gap analysis (analyze) — Task(gap-detector)
  -> [Iterate Gate] matchRate < 100% -> iterate -> re-analyze (up to 5 times)
  -> Generate analysis docs (analysis) -> record status.json documents.analysis
  -> Generate report (report) -> record status.json documents.report
```

**Expected artifacts:** design docs + implementation complete + analysis docs + report.

After completion, update status.json: `currentStage` = `"ralph-loop"`, `matchRate`, per-Phase `status`.

### 3. ralph-loop (impact-scan -> simplify)

After the cto-lead team's PDCA cycle completes, this stage explores out-of-plan impact and cleans up the code.

1. **impact-scan**: The CTO team only implements within plan scope. This stage explores and fixes surrounding code that is affected by the implementation but not covered by the plan.
   - Performance impact: whether callers of the changed code experience performance degradation
   - Security impact: vulnerabilities due to new input paths/permission changes
   - Related-code side effects: impact on call sites of changed functions/classes
   - Existing test impact: run the test suite and check for breakage
   - Exploration methods: Grep call sites, trace imports backward, run tests
2. **simplify**: Invoke the `/simplify` skill to review and clean up the final code for reusability, quality, and efficiency.

### 4. Commit

- Update status.json: `currentStage` = `"completed"`, final `matchRate`, `completedAt`.
- Include the plan file name + completed Phase list in the commit message.
- Direct pushes to main/develop are forbidden (use a feature branch).
- Multi-plan: independent commit per plan.

### 5. Archive

- Invoke `/pdca archive {feature}`.
- The archive is handled without a separate commit.
- Update status.json: immediately record the archive path in `documents.archive`, `currentStage` = `"archived"`.
- If the archive fails, print the error and keep `currentStage` as `"completed"`.
- Multi-plan: archive immediately after the commit and before the next plan starts.

## Progress Tracking (status.json)

File path: `.ww-w-ai/devtools/bypass-pdca/status.json`

```json
{
  "plans": [{
    "file": "docs/01-plan/features/my-feature.plan.md",
    "feature": "my-feature",
    "currentStage": "cto|ralph-loop|completed|archived|failed",
    "phases": [
      { "id": "Phase 1", "name": "...", "status": "pending|in-progress|completed|failed|blocked", "retries": 0 }
    ],
    "documents": { "design": null, "analysis": null, "report": null, "archive": null },
    "matchRate": null,
    "startedAt": "ISO8601",
    "completedAt": null
  }],
  "executionOrder": [],
  "startedAt": "ISO8601"
}
```

### Update Timings

| Timing | Updated Fields |
|--------|----------------|
| Plan Parse complete | Create status.json, `currentStage` = `"cto"` |
| CTO: when a document is generated | `documents.design`, `documents.analysis`, `documents.report` recorded immediately |
| CTO delegation complete | `currentStage` = `"ralph-loop"`, `matchRate`, per-Phase `status` |
| Commit | `currentStage` = `"completed"`, `completedAt` |
| Archive | `documents.archive` recorded immediately, `currentStage` = `"archived"` |

## Session Resume

If `.ww-w-ai/devtools/bypass-pdca/status.json` exists:
1. Read the file.
2. Find the first plan whose `currentStage` is not `completed`/`archived`/`failed`.
3. Resume from that plan's last Stage.
4. Skip Phases in `completed` state.

## Error Handling

| Situation | Response | Retries |
|-----------|----------|:-------:|
| Plan file not found | Print error + exit | None |
| Phase extraction failed | Log plan content + exit | None |
| cto-lead failure (including internal gap-detector/iterator) | Analyze the error, then re-run | Up to 3 |
| matchRate not reached after 5 iterations | cto-lead logs current state + generates a report | None |
| status.json write failure | Recreate .ww-w-ai/ directory + retry | 1 |

On retry, increment the Phase's `retries` field and print the error cause.

## Constraints

- Cannot run without a plan file. If you only need implementation without a plan, write the code directly.
- PM/Plan authoring is not included. Focus is on executing an already-written plan.
