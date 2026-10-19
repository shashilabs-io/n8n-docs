# 🔍 Debugging and Error Handling in n8n
## Master Automation Workflows with Execution Logs & Incident Triage

<p align="center">
  <a href="../3_deep%20dive%20concepts/20_Automate_File_Management_with_n8n.md">⬅️ Prev: Automate File Management</a> •
  <a href="../README.md">Main Hub</a> •
  <a href="2_Fixing_AI_Automation_Workflows_Fast_in_n8n.md">Next: Fixing AI Workflows Fast ➡️</a> •
  <a href="https://github.com/shashilabs-io/n8n-docs">GitHub Repository</a>
</p>

---

## 🧭 How to Use This Chapter

> [!TIP]
> **Production Reliability:** Automations succeed in tests and fail in production because real users supply empty names, expired tokens, and malformed files. Mastering n8n's **Execution Logs**, stack trace inspection, data-saving retention policies, and the difference between "Workflow Completed" vs "Business Success" turns you into a fearless production engineer.

### 🎯 Key Outcomes of this Chapter:
- Differentiate between **Manual/Test Executions** vs automated **Production Executions**.
- Navigate and filter the **Executions Hub** by workflow, status, and duration.
- Inspect incoming payloads, outgoing schemas, and remote API error stack traces.
- Diagnose the 6 most frequent production workflow failure archetypes.
- Know when **Retry On Fail** helps vs when it wastes API rate limits.
- Implement explicit **Error Output Routing** to prevent silent data loss.

---

# 1. Understanding Workflow Executions

A **workflow execution** represents a single discrete run of an automated pipeline from its trigger through all terminal action nodes.

```mermaid
flowchart TD
    subgraph MANUAL["🧪 Manual Execution (Editor Mode)"]
        M1["Clicked 'Test step' or 'Execute workflow'"] --> M2["Runs in browser session"]
        M2 --> M3["Used for live schema debugging"]
    end

    subgraph PROD["🚀 Production Execution (Live Mode)"]
        P1["Webhook Push, Cron Schedule, or App Event"] --> P2["Runs autonomously in background"]
        P2 --> P3["Logged to Execution History"]
    end

    style MANUAL fill:#f9f9f9,stroke:#4A90E2,stroke-width:2px
    style PROD fill:#f9f9f9,stroke:#2ECC71,stroke-width:2px
    style M1 fill:#4A90E2,stroke:#333,stroke-width:2px,color:#fff
    style P1 fill:#2ECC71,stroke:#333,stroke-width:2px,color:#fff
```

> [!NOTE]
> A workflow must be toggled **Active** and possess a valid trigger node (Cron, Webhook, App Trigger) to execute automatically in production.

---

# 2. Navigating the Executions Hub

The **Executions Hub** (accessible via the left sidebar) is your flight data recorder:

| Column | Information Captured | Triage Value |
|---|---|---|
| **Status** | `Success` (Green), `Error` (Red), `Waiting` (Blue). | Instant health overview. |
| **Start Time** | Exact timestamp of trigger activation. | Correlating with customer incident reports. |
| **Duration** | Total milliseconds/seconds to complete. | Detecting sudden API latency or rate throttling. |
| **Type** | Manual vs Production. | Discarding developer test runs from customer runs. |
| **Execution ID** | Immutable unique identifier. | Programmatic retrieval via n8n REST API. |

```mermaid
flowchart LR
    A["Executions List"] --> B["Filter by: Status = Error"]
    B --> C["Open Failed Execution ID"]
    C --> D["Locate Red Node"]
    D --> E["Inspect: Input vs Error Message"]

    style A fill:#34495E,stroke:#333,stroke-width:2px,color:#fff
    style B fill:#F39C12,stroke:#333,stroke-width:2px,color:#fff
    style C fill:#4A90E2,stroke:#333,stroke-width:2px,color:#fff
    style D fill:#E74C3C,stroke:#333,stroke-width:2px,color:#fff
    style E fill:#2ECC71,stroke:#333,stroke-width:2px,color:#fff
```

