# perf-skills Eval Guide

## Skill Under Evaluation

- **Skill name:** `perf`
- **Skill path:** `skills/perf/SKILL.md`
- **Evals path:** `skills/perf/evals/evals.json`
- **Workspace:** `perf-skills-workspace/` *(gitignored - outputs only)*

---

## Running Evals

### Full eval run (all test cases)

Tell Claude Code:

```
Run evals for skills/perf using skills/perf/evals/evals.json.
Save results to perf-skills-workspace/iteration-1/
```

Claude will:
1. Spawn a **with-skill** subagent - reads `SKILL.md`, then executes the prompt
2. Spawn a **without-skill** baseline subagent - same prompt, no skill context
3. Save outputs under `perf-skills-workspace/iteration-1/eval-{id}/`
4. Grade each assertion and write `eval_metadata.json` + `timing.json` per eval
5. Aggregate results into `perf-skills-workspace/iteration-1/benchmark.json`

### Single eval run

```
Run eval id=3 from skills/perf/evals/evals.json.
Save to perf-skills-workspace/iteration-1/eval-3/
```

---

## Output Directory Structure

```
perf-skills-workspace/
└── iteration-1/
    ├── eval-0/
    │   ├── with_skill/
    │   │   └── outputs/        ← full LLM response with skill context
    │   ├── without_skill/
    │   │   └── outputs/        ← full LLM response without skill context
    │   ├── eval_metadata.json  ← assertion grades, pass/fail, notes
    │   └── timing.json         ← latency for each subagent call
    ├── eval-1/
    │   └── ...
    └── benchmark.json          ← aggregate pass rates across all evals
```

### eval_metadata.json schema

```json
{
  "eval_id": 1,
  "category": "jmeter",
  "with_skill": {
    "passed": ["has-thread-group", "has-ramp-up", "has-http-sampler"],
    "failed": [],
    "notes": ""
  },
  "without_skill": {
    "passed": ["has-thread-group"],
    "failed": ["has-ramp-up", "has-http-sampler", "has-assertion"],
    "notes": "Missing ramp-up and assertion configuration without skill context"
  }
}
```

### benchmark.json schema

```json
{
  "iteration": 1,
  "total_evals": 10,
  "with_skill_pass_rate": 0.92,
  "without_skill_pass_rate": 0.54,
  "delta": 0.38,
  "categories": {
    "jmeter": { "with_skill": 1.0, "without_skill": 0.5 },
    "k6": { "with_skill": 1.0, "without_skill": 0.67 }
  }
}
```

---

## Description Optimization (one-time, after skill content is stable)

Run this to auto-optimize the `description` field in `SKILL.md` by testing trigger/no-trigger queries:

```bash
python -m scripts.run_loop \
  --eval-set skills/perf/evals/trigger-eval.json \
  --skill-path skills/perf \
  --model claude-sonnet-4-20250514 \
  --max-iterations 5 \
  --verbose
```

---

## Eval Categories

| Category | Evals | What it tests |
|---|---|---|
| `k6` | 1, 5, 7 | Diagnosis, executor choice, thresholds |
| `tool-selection` | 2, 12 | Recommending the right tool (Kafka and catalog-driven selection) |
| `jmeter` | 3 | OOM diagnosis, listener memory leak |
| `migration` | 4 | LoadRunner-to-k6 concept mapping |
| `locust` | 6 | catch_response silent failure |
| `artillery` | 8, 9 | ensure CI gating, arrivalRate vs concurrency |
| `llm-inference` | 10 | TTFT/TPOT metrics, benchmarking tools |
| `slo-capacity` | 11 | Error budgets, CI gating with burn rate |

---

## Assertion Writing Rules

- State in present tense: `"Output contains..."`, `"Response includes..."`
- Must be objectively verifiable - avoid subjective grades like "response is good"
- 2–4 assertions per eval is the target range
- Use unique `id` strings - they serve as the pass/fail key in `eval_metadata.json`

---

## What to Commit vs. Gitignore

```gitignore
perf-skills-workspace/    # eval run outputs - large, ephemeral
feedback.json             # personal review notes
```

Commit:
- `skills/perf/SKILL.md`
- `skills/perf/evals/evals.json`
- `.claude/CLAUDE.md`
