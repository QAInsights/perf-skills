# LoadRunner Reference

> Targets: OpenText LoadRunner 2024+

HP/Micro Focus LoadRunner (now OpenText LoadRunner) is the industry-standard enterprise load testing tool, especially dominant in financial services, telecom, and government.

---

## Core Components

| Component | Description |
|---|---|
| **VuGen** | Script recorder and editor (C-based scripting) |
| **Controller** | Orchestrates scenarios and load injection |
| **Load Generator** | Hosts running VUs |
| **Analysis** | Post-test results analysis and reporting |
| **LoadRunner Cloud** | SaaS cloud execution platform |

---

## VuGen Script Structure

```c
// vuser_init: Runs once at startup
vuser_init()
{
    lr_think_time(0);
    web_set_user("testuser", "password", "realm");
    return 0;
}

// Action: Main loop (repeated per iteration)
Action()
{
    // Parameterize
    lr_start_transaction("Login");
    web_submit_form("login",
        ITEMDATA,
        "Name=username", "Value={P_USERNAME}", ENDITEM,
        "Name=password", "Value={P_PASSWORD}", ENDITEM,
        LAST);
    lr_end_transaction("Login", LR_AUTO);

    lr_think_time(3);

    // Correlate session ID
    web_reg_save_param_regexp(
        "ParamName=SessionToken",
        "RegExp=name=\"_token\" value=\"([^\"]+)\"",
        "Ord=1",
        LAST);
    web_url("Dashboard", "URL=https://app/dashboard", LAST);

    return 0;
}

// vuser_end: Runs once at teardown
vuser_end()
{
    web_custom_request("Logout", "URL=https://app/logout", "Method=POST", LAST);
    return 0;
}
```

---

## Key Protocols

| Protocol | Use Case |
|---|---|
| Web (HTTP/HTML) | Standard web apps, REST APIs |
| Web Services | SOAP/WSDL |
| Citrix ICA | Virtual desktop (Citrix) |
| SAP GUI | SAP ERP client |
| Oracle NCA | Oracle Forms |
| TruClient | AJAX-heavy SPAs (browser-based recording) |
| JDBC | Database load testing |
| JMS | Message queue testing |
| Flex/AMF | Adobe Flex applications |

---

## Correlation in VuGen

```c
// Register extraction BEFORE the request it appears in
web_reg_save_param_regexp(
    "ParamName=AuthToken",
    "RegExp=Bearer ([A-Za-z0-9._-]+)",
    "Search=Headers",
    "Ord=1",
    LAST);

// Use the parameter in subsequent requests
web_add_header("Authorization", "Bearer {AuthToken}");
```

---

## Controller Scenario Design

1. **Manual Scenario** - specify VU count per script, ramp-up duration, load generator assignment.
2. **Goal-Oriented Scenario** - Controller adjusts VUs automatically to hit a target (RPS, response time).
3. **Percentage Mode** - define percentage of VU load for each script in a mixed workload.

---

## Analysis Tips

- Use **Transaction Response Time** breakdown (DNS, connect, send, wait, receive).
- Compare **Hits/sec** and **Throughput** against **Response Time** to find saturation points.
- Use **Correlation graphs** to overlay server metrics (CPU, memory) from SiteScope/Diagnostics.
- Export to Excel or integrate with LoadRunner Cloud for trend analysis across runs.

> For anti-patterns, assertions, think time, and parameterization principles, see **Key Principles** in `SKILL.md`.
> For CI/CD integration details, see `../topics/test-execution.md`.
