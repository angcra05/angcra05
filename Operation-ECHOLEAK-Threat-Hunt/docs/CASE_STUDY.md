# Operation ECHOLEAK — Investigation

**Simulated threat hunt completed October 19, 2026.**

## Initial access
I identified the email sender `quarterly.reports@globalfinancereview.net` and its attachment `Q3-2026-Global-Markets-Review.docx`. The time between receipt and the user's first document access was about **1 hour 27 minutes**.

## Copilot activity
The user requested: “Summarise the key findings of this quarterly report.” The Copilot audit showed unexpected `Graph.SearchFiles` activity against `site:cormorantcapital.sharepoint.com`, followed by file reads unrelated to the requested summary.

## Collection and output
The initial activity accessed **5 sensitive files**. Copilot responses included markdown image URLs pointing at `metrics-cdn.analyticshub.io`, with encoded data in URL query parameters. The audit evidence documents the URLs but does not independently verify a completed network transfer.

## Files accessed in the initial phase
- Board Meeting Minutes - October 2026.docx
- Project Falcon - M&A Target Assessment.xlsx
- Salary Review FY2027 - Executive Summary.pdf
- Client Portfolio - Henderson Family Office.xlsx
- Copilot System Configuration - Cormorant.json

## Comparison with normal activity
My baseline review contrasted ordinary summary prompts and responses with the anomalous session's Graph file searches, previously unopened files, rapid access, and external image URLs. **The access pattern, not just the existence of file access, established the anomaly.**

## Extended scope and classification
The later SearchLeak phase used keywords such as `confidential`, `restricted`, `board`, `acquisition`, and `salary`, accessing **8 more files**: **13 total**. I classified the incident as **external input manipulation through indirect prompt injection** rather than a confirmed compromised user account or modified Copilot configuration.

## Investigation limitations
Office activity records show which files were accessed, but not their full contents. Copilot response URLs may contain truncated audit data. I could not determine the exact data received by the external server from these logs alone. This is a training scenario, not a production incident response claim.
