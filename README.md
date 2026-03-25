# perf-skills

> *"The more you sweat in training, the less you bleed in battle."* — Richard Marcinko
>
> The more relevant skills your AI loads during development, the fewer fires you fight in production.

## What Is This?

`perf-skills` is a structured knowledge base that gives AI coding assistants deep, opinionated expertise in performance testing. It works with any AI tool that supports custom instructions, rules, or skill files — helping you plan, script, execute, and analyze load tests using any major tool.

### Supported Tools

| Open Source | Commercial |
|---|---|
| Apache JMeter | NeoLoad (Tricentis) |
| k6 (Grafana Labs) | LoadRunner (OpenText) |
| Gatling | OctoPerf (SaaS) |
| Locust | |

### Topics Covered

- **Workload design** — concurrency models, Little's Law, transaction mix, SLA targets
- **Test data** — parameterization, CSV feeds, synthetic data, data isolation
- **Script generation** — correlation, assertions, error handling, session management
- **Test execution** — local, distributed, CI/CD pipelines, cloud execution
- **Results analysis** — percentiles, bottleneck identification, trend comparison, reporting
- **Observability** — APM integration, Prometheus/Grafana, distributed tracing, log correlation
- **Production testing** — canary, shadow traffic, chaos engineering, safety controls
- **Protocol testing** — gRPC, GraphQL, WebSocket/SSE, Kafka/message queues
- **Database testing** — JDBC load testing, connection pools, query concurrency, replication lag
- **Modern architectures** — microservices, Kubernetes (HPA, service mesh), serverless (cold starts), frontend (Core Web Vitals)

## Compatible AI Coding Assistants

| Tool | Integration Method | Setup |
|---|---|---|
| **Windsurf (Cascade)** | Skills | Copy to skills directory — auto-triggers on perf questions |
| **Cursor** | Rules / Docs | Add as project rules or index via `@Docs` |
| **Claude Code** | CLAUDE.md / Custom instructions | Reference files in `CLAUDE.md` or feed as context |
| **Cline** | Custom instructions / `.clinerules` | Add to `.clinerules` or workspace instructions |
| **Roo Code** | Custom instructions / Rules | Add as workspace rules or custom instructions |
| **Aider** | Conventions / Chat context | Add to `.aider.conf.yml` conventions or `/read` files |
| **OpenCode** | Custom instructions | Add to project-level instructions |
| **Antigravity** | Context files | Add as context / knowledge files |
| **Pochi** | Custom instructions | Reference files in project instructions |
| **GitHub Copilot** | Custom instructions / `.github/copilot-instructions.md` | Reference in repo-level instructions |

## Installation

### Claude Code Plugin (Recommended)

**Add the marketplace and install:**
```bash
/plugin marketplace add QAInsights/perf-skills
/plugin install perf-skills@perf-skills
```

After install, the `/perf-skills` skill is available and auto-activates on performance testing questions.

**Update to latest version:**
```bash
/plugin marketplace update
```

### Install as Skills (npx)

```bash
npx skills add QAInsights/perf-skills
```

### Install as Skills (Manual)

```bash
# Clone and copy to Claude skills directory
git clone https://github.com/QAInsights/perf-skills.git
cd perf-skills
cp -r perf-skills/ ~/.claude/skills/
```

### Windsurf (Skills)

1. Copy the `perf-skills/` directory into your Windsurf skills location.
2. The skill auto-triggers when you ask about performance testing, load testing, or any supported tool.

### Cursor (Rules / Docs)

**Option A — Project Rules:**
1. Create `.cursor/rules/perf-skills.mdc` in your project root.
2. Add the content of `SKILL.md` as the rule, with file references pointing to the `references/` directory.

**Option B — @Docs indexing:**
1. Open Cursor Settings → Features → Docs.
2. Add the `perf-skills/` directory as a doc source.
3. Reference with `@Docs perf-skills` in chat.

### Claude Code (CLAUDE.md)

1. Copy the `perf-skills/` directory into your project.
2. In your `CLAUDE.md`, add:
```markdown
For performance testing questions, read `perf-skills/SKILL.md` for routing,
then load the relevant reference file(s) from `perf-skills/references/`.
```

### Cline / Roo Code

1. Copy the `perf-skills/` directory into your project.
2. Add to your custom instructions or `.clinerules`:
```
For performance testing guidance, consult the perf-skills knowledge base.
Start with perf-skills/SKILL.md for routing to the correct reference file.
```

### Aider

1. Copy the `perf-skills/` directory into your project.
2. Use `/read perf-skills/SKILL.md` to load the routing file.
3. Then `/read` the specific reference file(s) relevant to your question.

### GitHub Copilot

1. Copy the `perf-skills/` directory into your project.
2. In `.github/copilot-instructions.md`, add:
```markdown
For performance testing questions, reference the perf-skills knowledge base.
Start with perf-skills/SKILL.md, then load relevant files from perf-skills/references/.
```

### OpenCode / Antigravity / Pochi

