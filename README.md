# Use Case: Cross-Platform NSN Exposure Query (ISR/SOF Portfolio)

**Status:** Draft v1 for internal alignment and demo build
**Topology:** Single Istari instance, multiple Systems (SCS out of scope for v1)
**Data:** All demo data is notional. No real systems, aircraft, or part data.

---

## 1. Summary

When a deficiency alert is issued against a part (identified by NSN), a portfolio-level engineer needs to know quickly where else that part is installed across ISR/SOF platforms, and each program office needs to keep control of what it shares.

Today this is answered with phone calls, emails, and PDFs thrown over the wall, and each office's fleet management systems are siloed. This use case shows an engineer asking one question across three offices' as-installed configuration data, with each office deciding what to share and every answer traced to a specific source export.

**The one-sentence pitch:** Nobody prepares a package. Each office exports what its fleet management system already produces, registers it, and decides who can see it. The engineer queries across offices, every answer cites the export it came from, and access ends when the investigation does.

---

## 2. The problem

- A deficiency alert (a PQDR- or GIDEP-style report) names an NSN and often a lot range. Every program office that might have the part installed has to check its own fleet.
- As-installed configuration lives in each office's own maintenance and configuration systems, which are siloed and do not give access to other systems (as per the customer's own request).
- Cross-office answers are assembled manually, so they are slow, hard to audit, and easy to get incomplete.
- Offices are rightly cautious about sharing. Whole-fleet exports can contain readiness, capability, and maintenance data that other offices do not need.

### What the customer asked for (paraphrased)

Aggregating data across systems and platforms could inform parts supportability and commonality between program offices, and give insight into existing engineering solutions on similar designs. Current ETAR and fleet management systems are siloed. Assumption: other offices concur to share data without affecting their ATOs.

---

## 3. Actors

| Actor | Role in the use case |
|---|---|
| **Portfolio engineer** | Works in a portfolio-level ISR/SOF engineering System. Receives the alert, asks the exposure question, produces the summary. Not tied to any single platform. |
| **E-7 office** | Owns E-7 as-installed configuration data. |
| **RC-135 office** | Owns RC-135 as-installed configuration data. |
| **CV-22 office** | Owns CV-22 maintenance and configuration data, including sections it does not want to share. |
| **Istari platform** | Registers files, enforces permissions, extracts tables, runs AI queries with citations, records provenance. |

---

## 4. The data

The data is described as **as-installed configuration exports**: reports produced by each office's own maintenance or configuration system, registered unedited by their owners. A person still runs the export in v1, but nobody authors or curates a package for the request.

Each export carries system-style metadata (system name, report ID, run timestamp, run-by, data-as-of date), which becomes part of the provenance in answers.

| Office | Format | Contents |
|---|---|---|
| E-7 | CSV | As-installed items for 2 tails, with `#` metadata header lines |
| RC-135 | XLSX | As-installed items for 1 tail, plus a `Report Info` sheet |
| CV-22 | PDF (multi-section) | (1) Fleet capability and readiness summary, (2) as-installed configuration for 6 tails, (3) open maintenance actions |

Formats and column names deliberately differ across offices, as real exports from different systems would.

**Mock files:** `E7_asinstalled_cfg_26-0901_26-0902_20261002.csv`, `RC135_asinstalled_cfg_62-9901_20260929.xlsx`, `CV22_maint_config_report_20260930.pdf`

### Planted test NSNs

| NSN | Item | E-7 | RC-135 | CV-22 |
|---|---|---|---|---|
| `5945-01-999-0101` | Relay, electromagnetic | both tails | 62-9901 | 09-9001, 09-9003, 09-9004 |
| `4810-01-999-0202` | Valve, check | both tails | not present | 09-9001, 09-9004 |
| `5945-01-999-0110` | Decoy (near-miss relay) | not present | 62-9901 | not present |
| `6150-01-999-0303` | Negative control | not present | not present | not present |

The relay carries lot numbers, so an alert with a suspect lot range lets the AI qualify its answer by lot.

---

## 5. Flow

