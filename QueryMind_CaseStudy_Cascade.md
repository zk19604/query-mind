# Case Study: Deploying QueryMind at Cascade Systems

*A mock deployment of QueryMind, the no-code local NL2SQL agent framework, inside a mid-size software services company.*

---

## 1. The Company

**Cascade Systems** is a 240-person custom software services house based in Lahore. It builds web and mobile products for international clients on a time-and-materials and fixed-bid basis.

Its client roster is the reason security matters here — and the reason a cloud NL2SQL tool is off the table:

- Two retail **banks** (one local, one in the GCC) with contractual data-residency clauses.
- A US **healthcare SaaS** client whose BAA pushes HIPAA-style handling obligations downstream onto Cascade.
- A **government** digitization project under a confidentiality undertaking.

Every one of these clients' projects is referenced inside Cascade's *own internal systems* — ticket titles, time-log notes, project codenames, billing lines. Piping that data into ChatGPT or a cloud analytics LLM would breach NDAs Cascade has already signed. That is the load-bearing reason QueryMind's **local Ollama inference (nothing leaves the building)** is not decorative here — it is the entry ticket.

### Data systems (all on-premises / local datacenter)
| System | Purpose | Database |
|---|---|---|
| Jira | Engineering / sprint / tickets | PostgreSQL |
| HubSpot (self-hosted mirror) | Sales pipeline | PostgreSQL |
| **Pulse** (in-house ERP) | Timesheets, billing, utilization, project margin | MySQL |
| PeopleHub (HRIS) | Headcount, hiring, attrition, comp | MySQL |

These are **conventional operational databases**, not a cloud warehouse — again, exactly QueryMind's stated sweet spot.

---

## 2. The Bottleneck Before QueryMind

All reporting flows through two people:

- **Fahad** — sole data engineer / BI analyst.
- A junior analyst who helps part-time.

Anyone who needs a number that isn't on a canned dashboard files a request to Fahad. Measured over a representative month:

- **~60 ad-hoc data requests per week.**
- Average **35–45 minutes** per request (clarify → write SQL → format → send).
- That is roughly **38 hours/week — nearly a full-time person — spent answering questions, not building data infrastructure.**
- **Time-to-answer: half a day to two days**, because requests queue behind project work.

The cost isn't just Fahad's hours. It's **decision latency** — the COO waits two days to learn a project is bleeding margin, by which point another week has been billed at a loss.

---

## 3. The Deployment

QueryMind is installed on a single GPU box in Cascade's datacenter, running a local model via Ollama. Rollout, over three weeks:

1. **Read-only connections** to replicas of the four databases (no write access — querying only).
2. **Schema curation** with Fahad: table/column descriptions, business definitions ("billable hour", "active project", "bench"), and joins encoded so the agent links schema correctly. This is the curation burden we flagged earlier — it's real, and it's front-loaded here.
3. **Trust layer configured:** confidence scoring on every generated query, a custom benchmark of ~80 known question/answer pairs Cascade cares about, and an abstain threshold — below it, QueryMind shows the generated SQL and says it is unsure rather than returning a confident wrong number.
4. **Access scoping:** Finance data is visible only to finance + leadership; comp data restricted to HR + COO.

The pitch to Fahad isn't "you're replaced" — it's "you stop being a human query service and own the semantic layer instead."

---

## 4. A Week in the Life — Who Uses It, What They Ask, What It Saves

### Persona 1 — Ayesha, COO *(the champion)*
**Need:** weekly pulse on utilization and project margin.

> *"What was our company-wide billable utilization last month, and which three projects had the lowest margin?"*

QueryMind links across Pulse's timesheet and billing tables, applies the curated definition of "billable", returns the number plus the three projects with margin %, **confidence: high.**

- **Before:** Monday request to Fahad → answer Tuesday afternoon.
- **After:** answer in ~30 seconds, self-serve, during her own review.
- **Saves:** ~1.5 days of decision latency per week; she now catches margin erosion the same day.

### Persona 2 — Sana, Resource Manager / PMO
**Need:** bench and capacity planning.

> *"Which developers will be unallocated after the 30th, and what are their primary skills?"*

