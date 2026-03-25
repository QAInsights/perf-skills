# NeoLoad Reference

> Targets: NeoLoad 2024.x+

NeoLoad is a commercial enterprise performance testing tool from Tricentis, designed for large-scale, protocol-rich testing including HTTP, SAP, Citrix, and Flex.

---

## Core Concepts

| Concept | Description |
|---|---|
| **Virtual User Profile** | Defines a user type and its behavior |
| **Population** | Group of VU profiles with percentage mix |
| **Scenario** | Load injection profile (ramp, constant, peak) |
| **Container** | Transaction group (like JMeter Transaction Controller) |
| **Variable Extractor** | Correlation mechanism |
| **SLA Profile** | KPI thresholds; alerts and pass/fail criteria |
| **Controller** | Orchestrates test execution |
| **Load Generator** | Injects simulated users |

---

## Workload Design in NeoLoad

1. **Record** via NeoLoad browser proxy or VU recorder
2. **Parameterize** using Variable Extractors (Regexp, XPath, JSONPath) and File Variables
3. **Apply SLA Profile** with thresholds for response time, error rate, throughput
4. **Design Population** mixing multiple VU profiles
5. **Configure Scenario** with ramp-up policies and constant phase

---

## CLI / API Execution

```bash
# Run from command line
NeoLoadCmd -project mytest.nlp \
  -scenario "Regression_Load" \
  -testResultName "CI_Run_${BUILD_NUMBER}" \
  -leaseServer neoload-controller:7400

# REST API trigger (for CI/CD)
curl -X POST "https://neoload-api.tricentis.com/v3/tests/{testId}/start" \
  -H "accountToken: $NEOLOAD_TOKEN" \
  -d '{"scenario": "Load_Scenario"}'
```

---

## NeoLoad-Specific Tips

- Always set **Network Emulation** to match target environment (LAN, WAN, mobile).
- Use **Shared Containers** for reusable sequences (e.g., authentication flow).
- Configure **Cache and Cookie Policies** per application behavior.
- Use **NeoLoad Web** for centralized result storage, trend analysis, and team collaboration.
- Integrate with **Dynatrace/AppDynamics** via built-in APM connectors.
- Use **as-code YAML definitions** (NeoLoad as Code) for version-controlled test configs.

> For anti-patterns, assertions, think time, and parameterization principles, see **Key Principles** in `SKILL.md`.
> For CI/CD integration details, see `references/topics/test-execution.md`.
