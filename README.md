# ⚡ Performance Testing Skills (`perf-skills`)

> **Turn your AI assistant into a senior performance engineer.** An opinionated, token-optimized knowledge base for planning, scripting, executing, and analyzing load tests across any major tool.

> *"The more you sweat in training, the less you bleed in battle."* - Richard Marcinko  
> *"The more relevant skills your AI loads during development, the fewer fires you fight in production."*

---

## 📖 Table of Contents

- [🔍 What is perf-skills?](#-what-is-perf-skills)
- [📦 Supported Tools & Protocols](#-supported-tools--protocols)
- [⛓️ Dynamic Value Correlation](#️-dynamic-value-correlation)
- [🛠️ Compatible AI Assistants](#️-compatible-ai-assistants)
- [🚀 Installation & Setup](#-installation--setup)
- [📂 File Structure & Routing](#-file-structure--routing)
- [💡 How the Skill Works](#-how-the-skill-works)
- [🤝 Contributing & License](#-contributing--license)

---

## 🔍 What is perf-skills?

`perf-skills` is a highly structured knowledge base designed specifically for AI coding assistants (like Claude, Cursor, Windsurf, etc.). It injects deep, expert-level performance testing context on-demand, allowing your AI to generate correct scripts, follow best practices, and debug bottlenecks without bloating its context window.

---

## 📦 Supported Tools & Protocols

| Category | Supported Technologies |
| :--- | :--- |
| **Open Source** | Apache JMeter, k6 (Grafana Labs), Gatling, Locust |
| **Commercial/SaaS** | NeoLoad (Tricentis), LoadRunner (OpenText), OctoPerf |
| **Protocols** | HTTP/REST, gRPC, GraphQL, WebSocket, SSE, JDBC, SOAP, Kafka/MQ, Citrix, SAP |

---

## ⛓️ Dynamic Value Correlation

A major highlight of this library is the **Dynamic Value Correlation** engine ([correlation.md](skills/perf/references/topics/correlation.md)). Correlation is often the most fragile part of performance scripting; `perf-skills` equips your AI with robust strategies to handle it flawlessly:

- **The Correlation Workflow**: Standardizes identifying, extracting, injecting, and verifying dynamic tokens.
- **Advanced Scanning**: Strategies for Next.js/React SSR hydration payloads (`__NEXT_DATA__`), SPA client-side token retrieval, runtime hooking, and Swagger contract mapping.
- **Robust vs. Brittle Regex**: Guides your AI to write expressions that don't break on minor HTML structure modifications.
- **Tech Stack Rules**: Pre-defined extraction rules for enterprise environments:
  - **ASP.NET**: ViewState, RequestVerificationToken, EventValidation.
  - **Java**: JSF ViewState, Spring Security `_csrf`, JSESSIONID.
  - **SAP & Oracle**: NetWeaver `sap-contextid`, ADF `_afrLoop`.
  - **OAuth 2.0 / SAML**: SAMLResponse, code, state tokens.
  - **Modern SSR/SPA**: Hydration states, Laravel `_token`, Django `csrfmiddlewaretoken`, Rails `authenticity_token`.
- **Diagnostic Playbook**: Step-by-step troubleshooting for URL-encoding, base64 url-safe transforms, JSON-escaping, and thread safety.

---

## 🛠️ Compatible AI Assistants

`perf-skills` works seamlessly with any modern AI development tool:

| AI Assistant | Integration Method |
| :--- | :--- |
| **Claude Code** | Native Plugin or `CLAUDE.md` context referencing |
| **Windsurf (Cascade)** | Local Skill configuration |
| **Cursor** | Custom rules `.cursor/rules/` or index via `@Docs` |
| **Cline / Roo Code** | Workspace instructions / custom rules |
| **Aider** | Conventions file or `/read` context loading |
| **GitHub Copilot** | Repository-level custom instructions |
| **Antigravity / Pochi** | Context files / instruction references |

---

## 🚀 Installation & Setup

### Claude Code Plugin (Recommended)

To install as a Claude Code plugin, run:
```bash
/plugin marketplace add QAInsights/perf-skills
/plugin install perf@qainsights
/reload-plugins
```

Once installed, the `/perf` skill will be available and automatically load when you ask performance-related questions.

To update to the latest version:
```bash
/plugin marketplace update
/plugin uninstall perf
/plugin install perf@qainsights
/reload-plugins
```

---

### Install as local Skills (npx)

If you are using a tool that supports `npx skills`, add it directly:
```bash
npx skills add QAInsights/perf-skills
```

---

### Setup for Other Editors

<details>
<summary><b>🌊 Windsurf (Cascade)</b></summary>
<br>

1. Clone this repository:
   ```bash
   git clone https://github.com/QAInsights/perf-skills.git
   ```
2. Copy the skill directory:
   ```bash
   cp -r perf-skills/skills/perf ~/.windsurf/skills/
   ```
The skill will auto-trigger when you ask about performance testing, load testing, or any supported tool.
</details>

<details>
<summary><b>🎯 Cursor</b></summary>
<br>

**Option A (Project Rules):**
1. Copy [SKILL.md](skills/perf/SKILL.md) content into a rule file: `.cursor/rules/perf.mdc`.
2. Configure rules to trigger on file paths matching performance scripts or performance-related prompts.

**Option B (@Docs indexing):**
1. Open Cursor Settings → Features → Docs.
2. Add the path to `skills/perf/` directory as a doc source.
3. Reference with `@Docs perf` in your prompts.
</details>

<details>
<summary><b>🤖 Claude Code (CLAUDE.md)</b></summary>
<br>

1. Copy the `skills/perf` directory into your project root:
   ```bash
   cp -r perf-skills/skills/perf ./perf
   ```
2. In your `CLAUDE.md`, add:
   ```markdown
   For performance testing questions, read `perf/SKILL.md` for routing, then load the relevant reference files from `perf/references/`.
   ```
</details>

<details>
<summary><b> Cline / Roo Code</b></summary>
<br>

1. Copy the `skills/perf` directory into your project root.
2. In your `.clinerules` or custom instructions, append:
   ```text
   For performance testing guidance, consult the perf knowledge base starting at perf/SKILL.md.
   ```
</details>

<details>
<summary><b>⚓ Aider</b></summary>
<br>

1. Copy the `skills/perf` directory into your project root.
2. Tell Aider to read the entrypoint:
   ```bash
   /read perf/SKILL.md
   ```
3. Ask your question. Aider will load the corresponding tool/topic files from `perf/references/`.
</details>

---

## 📂 File Structure & Routing

Here is how the knowledge base is organized. The entry point [SKILL.md](skills/perf/SKILL.md) routes queries to specific reference guides to keep prompt sizes minimal.

```text
perf-skills/
├── .claude-plugin/
│   ├── marketplace.json                  # Claude Code marketplace catalog
│   └── plugin.json                       # Claude Code plugin manifest
├── README.md
├── LICENSE.md
└── skills/
    └── perf/                             # Skill module
        ├── SKILL.md                      # [Entry Point] Routing, Key Principles
        └── references/
            ├── tools/                    # Tool-specific syntax & configs
            │   ├── jmeter.md             # Apache JMeter (Groovy, Samplers, Extractors)
            │   ├── k6.md                 # k6 JS/TS scripting, thresholds, VU options
            │   ├── gatling.md            # Gatling Scala/Java DSL & feeders
            │   ├── locust.md             # Locust Python scripting & FastHttpUser
            │   ├── neoload.md            # NeoLoad GUI configs & API execution
            │   ├── loadrunner.md         # LoadRunner VuGen C scripting
            │   └── octoperf.md           # OctoPerf JMeter-based cloud execution
            └── topics/                   # Cross-cutting topics (Tool-agnostic)
                ├── workload-design.md    # Little's Law, concurrency, pacing
                ├── test-data.md          # Parameterization, CSV, data isolation
                ├── script-generation.md  # Naming conventions, error handling
                ├── correlation.md        # Dynamic values, Regex, JSONPath (Highlighted!)
                ├── test-execution.md     # Distributed load, CI/CD pipelines
                ├── results-analysis.md   # Latency percentiles, bottleneck triage
                ├── observability.md      # Grafana, APM integration, VM profiling
                ├── production-testing.md # Chaos engineering, canary safety controls
                ├── protocol-testing.md   # gRPC, GraphQL, WebSockets, Kafka
                ├── database-testing.md   # JDBC, pools, deadlocks, slow queries
                └── modern-architectures.md # K8s scaling, HPA, Frontend CWV
```

---

## 💡 How the Skill Works

To avoid hitting context window limits, the skill operates on a **selective loading** design:

1. **Routing**: [SKILL.md](skills/perf/SKILL.md) is always read first.
2. **Selective Context**: Based on the query, only the exact tool file (e.g., `k6.md`) and the relevant topic file (e.g., `correlation.md`) are loaded.
3. **Consistency**: Global rules (like realistic think times and assertions) are enforced uniformly across all tool files by anchoring them in [SKILL.md](skills/perf/SKILL.md).

### ❓ Example Queries and Loaded Files

| Query | Files Loaded |
| :--- | :--- |
| *"Help me write a k6 load test for our REST API"* | `k6.md` |
| *"How should I correlate a dynamic JSESSIONID in JMeter?"* | `jmeter.md` + `correlation.md` |
| *"Set up a distributed Locust execution in GitLab CI"* | `locust.md` + `test-execution.md` |
| *"Our p95 latency spikes during database writes, how do we fix?"* | `database-testing.md` + `results-analysis.md` |

---

## 🤝 Contributing

We welcome contributions to expand the knowledge base:
1. **Tool-specific syntax** belongs in `references/tools/`.
2. **Generic concepts** belong in `references/topics/`.
3. **Keep it DRY**: Reference topic files from tool files instead of duplicating.
4. **Update the Index**: Remember to update the Reference Map in [SKILL.md](skills/perf/SKILL.md) when adding files.

## 📄 License

This project is licensed under the MIT License. See [LICENSE.md](LICENSE.md) for details.