1. Copy the `perf-skills/` directory into your project.
2. Add to your project-level custom instructions or context files:
```
For performance testing guidance, consult the perf-skills knowledge base.
Start with perf-skills/SKILL.md for routing to the correct reference file.
```

### Any Other AI Tool

The skill is plain markdown files. Any AI tool that can read files or accept custom instructions can use it:
1. Point the tool to `SKILL.md` as the entry point.
2. Let the Reference Map in `SKILL.md` guide which file(s) to load.

### As a Standalone Knowledge Base

Browse the markdown files directly — they're self-contained references useful even without an AI assistant.

## File Structure

```
perf-skills/                              # Repository root
├── .claude-plugin/
│   └── marketplace.json                  # Claude Code marketplace catalog
├── README.md
├── LICENSE.md
└── perf-skills/                          # Plugin / Skill directory
    ├── .claude-plugin/
    │   └── plugin.json                   # Claude Code plugin manifest
    ├── SKILL.md                          # Entry point — tool selection, lifecycle, key principles
    └── references/
        ├── tools/                        # Tool-specific syntax and configuration
        │   ├── jmeter.md                 # JMeter 5.6+ — samplers, extractors, plugins, Groovy
        │   ├── k6.md                     # k6 v0.50+ — executors, checks, thresholds, modules
        │   ├── gatling.md                # Gatling 3.10+ — Scala/Java DSL, feeders, injection
        │   ├── locust.md                 # Locust 2.20+ — Python scripts, events, FastHttpUser
        │   ├── neoload.md                # NeoLoad — GUI workflow, CLI, API execution
        │   ├── loadrunner.md             # LoadRunner — VuGen, protocols, Controller scenarios
        │   └── octoperf.md              # OctoPerf — JMeter-based SaaS, HAR import, cloud
        └── topics/                       # Cross-cutting concepts (tool-agnostic)
            ├── workload-design.md        # Concurrency models, load profiles, Little's Law
            ├── test-data.md              # CSV, DB seeding, Faker, data isolation patterns
            ├── script-generation.md      # Correlation, assertions, error handling, naming
            ├── test-execution.md         # Distributed, CI/CD (GitHub Actions, GitLab, Jenkins)
            ├── results-analysis.md       # Percentiles, bottleneck framework, reporting
            ├── observability.md          # APM, Prometheus, Grafana, tracing, JVM metrics
            ├── production-testing.md     # Canary, shadow traffic, chaos, safety controls
            ├── protocol-testing.md       # gRPC, GraphQL, WebSocket, Kafka/message queues
            ├── database-testing.md       # JDBC, connection pools, slow queries, deadlocks
            └── modern-architectures.md   # Microservices, K8s, serverless, browser/Web Vitals
```

## How the Skill Works

### Routing Logic

`SKILL.md` acts as the entry point and router. It contains:

1. **Loading Priority Rules** — tells the AI which file(s) to load based on the user's question (never all at once).
2. **Reference Map** — maps user intent to the right file.
3. **Protocol Routing Table** — maps protocols (gRPC, GraphQL, etc.) to recommended tools and reference files.
4. **Tool Selection Matrix** — helps recommend a tool when the user hasn't chosen one.
5. **Key Principles** — the single source of truth for cross-cutting best practices (assertions, think time, parameterization, correlation).

### Design Principles

- **Token-efficient**: Tool files contain only tool-specific syntax. Cross-cutting concepts live in topic files. No duplication.
- **Selective loading**: The AI loads 1–2 files per question, not the entire knowledge base.
- **Single source of truth**: Each concept is defined in exactly one place. Tool files cross-reference topic files for shared concepts.
- **Opinionated**: The skill prescribes best practices, not just documentation. It tells you what to do, not just what's possible.

## Example Questions This Skill Handles

| Question | Files Loaded |
|---|---|
| "Help me write a k6 load test for our REST API" | `k6.md` |
| "How should I design the workload for our e-commerce app?" | `workload-design.md` |
| "Set up JMeter in our GitHub Actions pipeline" | `test-execution.md` + `jmeter.md` |
| "Our p95 latency is spiking at 500 VUs — how do I debug?" | `results-analysis.md` + `observability.md` |
| "How do I load test a gRPC service?" | `protocol-testing.md` + `k6.md` |
| "What tool should I use? We're a Python team." | `SKILL.md` (Tool Selection Matrix) |
| "Test our Kafka consumer throughput" | `protocol-testing.md` |
| "Validate our K8s HPA scales correctly under load" | `modern-architectures.md` |
| "Load test our PostgreSQL connection pool" | `database-testing.md` + `jmeter.md` |

## Contributing

To add or update content:

1. **Tool-specific content** goes in `references/tools/<tool>.md` — syntax, config, tool-unique tips only.
2. **Cross-cutting concepts** go in `references/topics/<topic>.md` — patterns that apply across tools.
3. **Never duplicate** — if a concept exists in a topic file, tool files should cross-reference it, not restate it.
4. **Update SKILL.md** if you add a new file — add it to the Reference Map and Protocol Routing Table if applicable.
5. **Add a version indicator** (`> Targets: ...`) to new tool files.

## License

See repository license.