Joins Pulse allocations with PeopleHub skill tags.

- **Before:** a recurring Friday ticket, ~45 min of Fahad's time, every week.
- **After:** Sana runs it herself, daily if she wants.
- **Saves:** ~3 hrs/week of Fahad's time on this query alone; lets Sana redeploy bench staff days earlier, directly reducing non-billable cost.

### Persona 3 — Bilal, Delivery Lead
**Need:** project health across the banking client's three workstreams.

> *"Show open critical bugs and average ticket cycle time for the GCC bank projects this sprint."*

Queries Jira's Postgres DB filtered to the client's project keys.

- **Before:** he'd ping Fahad or eyeball Jira manually and undercount.
- **After:** accurate, in seconds, before standup.
- **Saves:** ~20 min/day of manual digging; better staffing calls because the number is real.

### Persona 4 — Omar, Finance Analyst
**Need:** profitability and receivables.

> *"Total unbilled hours per active client over 90 days old, sorted by amount."*

This one comes back **confidence: medium** — the agent flags an ambiguity between "unbilled" and "uninvoiced" and shows the SQL it used.

- **This is the differentiator in action:** instead of a confidently wrong figure, Omar sees the assumption, confirms it with Fahad once, and the definition gets added to the semantic layer so it's correct forever after.
- **Saves:** prevents a wrong number reaching a client conversation — the kind of error that's expensive precisely because no one catches it.

### Persona 5 — Hina, HR Lead
**Need:** attrition and hiring velocity.

> *"What's our 12-month rolling attrition by department, and how many open senior roles are still unfilled after 60 days?"*

Queries PeopleHub; comp columns are access-scoped out of her view automatically.

- **Before:** quarterly, manually, in a spreadsheet.
- **After:** monthly, in seconds.
- **Saves:** turns an annual ritual into a live signal; earlier intervention on attrition.

### Persona 6 — Fahad, Data Engineer *(the one relieved)*
**Need:** to stop being the bottleneck.

- Ad-hoc request load drops from **~60/week to ~18/week** — only the genuinely complex, multi-source, or low-confidence ones now reach him.
- He spends the reclaimed ~25 hrs/week on the semantic layer, data quality, and the integrations he never had time for.
- He becomes QueryMind's **curator**, not its competitor.

---

## 5. The Savings, Summarized

| Dimension | Before | After | Effect |
|---|---|---|---|
| Ad-hoc requests routed to data team | ~60/week | ~18/week | **−70%** |
| Data-engineer time on ad-hoc queries | ~38 hrs/week | ~12 hrs/week | **~26 hrs/week reclaimed (≈0.65 FTE)** |
| Time-to-answer (typical question) | 0.5–2 days | seconds–minutes | decision latency near-eliminated |
| Wrong-number risk | silent | surfaced via confidence + visible SQL | errors caught *before* they cost money |
| Reporting cadence (utilization, attrition) | weekly/quarterly | on-demand | live signals replace stale snapshots |

**Cost framing for the FYP:** the reclaimed ~0.65 FTE of skilled data-engineer time is the easy-to-quantify number, but the larger value is *avoided cost of slow and wrong decisions* — margin caught the same day, bench redeployed earlier, a bad figure stopped before a client sees it.

---

## 6. Why This Maps to QueryMind's Differentiators

1. **Local inference** — the only reason Cascade can do this at all, given client NDAs. Not a nice-to-have; the gate.
2. **Conventional databases** — Jira/Pulse/CRM/HRIS are operational SQL stores, not Snowflake. Tools built for cloud warehouses don't fit; QueryMind does.
3. **Trust instrumentation** — confidence scoring and visible SQL turn the honest weakness (NL2SQL accuracy isn't 100%) into the *selling point*: "honest when uncertain" beats "confidently wrong" for a company making money decisions.

---

## 7. Honest Caveat (worth keeping in the report)

A software house is a **weaker security buyer than a bank** — the NDA/downstream-compliance argument only holds because Cascade serves regulated clients. A pure product company with no sensitive clients would feel this pressure far less, and for them QueryMind would compete on convenience, not necessity. This case study deliberately picks the segment of software houses where the security story is real.
