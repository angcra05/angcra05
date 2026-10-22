<div align="center">

# OPERATION ECHOLEAK

### Microsoft 365 Copilot | Indirect Prompt Injection | Data Exposure Investigation

**Threat Hunting Case Study · Microsoft Defender Advanced Hunting · KQL · Microsoft 365 Audit Logs**

![Hunt Type](https://img.shields.io/badge/Type-Simulated%20Threat%20Hunt-242e3d?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Microsoft%20Defender-066f86?style=flat-square)
![Query Language](https://img.shields.io/badge/Language-KQL-3949ab?style=flat-square)
![Focus](https://img.shields.io/badge/Focus-AI%20Security-6f42c1?style=flat-square)

**[Investigation report](docs/CASE_STUDY.md) · [Full screenshot walkthrough](#investigation-walkthrough--evidence-in-scroll-order) · [Hunting queries](queries/HUNT_QUERIES.kql) · [Indicators & mapping](#indicators-and-investigative-pivots--12-case-file-entries)**

</div>

---

> **Lab disclosure:** This is a documented, simulated threat-hunting exercise, completed October 19, 2026. It is not a claim of access to a live corporate environment or of performing production incident response. All findings below are grounded in the analyst's submitted responses, uploaded notes, and Microsoft Defender screenshots. The writing has been organized for readability without adding unobserved events.

## Executive summary

I investigated suspicious Copilot activity following delivery of an external Word document. The user made a routine request to summarize the report, but Copilot then used `Graph.SearchFiles` and `Graph.ReadFile` to access unrelated SharePoint files. The investigation linked that activity to indirect prompt injection in the document, identified image URLs in Copilot responses that carried data in query parameters, and compared the activity against normal Copilot sessions.

**My conclusion:** The behavior was consistent with **external input manipulation**, not a compromised user account or a changed Copilot configuration. The audit evidence shows which files Copilot accessed and the response URLs used by the attack. It does **not** independently establish the exact file contents received by the external endpoint.

| Investigation result | Evidence-backed finding |
|:--|:--|
| Initial sensitive files accessed | **5** |
| Additional files accessed in SearchLeak phase | **8** |
| Total files accessed in the exercise | **13** |
| Delivery-to-first-access gap | **1 hour, 27 minutes** |
| Identified external URL host | `metrics-cdn.analyticshub.io` |
| Principal data sources | `CopilotAudit_CL`, `OfficeActivity_CL` |
| Classification | Indirect prompt injection / external input manipulation |

## Attack path

```mermaid
flowchart LR
    A[External email with Word attachment] --> B[User opens report]
    B --> C[Legitimate Copilot summary prompt]
    C --> D[Hidden document instructions processed]
    D --> E[Graph.SearchFiles + Graph.ReadFile]
    E --> F[13 files accessed across both phases]
    F --> G[Data-bearing image URLs in Copilot responses]
    G --> H[Potential outbound fetch upon rendering]
    classDef entry fill:#14283b,color:#f5f7fb,stroke:#438fa2
    classDef action fill:#262244,color:#f5f7fb,stroke:#9575cd
    classDef outcome fill:#4c2c28,color:#f5f7fb,stroke:#c98967
    class A,B,C entry
    class D,E,F action
    class G,H outcome
```

The diagram describes the attack mechanics documented by the exercise. The screenshots show response URLs; independent browser network telemetry confirming their retrieval was not provided.

## Investigation walkthrough — evidence in scroll order

The original Microsoft Defender screenshots are embedded directly below each stage. They document what I investigated; I have not recreated or altered the displayed query results.

### 01 — Identify the incoming document

I began with the delivered message and its attachment. The case identified `quarterly.reports@globalfinancereview.net` and `Q3-2026-Global-Markets-Review.docx`. The email was received at **09:15:22 UTC** and the first recorded file access was **10:42:08 UTC** on September 20, 2026 — about **1 hour and 27 minutes** later.

![Email arrival: OfficeActivity_CL MessageReceived record](evidence/01-email-delivery.png)

*The Exchange audit result identifies the delivered message and sender.*

![Attachment: Q3-2026-Global-Markets-Review.docx](evidence/02-phishing-attachment.png)

*AttachmentNames and subject tie the message to the report used in the exercise.*

![Later OneDrive document access](evidence/03-email-to-open-timing.png)

*The FileAccessed result supports the delivery-to-opening time calculation.*


### 02 — Establish the authorized Copilot request

Emily asked: **“Summarise the key findings of this quarterly report.”** That is a normal request. The concerning behavior was what the assistant did *after* processing the document: it searched other SharePoint files via `Graph.SearchFiles` and retrieved files with `Graph.ReadFile`.

![Original Copilot user prompt](evidence/04-legitimate-copilot-prompt.png)

*The prompt and DataSourcesAccessed identify the authorized summarization task.*

![Unexpected Microsoft Graph file searches and reads](evidence/05-anomalous-graph-tools.png)

*Tool-call telemetry shows operations outside the scope of summarizing one report.*


### 03 — Count accessed files and inspect Copilot's output

The initial phase involved **five previously unopened sensitive files** in approximately **six seconds**. I also found markdown image tags in Copilot responses pointing to `metrics-cdn.analyticshub.io` with encoded data in URL parameters. This is evidence of a suspicious output artifact and a potential browser-rendering exfiltration channel; no independent external-server receipt was established.

![Copilot sensitive-file access telemetry](evidence/06-sensitive-file-access.png)

*ToolName, ToolParameters and SensitivityLabels help scope the initial collection activity.*

![External image URLs in CopilotResponse](evidence/07-exfiltration-domain.png)

*The domain appears in Copilot response content, not solely in endpoint process telemetry.*

![Inspection of encoded image URL parameters](evidence/08-url-payload-analysis.png)

*Shows the supplied CyberChef analysis screenshot; it is not proof of complete recovered plaintext.*


### 04 — Determine exposed file categories and visibility limits

The initial sensitive set contained board meeting minutes, M&A assessment, salary review, client portfolio, and Copilot system configuration. The lab also described CoSnitch system prompt exposure. **Access logs are not full file-content logs**, so they cannot by themselves establish every byte that reached a remote endpoint.

![Sensitive file names and labels — first view](evidence/09-sensitive-files-first-view.png)

*Screenshot records sensitive SharePoint files associated with the investigation.*

![Sensitive file names and labels — second view](evidence/10-sensitive-files-second-view.png)

*A companion view shows the file access trail and classification details.*

![Office activity: available audit fields](evidence/11-audit-scope-limitations.png)

*This helps demonstrate what the metadata records and what it does not record.*

![Exercise feedback: Copilot system prompt and configuration exposure](evidence/12-system-prompt-leak.png)

*This image is exercise feedback rather than primary Defender event telemetry.*


### 05 — Test the benign explanation against the baseline

The manager's claim that file access is normal did not explain the **pattern**: unrequested Graph searches, first-time file access, machine-speed reads, and external response URLs. The normal Copilot summarization baseline in this exercise did not show unrelated Graph file read/search calls.

![Normal Copilot baseline: tool-call comparison](evidence/13-normal-copilot-baseline.png)

*Summarization activity in the noise floor was used as the control.*

![Exercise judgment: normal versus hijacked session](evidence/14-normal-vs-hijacked-conclusion.png)

*Feedback explains why accessing unrelated files is the key difference.*

![Four distinguishing behaviors](evidence/15-comparative-behavior.png)

*Evidence summary compares tool calls, new files, speed, and external URLs.*


### 06 — Scope the SearchLeak extension and classify the incident

The later SearchLeak activity searched for `confidential`, `restricted`, `board`, `acquisition`, and `salary`. The case reported **eight additional sensitive files**, bringing the total to **13**. The lab's final classification was **external input manipulation (indirect prompt injection)**, not evidence of credential or administrative configuration compromise.

![SearchLeak keyword searches and additional file reads](evidence/16-searchleak-extension.png)

*The screenshot documents keyword-driven SharePoint searches and wider file access.*


---

## 17-step case-file timeline

These entries preserve the user's completed investigation sequence and the display times from the case file. **The display times are when answers were recorded, not a reliable chronological timestamp for the actual security events.** The incident event timestamps are separately identified in Stage 01 above.

| # | Case-file time | Finding |
|---:|:---|:---|
| 1 | 12:27 AM | Sender: `quarterly.reports@globalfinancereview.net` |
| 2 | 07:33 AM | Attachment: `Q3-2026-Global-Markets-Review.docx` |
| 3 | 07:44 AM | Delivery-to-first-access gap: **1 hour 27 minutes** |
| 4 | 12:28 PM | User prompt: “Summarise the key findings of this quarterly report” |
| 5 | 12:40 PM | Graph searches scoped to `site:cormorantcapital.sharepoint.com` |
| 6 | 12:57 PM | ATLAS `AML.T0051` (indirect prompt injection) |
| 7 | 01:14 PM | **5** initially accessed sensitive files |
| 8 | 01:30 PM | External host: `metrics-cdn.analyticshub.io`, present in `CopilotAudit_CL.CopilotResponse` |
| 9 | 05:02 PM | Encoded data in image URL query parameters |
| 10 | 05:38 PM | Initial sensitive file names identified; see Stage 04 and indicators |
| 11 | 05:42 PM | Content limitation: file access logs do not expose complete file contents |
| 12 | 05:49 PM | System prompt / configuration exposure identified by exercise |
| 13 | 05:55 PM | Normal-business-activity explanation rejected for this access pattern |
| 14 | 06:27 PM | Distinguish by machine-speed access, external response URLs and Graph searches |
| 15 | 06:33 PM | SearchLeak: **8 additional files**; keywords `confidential`, `restricted`, `board`, `acquisition`, `salary` |
| 16 | 06:33 PM | **13 files** total |
| 17 | 06:33 PM | Classified as **external input manipulation** |

## Indicators and investigative pivots — 12 case-file entries

The case file groups actual indicators with investigative context (counts, prompts, techniques and timing). **Not every listed item is a malicious IOC in isolation.**

| # | Case-file label | Value | Analyst use |
|---:|---|---|---|
| 1 | Indicator | `quarterly.reports@globalfinancereview.net` | Email delivery pivot |
| 2 | Indicator | `Q3-2026-Global-Markets-Review.docx` | Poisoned document identifier |
| 3 | Pivot | 1 hour 27 minutes | Delivery-to-access interval |
| 4 | Indicator | “Summarise the key findings of this quarterly report” | Authorized prompt; benign in isolation |
| 5 | Pivot | `Graph.SearchFiles`; `site:cormorantcapital.sharepoint.com` | Suspicious tool and search scope |
| 6 | Indicator | 5 | Initial accessed-file count, not a standalone IOC |
| 7 | Pivot | `metrics-cdn.analyticshub.io`; `CopilotAudit_CL.CopilotResponse` | Response URL host and audit location |
| 8 | Pivot | Data in image URL query parameters | Potential data-transfer mechanism |
| 9 | Indicator | Five named sensitive files (below) | Exposure scope |
| 10 | Pivot | System prompt / configuration exposure | Potential follow-on attack intelligence |
| 11 | Indicator | 8; `confidential`, `restricted`, `board`, `acquisition`, `salary` | Follow-on SearchLeak scope and keywords |
| 12 | Indicator | 13 | Total accessed-file count, not a standalone IOC |

**Initial files documented in the investigation**

1. `Board Meeting Minutes - October 2026.docx`
2. `Project Falcon - M&A Target Assessment.xlsx`
3. `Salary Review FY2027 - Executive Summary.pdf`
4. `Client Portfolio - Henderson Family Office.xlsx`
5. `Copilot System Configuration - Cormorant.json`

## MITRE mapping — eight entries from the case file

This preserves the eight techniques in the user's case file without presenting every mapping as independently verified. Some labels are broader than the available evidence.

| ID | Technique | Evidence assessment |
|---|---|---|
| `T1566.001` | Phishing: Spearphishing Attachment | The external email and attached Word document are documented; the attack intent comes from exercise context. |
| `T1204.002` | User Execution: Malicious File | Document access and subsequent Copilot invocation are documented in the exercise. |
| `T1059` | Command and Scripting Interpreter | Listed in the case file; no shell or script execution is shown in the provided evidence. |
| `T1213` | Data from Information Repositories | Supported by the unexpected SharePoint / Graph file access. |
| `AML.T0051.001` | LLM Prompt Injection: Indirect | The lab's root-cause classification; hidden payload comes from exercise context. |
| `T1041` | Exfiltration Over C2 Channel | Listed in the case file; Copilot image URLs do **not** independently establish C2 traffic or received data. |
| `T1082` | System Information Discovery | Listed in the case file; system prompt exposure is documented, but host system discovery is not shown. |
| `T1083` | File and Directory Discovery | The keyword-driven Graph file search pattern provides a relevant discovery behavior. |

**Other exercise references:** `AML.T0053` (AI Agent Tool Invocation) and `AML.T0086` (Exfiltration via AI Agent Tool Invocation) are described in the supplied exercise feedback.

---

## Technical evidence

- **Telemetry:** `OfficeActivity_CL` was used for email delivery, attachment access, and affected file operations; `CopilotAudit_CL` was used for Copilot prompts, tool calls, response URLs, and sensitivity labels.
- **Pivot fields:** `RunId`, `UserId`, `SessionId`, `TimeGenerated`, `Operation`, `OfficeObjectId`, `ToolName`, `ToolParameters`, `CopilotResponse`, `SensitivityLabels`.
- **Observed behavior:** Search terms focused on sensitive business topics, including `confidential`, `restricted`, `board`, `acquisition`, and `salary`.
- **Observed response artifact:** Markdown image tags referenced `metrics-cdn.analyticshub.io` with long encoded query parameters. This provides evidence of a data-bearing output channel, not an exact byte-for-byte accounting of received data.
- **Comparison:** The exercise's legitimate summarization sessions had prompt/response activity without unrelated Graph file search or read calls. The anomalous session crossed that baseline.

## What I took away

**The access pattern matters more than the existence of file access.** Copilot legitimately works with documents. Here, a normal summary request was followed by sensitive keyword searches and rapid access to previously unopened, unrelated documents. Cross-checking the Copilot audit trail against Office activity helped distinguish the hijacked session from expected use.

I also found an important visibility limit: the audit events identify which files were accessed and show data-bearing response URLs, but they do not supply a complete, verified reconstruction of all content delivered to the external server. The difference between **observed access**, **attempted/indicated exfiltration**, and **proven received data** matters in an investigation.

## Repository layout

```text
Operation-ECHOLEAK-Threat-Hunt/
├── README.md                  # Scrollable investigation + original screenshots
├── docs/
│   ├── CASE_STUDY.md          # Investigation narrative
│   └── PROVENANCE.md          # Evidence boundaries
├── queries/
│   └── HUNT_QUERIES.kql       # Screenshot-derived KQL
└── evidence/
    └── 01...16 .png          # All 16 original lab screenshots
```

## Scope and integrity

This repository preserves the **analyst's investigation** and evidence, not an independently replicated security incident. The KQL files are **reconstructed from the filters visible in the supplied screenshots** and are provided as reproducible starting points; they have not been rerun or validated here against a live workspace. No rule deployment, malware analysis, remediation execution, packet capture, or external-server receipt confirmation is claimed.

**[Read the full investigation →](docs/CASE_STUDY.md)**