> [!CAUTION]
> **Data Privacy in Execution Logs:** Saved inputs and outputs contain real customer records, email addresses, and occasionally sensitive tokens. Review your **Execution Data Retention** settings (`Save Failed Executions`, `Save Successful Executions`) to ensure compliance with GDPR and company privacy policies.

---

# 3. Systematic 9-Step Debugging Checklist

When a production workflow crashes, follow this disciplined diagnostic sequence:

```mermaid
flowchart TD
    S1["1. Open Executions View"] --> S2["2. Filter by Workflow & Failed Status"]
    S2 --> S3["3. Inspect Start Time & Execution Type"]
    S3 --> S4["4. Open Execution & Pinpoint Failed Node (Red)"]
    S4 --> S5["5. Read Raw Error Stack Trace"]
    S5 --> S6["6. Inspect Exact Input JSON Received by Node"]
    S6 --> S7["7. Check Upstream Node Transforms for Typos"]
    S7 --> S8["8. Replicate Failure in Editor via Edit Output"]
    S8 --> S9["9. Apply Targeted Fix & Verify Controlled Test"]

    style S1 fill:#34495E,stroke:#333,stroke-width:2px,color:#fff
    style S4 fill:#E74C3C,stroke:#333,stroke-width:2px,color:#fff
    style S5 fill:#E67E22,stroke:#333,stroke-width:2px,color:#fff
    style S6 fill:#F39C12,stroke:#333,stroke-width:2px,color:#fff
    style S9 fill:#2ECC71,stroke:#333,stroke-width:2px,color:#fff
```

> [!IMPORTANT]
> **Core Debugging Maxim:** Never stop at *"the node failed."* Determine:
> 1. Which node failed?
> 2. What exact input did it receive?
> 3. What exact error code/message was emitted?

---

# 4. The 6 Most Frequent Production Failure Archetypes

| Archetype | Common Symptom | Diagnostic Investigation |
|---|---|---|
| **Missing / Null Field** | `Cannot read property of undefined` | Upstream form omitted an optional field (e.g. `companyName: null`). |
| **Malformed Data Format** | `Invalid email` or `Date out of bounds` | User entered `"none"` in an email field or submitted European date formats. |
| **API Auth Expiration** | `401 Unauthorized` or `Invalid Token` | OAuth2 refresh token revoked or API key expired. |
| **Rate Limit Throttling** | `429 Too Many Requests` | Burst of items overwhelmed external API; needs a **Wait** node or batching. |
| **Schema / File Type Mismatch** | `Expected image/*, received application/pdf` | User uploaded an unsupported attachment format. |
| **Incorrect Logic Inversion** | Items take wrong branch on edge case | Evaluated `AND` instead of `OR`, or neglected `0` vs `null`. |

---

# 5. When to Use "Retry On Fail" vs When It Hurts

Under node **Settings &rarr; Retry On Fail**, you can specify automatic retries with backoff intervals:

```mermaid
flowchart TD
    ERR["Node Throws Error"] --> EVAL{"Is failure transient or deterministic?"}
    
    EVAL -->|"Transient (Network drop, 502, 429)"| ROK["✅ Enable Retry On Fail<br/>• Max 3 Tries<br/>• 2000ms Interval"]
    EVAL -->|"Deterministic (Missing field, 400 Bad Request, PDF rejected)"| RNO["❌ DO NOT RETRY!<br/>• Wastes execution quota<br/>• Fix schema upstream"]

    style ERR fill:#E74C3C,stroke:#333,stroke-width:2px,color:#fff
    style EVAL fill:#F39C12,stroke:#333,stroke-width:2px,color:#fff
    style ROK fill:#2ECC71,stroke:#333,stroke-width:2px,color:#fff
    style RNO fill:#C0392B,stroke:#333,stroke-width:2px,color:#fff
```

---

# 6. Node Error Policies: Stop vs Continue vs Route