### Setup (pre-staged, not shown)
- Four Systems on one instance: **Portfolio**, **E-7**, **RC-135**, **CV-22**, each with its own users and permissions.
- Each office has already registered its own export in its own System.
- Nothing in the three office Systems is visible to the portfolio engineer by default.

### Beat 1: The alert arrives *(Portfolio engineer)*
- Upload the alert document (PQDR/GIDEP-style) into the Portfolio System. It flags the relay and the valve and gives a suspect lot range.
- The engineer reads the NSNs off the document. No extraction is needed for this.
- *Why register it:* so the final exposure summary can cite the alert as its trigger.

### Beat 2: The question and the gap *(Portfolio engineer)*
- Ask the AI: "What is our exposure to these NSNs across E-7, RC-135, and CV-22?"
- The AI answers from what is visible, which is nothing yet, and says so: exposure is unknown, not zero.
- *Point to land:* unknown is a different answer from no.

### Beat 3: Requests go out *(Portfolio engineer)*
- The engineer asks the three offices to share as-installed data for specific tails.
- Show the three office Systems as empty or denied from the engineer's side.
- The request itself may happen outside the platform (email, chat, ticket). Treat it as real-world friction and do not spend time on it. *(Open question: whether the platform has a built-in request or catalog workflow. Not verified.)*

### Beat 4: E-7 and RC-135 share the whole export *(E-7 admin, RC-135 admin)*
- Each office grants the engineer access to its export file.
- The engineer refreshes and sees the new files.
- These offices have nothing in the export they need to withhold, so the whole file is the right unit of sharing.
- *Say:* "Share what was requested, with an audit trail." Do not claim minimization here.

### Beat 5: CV-22 has a problem *(CV-22 admin)*
- Open the CV-22 PDF. It holds capability and readiness data, open maintenance actions, and as-installed configuration for six tails.
- The office is willing to share as-installed configuration for two tails (09-9001 and 09-9002) and nothing else.
- A PDF can only be shared whole.

### Beat 6: CV-22 extracts and shares a projection *(CV-22 admin)*
- The CV-22 office extracts the tables from its PDF into structured data.
- It selects only the as-installed configuration for the two requested tails and grants those artifacts to the engineer.
- *On screen:* the capability tables, maintenance actions, and other four tails stay in the CV-22 System. Linger on what did not cross.
- *This is where extraction earns its place:* it makes selective sharing possible, which a file cannot do.

### Beat 7: Make everything queryable *(Portfolio engineer)*
- Confirm all sources are in machine-readable form. The CSV, XLSX, and CV-22 tables already are, so this may be just a quick normalization or no step at all.
- Be honest about this step. At demo scale the AI can read most of these formats directly. Structured tables matter because they allow an exhaustive, checkable query rather than reading pages.
- Optional beat. Skip it for a shorter demo.

### Beat 8: The exposure question *(Portfolio engineer)*
- Ask: "What is our exposure to these NSNs across E-7, RC-135, and CV-22, and does the suspect lot range apply?"

### Beat 9: The answer, with coverage *(Portfolio engineer)*
- The answer lists each hit with its tail, installed assembly, location, lot or serial, and the source (file and row, or page and line).
- It also states **coverage**: what was searched, the report dates, and what was not shared. Example: "Searched E-7 tails 26-0901 and 26-0902 (CSV, run 2026-10-02), RC-135 tail 62-9901 (XLSX, run 2026-09-29), CV-22 tails 09-9001 and 09-9002 (as-installed artifacts, run 2026-09-30). CV-22 tails 09-9003 through 09-9006 were not shared, so exposure there is unknown."
- Run the negative-control NSN. The AI answers "not present in the shared data as of these dates," not "no exposure."
- Run the decoy. The AI does not confuse the near-miss relay with the flagged one.
- Capture the result as an **exposure summary artifact**, with provenance back to the alert and every source.

### Beat 10: Close the investigation *(Office admins)*
- Each office revokes its grant.
- The engineer re-asks the question. The answer shrinks to what is still visible, and the offices show as no longer visible.
- The exposure summary keeps its recorded provenance.
- If a security person asks: revocation ends platform access. Anything manually exported or pasted elsewhere is outside the platform's control.

