# OctoPerf Reference

> Targets: OctoPerf SaaS (current)

OctoPerf is a cloud-based SaaS performance testing platform built on top of JMeter. It provides a professional UI for test design, cloud execution, and centralized result management.

---

## Key Capabilities

| Feature | Description |
|---|---|
| **JMeter import/export** | Import existing `.jmx` files; export OctoPerf designs as JMeter |
| **Visual scenario builder** | GUI for scripting without raw JMeter XML editing |
| **Cloud execution** | On-demand cloud load generators (AWS regions) |
| **Real-time dashboards** | Live metrics during test run |
| **Trend analysis** | Compare runs over time |
| **SLA alerts** | Email/Slack notifications on KPI breach |

---

## Workflow

1. **Import or Record** - upload existing JMX or use OctoPerf's HAR import (record browser -> export HAR -> import).
2. **Parameterize** - use OctoPerf variable extractors (JSON, Regexp, Header) in the GUI.
3. **Configure Load Profile** - ramp-up wizard or custom curve editor.
4. **Select Regions** - pick cloud injection regions (US-East, EU-West, APAC, etc.).
5. **Run and Monitor** - real-time dashboard with RPS, response time percentiles, error rate.
6. **Analyze** - use built-in report builder or export to CSV/JSON.

---

## OctoPerf-Specific Tips

- Use OctoPerf's **HAR importer** to quickly create scripts from browser recordings.
- Leverage **Virtual User Groups** to model different user populations.
- Store test plans in **Workspaces** with version history.
- Use the **REST API** to trigger runs from CI/CD:

```bash
# Trigger run via API
curl -X POST "https://api.octoperf.com/analysis/executions" \
  -H "X-Api-Key: $OCTOPERF_API_KEY" \
  -d '{"scenarioId": "abc123", "name": "CI Build #42"}'
```

- Always set **SLA thresholds** in the test profile so CI gates work reliably.

> For anti-patterns, assertions, think time, and parameterization principles, see **Key Principles** in `SKILL.md`.
> For CI/CD integration details, see `references/topics/test-execution.md`.