```mermaid
flowchart TD
    ACT["Action Node Encounters Error"]
    ACT --> O1["Option A: Stop Workflow (Default)<br/>Halts run immediately; protects downstream data"]
    ACT --> O2["Option B: Continue (Regular Output)<br/>Passes error object in item array; risks silent data loss"]
    ACT --> O3["Option C: Continue (Using Error Output)<br/>Bifurcates success vs error paths cleanly"]

    style ACT fill:#34495E,stroke:#333,stroke-width:2px,color:#fff
    style O1 fill:#E74C3C,stroke:#333,stroke-width:2px,color:#fff
    style O2 fill:#F39C12,stroke:#333,stroke-width:2px,color:#fff
    style O3 fill:#2ECC71,stroke:#333,stroke-width:2px,color:#fff
```

### The Gold Standard: Error Output Routing (Option C)
Routing failed items through a dedicated error terminal enables fault-tolerant processing:

```mermaid
flowchart LR
    IN["Batch of 50 Leads"] --> CRM["🗄️ Salesforce Create"]
    CRM -->|"✅ 45 Valid Leads"| EM["📧 Send Welcome Email"]
    CRM -->|"❌ 5 Failed Leads"| DLQ["📝 Dead-Letter Sheet + 🚨 Slack Alert"]

    style IN fill:#4A90E2,stroke:#333,stroke-width:2px,color:#fff
    style CRM fill:#34495E,stroke:#333,stroke-width:2px,color:#fff
    style EM fill:#2ECC71,stroke:#333,stroke-width:2px,color:#fff
    style DLQ fill:#E74C3C,stroke:#333,stroke-width:2px,color:#fff
```

> [!WARNING]
> **Workflow Completion $\neq$ Business Success:** A workflow can show a green checkmark if configured with "Continue on Fail", even while 90% of your records failed to send! Always verify business-level item metrics, not just canvas exit status.

---

# 7. Production Debugging Checklist

Before concluding an incident investigation:
- [ ] Confirmed whether the failed run was manual or triggered by production.
- [ ] Located the red failure node and copied the raw JSON input that triggered the bug.
- [ ] Verified whether third-party API credentials, scopes, or rate limits caused the crash.
- [ ] Replicated the error locally inside the canvas editor using **Edit Output**.
- [ ] Implemented upstream validation or dead-letter error routing.
- [ ] Confirmed that retries are restricted strictly to transient network glitches.

---

# 8. Practice Exercises & Active Recall

<details>
<summary><strong>1. What is the fundamental operational difference between a Manual and Production execution?</strong></summary>

A manual execution is triggered by a human developer in the editor during testing; a production execution runs autonomously in the background via live webhooks, cron schedules, or app event triggers.
</details>

<details>
<summary><strong>2. Why is inspecting node input data the most critical step in debugging?</strong></summary>

Because 90% of workflow errors stem from unexpected data shapes (missing properties, null values, or bad types) passed in by upstream nodes.
</details>

<details>
<summary><strong>3. Give two examples of errors where Retry On Fail is appropriate, and two where it is useless.</strong></summary>

- **Appropriate:** HTTP 502 Bad Gateway (temporary server blip) and HTTP 429 Rate Limit (service recovers after delay).
- **Useless:** HTTP 400 Bad Request (missing required JSON key) and passing a PDF to an image-only node.
</details>

<details>
<summary><strong>4. What is the architectural danger of setting On Error to "Continue (Regular Output)" without an audit step?</strong></summary>

It causes silent data loss. The workflow completes with a green success status while failed records vanish without alerting the engineering team.
</details>

<details>
<summary><strong>5. How does a dedicated Error Output terminal resolve silent failures?</strong></summary>

It separates successful records from failing records, allowing valid items to proceed while routing failed items to logging sheets and Slack alerts.
</details>

---

# 9. Key Takeaways

- Use the **Executions Hub** to inspect historical run parameters, durations, and error messages.
- Always check the exact input payload that reached a failed node before attempting a fix.
- Apply **Retry On Fail** only to transient network and server outages.
- Distinguish between **Workflow Completion** (green run) and **Business Success** (all records processed).
- Route failed items to a dead-letter queue or incident alert rather than silently swallowing errors.

---

<p align="center">
  <b>Next Step:</b> Troubleshoot and stabilize AI LLM pipelines in <a href="2_Fixing_AI_Automation_Workflows_Fast_in_n8n.md">Chapter 2: Fixing AI Automation Workflows Fast</a> or return to the <a href="../README.md">Main Hub</a>.
</p>