---

## 6. What the demo shows

1. **Explicit, permissioned sharing across offices** with revocation, on one instance, without touching any source system.
2. **One question across three offices and three formats**, with every claim traced to a specific export.
3. **Answers that state their coverage**, so absence and unshared data are never silently treated as "safe."
4. **Two sharing patterns side by side:** an office that shares a whole file, and an office that extracts and shares only a projection. This shows when extraction is needed and when it is not.
5. **A work product with provenance** that a safety board could use.

## 7. What the demo does not claim

- **Not data minimization for E-7 and RC-135.** Whole files cross for those offices.
- **Not a live connection to fleet systems.** v1 uses exports registered by their owners. A scheduled or automated export is the mature direction, but whether the platform supports automatic registration of recurring drops is unverified.
- **Not a digital twin.** What is assembled is as-installed configuration data per tail, which is the foundation layer a twin would build on.
- **Not exhaustive at fleet scale on chat alone.** At demo scale the AI finds every hit. At fleet scale the exhaustive part should be backed by a generated and executed query, with chat as the front door.
- **Not physical separation between offices.** Permissions and control tags enforce who sees what on one instance. In production, which instance and under whose ATO is a real question. SCS is the scale-out answer if offices want separate instances.

---

## 8. Risks and open questions for the customer

1. **Will offices share?** This is the largest risk, bigger than any technical piece. Tail-level configuration, especially alongside capability or status data, can be sensitive. Platform-level answers ("installed here, in these assemblies") are an easier ask than tail-level detail, and tail detail can follow on request.
2. **What do these exports really look like?** Real formats, report names, and fields vary by platform and sustainment concept (organic, contractor-supported, hybrid). The mock formats are guesses.
3. **What would each office not want to share?** Likely candidates are serials, locations, mission capable status, and open maintenance actions. Confirm.
4. **Who produces these reports and how are they shared today?** If the answer is "email a PDF," that is the status quo this improves on.
5. **Alternative the customer may prefer:** each office answers the exposure question against its own data and shares only the answer. The data-sharing version is the more ambitious path.
6. **Identity resolution.** Real data has aliases, supersessions, and reassigned NSNs. The demo uses clean NSNs and one decoy. Real deployment would need proper part-identity resolution.
7. **Platform capabilities to verify before promising:** a built-in access-request or catalog workflow, automatic registration of scheduled exports, and the exact current sharing mechanics.

---

## 9. Relationship to the other use cases

This is the sustainment-side companion to the Block 0 SWaP-C use case. It reuses the same machinery (registration, extraction, permissions, provenance, AI queries, revocation) with the join key changed from part ID in an architecture model to NSN across fleets. Per-office as-installed data structured here also becomes each office's node in a broader cross-platform parts index, which is the long-term direction in the customer's request.

# Open Questions From Runthrough:
Overall, works very well. AI performed exceptionally well in finding the compromised NSNs across sources and artifacts, with correct attributions.

1. Generate exposure report w/lineage? 
    - ANSWER: YES

2. Best way to share data? Enveloped in Systems, raw files, etc?
    - ANSWER: Share E-7/RC-135 Resources and CV-22 Artifacts as raw files via right sidebar (Details > Shared with > <add users here>) in the Resource view inside the System.
    - The Portfolio engineer will then add these resources to their System that contains the GIDP for organization and analysis.

3. Finalize LLM prompts

- Perform exposure search: 

    - "Across the RC-135, E-7, and CV-22 platforms, what's our exposure to the NSNs named in the PQDR report in this system? Cite each hit's source and tell me what was and was not searched."

- Build provenance-backed exposure report:

    - "Save that as a Markdown exposure summary report, and draw its lineage back to the source exports it came from. Commit this report to the same system."


*Important Notes*
- As for actual sharing in-demo, Gideon should probably represent the E-7, RC-135, and MV-22 offices and share the data with me.
- The LLM requires XLSX files to be extracted in order to read the data.