# Databricks Genie Agents — Best Practices

**Version 2 — verified 19 September 2026.** Every statement below is taken from the sources in the table. Nothing has been added from outside them.

| Ref | Source | Page last updated | URL |
|-----|--------|-------------------|-----|
| **[OFF-1]** | Curate an effective Genie Agent | 2026-09-11 | https://learn.microsoft.com/en-us/azure/databricks/genie-agents/best-practices |
| **[OFF-2]** | Tune Genie Agent quality | 2026-09-11 | https://learn.microsoft.com/en-us/azure/databricks/genie-agents/tune-quality |
| **[OFF-3]** | Create and manage a Genie Agent | 2026-08-20 | https://learn.microsoft.com/en-us/azure/databricks/genie-agents/set-up |
| **[OFF-4]** | Test and monitor a Genie Agent | 2026-09-17 | https://docs.databricks.com/aws/en/genie-agents/monitor |
| **[OFF-5]** | Troubleshoot Genie Agents | 2026-09-11 | https://docs.databricks.com/aws/en/genie-agents/troubleshooting |
| **[OFF-6]** | Use the Genie Agents API | 2026-08-27 | https://learn.microsoft.com/en-us/azure/databricks/genie-agents/conversation-api |
| **[GH-1]** | `create-genie-agent.md`, databricks/databricks-agent-skills | fetched 2026-09-19 | https://github.com/databricks/databricks-agent-skills/blob/main/skills/databricks-genie-agents/references/create-genie-agent.md |

> **Naming note [OFF-1 – OFF-6]:** Genie Agents were formerly known as Genie Spaces. Older material, API paths (`/api/2.0/genie/spaces`) and CLI commands (`create-space`) still use the previous name.

### What changed in v2

- Added **[OFF-3]** through **[OFF-6]**: hard limits, benchmark mechanics and rating rules, the full troubleshooting playbook, token-limit management, and `serialized_space` validation rules.
- **Table-limit discrepancy re-checked and partly resolved.** The official *technical requirements and limits* page **[OFF-3]** states **30 tables or views**, which matches the GitHub skill **[GH-1]**. The best-practices page **[OFF-1]** states **50**. Both official pages are current. See §4.
- Added the Genie Code assist paths that now appear throughout the official docs (agent creation, response debugging, usage analysis, benchmark-run analysis).

---

## 1. Core mental model

**[OFF-1]** Think of Genie as a new data analyst joining your company. Like any new team member, Genie needs clear context to be effective. It relies on quality table and column descriptions to understand what the data represents, example SQL queries to learn how to solve common problems, SQL expressions to define business terminology, and text instructions only when other methods don't apply. The more structured context you provide, the more accurately Genie can interpret questions and generate correct results.

**[OFF-1]** The curator's job is to bridge the gap between Genie's general world knowledge and the specialized language used in a specific domain or company.

**[OFF-4]** A Genie Agent can be thought of as a long-term collaboration tool between data teams and business users. It accumulates knowledge over time rather than serving as a one-time deployment.

**[OFF-4]** Like other large language models, Genie can exhibit non-deterministic behaviour — you might occasionally receive different outputs for the same prompt. Providing example SQL queries helps make Genie more consistent.

---

## 2. The four guiding principles

**[OFF-1]**

1. **Provide concise, well-documented datasets.** Quality table and column descriptions in Unity Catalog are critical for Genie accuracy. Resolve column ambiguities and pre-join or de-normalize tables using views or metric views.
2. **Prioritize SQL expressions and example SQL over text instructions.** Structured definitions through SQL are more reliable and maintainable than plain text guidance.
3. **Write clear, specific text instructions.** Instead of "Ask clarification questions when asked about sales", write "When users ask about sales metrics without specifying product name or sales channel, ask: To proceed with sales analysis, specify your product name and sales channel."
4. **Avoid conflicting instructions.** If text instructions specify rounding decimals to two digits, example SQL queries must also round to two digits.

**[OFF-2]** Because Genie operates in a nondeterministic manner, guidance must be free from conflicting or ambiguous information to minimize the risk of undesirable responses. Reviewing and resolving inconsistencies is a key setup task.

**[OFF-2]** Defining business logic as SQL expressions produces more consistent results than text instructions, because Genie applies the logic exactly as written rather than interpreting it from natural language.

---

## 3. Before you build

### 3.1 Purpose, audience, owner

**[OFF-1]** An agent should answer questions for a particular topic and audience, not general questions across various domains. Simplify datasets by pre-joining tables and removing unnecessary columns before adding data, and hide any columns that might be confusing or unimportant.

**[OFF-1]** Have a domain expert define the agent — an effective creator needs to understand the data and the insights that can be gleaned from it. Data analysts proficient in SQL typically have the knowledge and skills to curate it.

### 3.2 Requirements to gather first (gate)

**[GH-1]** Ask for all of the following before doing anything else — do not profile data, write JSON, or run any CLI command until you have items 1–4 at minimum:

1. **Target audience** — who will use this Agent and what is their domain fluency?
2. **Agent purpose** — what business area or data domain does it cover?
3. **3–5 real business questions** — concrete questions users will ask (not generic examples).
4. **Known data sources** — catalog/schema/table names or keywords to search for.
5. **Terminology and KPI definitions** — business terms, fiscal conventions, default filters.
6. **Benchmark intent** — should this Agent be evaluated? Chat mode, Agent mode, or none?

If the user provides partial information, ask for what is missing rather than assuming.

### 3.3 Non-negotiable rules during design **[GH-1]**

- Use only bounded read-only SQL: `SELECT`, `WITH`, `SHOW`, `DESCRIBE`, `EXPLAIN`, `information_schema`.
- Never mutate Unity Catalog objects or data (`CREATE`, `ALTER`, `DROP`, `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `COPY INTO`).
- Do not create or alter Metric Views here — use the `databricks-metric-views` skill.
- Do not create or update a live Genie Agent without explicit user approval.
- Do not invent business definitions, joins, fiscal calendars, or metric formulas — ask the user.
- Do not add benchmark SQL without checking it with read-only execution or `EXPLAIN` first.

### 3.4 Prerequisites and permissions **[OFF-3]**

- **Unity Catalog:** the data must be registered to Unity Catalog.
- **Compute:** a pro or serverless SQL warehouse, with at least CAN USE on it. For optimal performance Databricks recommends serverless.
- **To create or edit:** Databricks SQL entitlement, CAN USE on a warehouse, `SELECT` on the data, and at least CAN EDIT on the agent (creators get CAN MANAGE).
- **To consume:** consumer access or Databricks SQL entitlement, `SELECT` on all data objects used, and at least CAN VIEW / CAN RUN on the agent. End users do not need warehouse permissions.
- **Account setup:** Genie uses partner-powered AI features, which must be enabled at account and workspace level.

**Two credentials [OFF-3]:** compute access to the warehouse uses the embedded credentials of the author who configured it; **data access is always evaluated as the end user's own Unity Catalog identity**. Any question about data a user cannot access generates an empty response. To restrict what each user sees within one agent, apply row filters and column masks in Unity Catalog — they are enforced per user automatically.

---

## 4. Sizing and hard limits

### 4.1 Consolidated limits

| Limit | Value | Source |
|---|---|---|
| Tables / views per agent | **30** (stated on the technical requirements and limits page) | **[OFF-3]** |
| Tables / views / metric views per agent | **50** (stated on the best-practices page) | **[OFF-1]** |
| Tables / views / Metric Views per agent | **30** (hard limit) | **[GH-1]** |
| Recommended starting size | **5 or fewer** | **[OFF-1] [GH-1]** |
| Instructions per agent (each example SQL query, each SQL function, and the whole General instructions block count as one) | **100** | **[OFF-2]** |
| Knowledge store snippets per agent (table descriptions + join relationships + SQL expressions) | **200** | **[OFF-2]** |
| Text instruction objects in `serialized_space` | **at most 1 per agent** | **[OFF-6]** |
| Entity-matching columns | **120** | **[OFF-2]** |
| Distinct values per entity-matching column | **1,024**, each up to **127 characters** | **[OFF-2]** |
| Benchmark questions per agent | **500** | **[OFF-4]** |
| Conversations per agent | **10,000** | **[OFF-3]** |
| Messages per conversation | **10,000** | **[OFF-3]** |
| Chat-mode benchmark result-set comparison | up to **5,000 rows** | **[OFF-4]** |
| CSV download from a response | up to approximately **1 GB** | **[OFF-4]** |
| String element in `serialized_space` | **25,000 characters**; repeated fields **10,000 items** | **[OFF-6]** |
| UI throughput | (see Genie troubleshooting for per-workspace question rate limits) | **[OFF-5]** |

> ⚠️ **Unresolved discrepancy on the table limit.** **[OFF-3]**, the page that carries the "Technical requirements and limits" heading, says *you can add up to 30 tables or views to a Genie Agent*, and its page description repeats *up to 30 Unity Catalog tables and 10,000 conversations*. **[OFF-1]** says *Genie Agents support up to 50 tables, views, or metric views*. **[GH-1]** says 30. Two of the three sources, including the dedicated limits page, say 30 — **design to 30**, and confirm in your own workspace before planning anything above it.

### 4.2 Start small **[OFF-1]**

- **Stay focused.** Include only the tables necessary to answer the questions the agent should handle. Aim for five or fewer, and limit the number of columns in those tables.
- **Pre-join when you would exceed the limit.** Pre-join related tables into views or metric views. Metric views are particularly effective because they pre-define metrics, dimensions and aggregations — this keeps you within the limit, simplifies the data model, and can improve response accuracy.
- **Plan to iterate.** Start minimal; add detailed guidance and examples as you refine, rather than aiming for perfection initially.
- **Build on well-annotated tables.** Column descriptions should offer precise contextual information and avoid ambiguous or unnecessary details. Inspect AI-generated descriptions for accuracy and use them only if they match what you would have written.

### 4.3 Organize by domain **[GH-1]**

- Keep Agents focused — the tighter the domain, the better Genie performs.
- Organize by **business domain/subdomain, not by report**. A domain (e.g. "Marketing") maps to an Agent; split broad domains by subdomain (e.g. "Online Marketing").
- If a domain approaches the object limit, split it into multiple Agents by subdomain.
- Assign domain/subdomain tags to both Agents and their underlying tables/Metric Views for discoverability and observability.
- Optional: mirror the hierarchy in Unity Catalog — one schema per domain or subdomain.

### 4.4 Scope caveat worth knowing **[OFF-3]**

Genie can query tables beyond those explicitly added to an agent. Access is controlled by Unity Catalog permissions, not by the agent itself. Genie uses the attached tables and views by default, but users can query others by prompting for joins or editing SQL directly, and if instructions or metadata reference tables outside the agent, Genie can include them in generated queries.

---

## 5. Design workflow **[GH-1]**

Follow top to bottom; the embedded gates tell you when to pause and ask the user.

| Step | Action |
|------|--------|
| **1. Gather requirements** ⛔ *Gate* | Collect items 1–6 in §3.2 before anything else. |
| **2. Discover or confirm data** | Use provided identifiers when available; otherwise search using requirement terms, synonyms, abbreviations, and likely fact/dimension naming patterns. Recommend a focused source set and explain how each source maps to the business questions. |
| **3. Metric View check** ⛔ *Gate* | If the source set is raw tables/views and the questions centre on reusable KPIs (revenue, margin, conversion rate), **stop and recommend building a governed Metric View first**, state the benefit, and **pause until the user answers**. Proceed with raw tables only if they decline or the questions need raw detail. |
| **4. Inspect and profile in phases** | Unity Catalog metadata first, then bounded SQL — see §7. |
| **5. Inspect Metric View semantics** | Inspect measures, dimensions, filters, joins, time dimensions, comments, display names, synonyms and formatting before adding extra Genie context. |
| **6. Assess readiness** | Score each business question High/Medium/Low — see §11. Do not proceed with Low-confidence questions unresolved. |
| **7. Design Agent surfaces** | Structured context first — see §6. |
| **8. Review draft** | Check against §16 health checks and §15 anti-patterns. |
| **9. Present and get approval** ⛔ *Gate* | Present the full proposed configuration. Do not run `create-space` or `update-space` until the user explicitly approves. |

### Genie Code as an assist path **[OFF-3] [OFF-4] [OFF-5]**

- **On creation [OFF-3]:** when you create an agent, Genie Code launches automatically. It reads your data and suggests context — table descriptions, example queries — for you to review and accept. You can also ask it to build an agent when you don't yet know which tables to use: describe the domain, the questions and the data, and it identifies relevant data, confirms the sources, and builds the agent.
- **On a bad response [OFF-4]:** open Genie Code from the response, describe the problem and desired behaviour, and review the context changes it proposes.
- **On usage [OFF-4]:** **Analyze Agent Usage** reviews user messages, feedback and issues from the last six months and reports common topics, recurring issues and suggested context improvements, with citations back to conversations.
- **On benchmarks [OFF-4]:** after a run, Genie Code can review the whole run — expected results, generated results, and current context — and suggest improvements.
- **[OFF-5]** For many common problems, the fastest path to a fix is Genie Code.

---

## 6. Choosing the right instruction type

### 6.1 Official decision rule **[OFF-1]**

- **SQL expressions for common business terms** — frequently used metrics, filters or dimensions representing standard business concepts: `revenue`, `active_customers`, `gross_margin`, `recent_sales`.
- **Example SQL queries for complex questions** — hard-to-interpret, multi-part or complex questions needing intricate query patterns and multi-step logic, e.g. "breakdown my team's performance" or "For customers who've only joined recently, what products are doing the best?".
- **Text instructions only as a last resort** — guidance requiring natural-language explanation, such as asking users to clarify a time range, or rounding percentages in summaries. Avoid using text instructions to define metrics, filters or query patterns expressible through SQL.

**[OFF-1]** Genie Agents perform best with a limited, focused set of instructions. Too many instructions can reduce effectiveness, especially in longer conversations, because Genie might struggle to prioritize the most important guidance.

**[OFF-2]** General text instructions apply to all prompts. If an instruction is relevant only to a subset of prompts, it belongs in an example query or function, or in table comments/metadata.

**[OFF-2]** SQL expressions complement example SQL queries: expressions define reusable business concepts, while examples teach Genie how to approach common prompt formats. If users commonly ask for "a breakdown of performance", an example query can show that this means closed sales by region, sales rep and manager.

### 6.2 Full surface priority order **[GH-1]**

1. **Agent description** — set **first**; required for multi-agent routing.
2. **Metric View semantic metadata** when it already owns the business definition.
3. **Focused data source selection.**
4. **Table, Metric View and column descriptions** (`column_configs[].description`).
5. **Synonyms and display names** (`column_configs[].synonyms`).
6. **Format assistance and entity matching** (`enable_format_assistance` / `enable_entity_matching`), enabled **selectively** — never on IDs, hashes, free text, lat/long or raw measures. These toggles are space-only, so emit them even for a fully governed Metric View source.
7. **Hidden fields** (`column_configs[].exclude`).
8. **Join specs** for raw tables exposed together, only where evidence or user confirmation supports them.
9. **SQL snippets** for reusable filters/expressions/measures not already governed by Metric Views.
10. **Example SQL** for representative complex question patterns; instructive, not memorized benchmark answers.
11. **SQL functions** — trusted registered UC logic.
12. **Text instructions** — **last resort**.
13. **Sample questions and benchmarks.**

**[GH-1]** Surfaces 4–7 are applied **per column** through `data_sources.tables[].column_configs[]`. The array is optional, so an Agent created without it ships with none of these — build it explicitly during creation.

### 6.3 Vocabulary mapping **[GH-1]**

| Genie-UI / common term | Surface |
|---|---|
| Agent description / instructions header | Agent description (#1) |
| SQL expressions | SQL snippets (#9) |
| SQL queries / SQL instructions / trusted/certified SQL | Example SQL (#10) |
| SQL functions | SQL functions (#11) |
| General instructions / notes / text instructions | Text instructions (#12) |

---

## 7. Data profiling **[GH-1]**

Use workspace metadata first, then focused read-only SQL only when metadata is not enough.

**Phased inspection:** (1) **Structure** — objects, comments, columns, data types, constraints, sample rows with a narrow column list. (2) **Quality and usage** — nulls, empty strings, constants, distinct counts, casing issues, boolean-as-string values, sensitive/noisy columns, usage and lineage where system tables are accessible. (3) **Column profiling** — only columns that affect Genie quality: dates, likely filters, categorical strings, join keys, candidate measures. (4) **Readiness** — map findings back to the 3–5 business questions.

**Required signals per table/view:** row count, grain, freshness/date range, measures, dimensions, likely filters, data-quality caveats, sensitive/noisy fields, join candidates, and whether joins are supported by constraints, naming, row-count checks, query history or user confirmation.

**Required signals per Metric View:** governed measures, dimensions, filters, joins, time dimensions, display names, synonyms, formatting, comments, valid `MEASURE()` query patterns, and upstream semantic gaps.

**How to use findings:**

- Hide ETL metadata, all-null columns, raw blobs, embeddings, secrets, tokens and sensitive free text.
- Put high-null, constant, inconsistent-casing and boolean-as-string caveats in `DATA QUALITY NOTES` only when Genie needs them.
- Enable format assistance on useful dimensions and filters; enable entity matching only for stable low/medium-cardinality strings users are likely to mention.
- Use actual profiled values for example SQL parameters and benchmark literals.
- Use query history as evidence for joins, sample questions, examples and benchmarks.
- Ask the user to confirm metric formulas, joins, fiscal/calendar rules and default filters not supported by evidence.

**Related official behaviour [OFF-3]:** when you add data assets, Genie automatically searches for relevant popular workspace queries associated with them, using your credentials, and surfaces them for review in the **Data** tab. Accepted queries become example SQL. Suggestions won't appear if you lack access to the queries, no queries have run on the included tables, the queries only perform basic writes, or the relevant queries run against source tables not attached to the agent.

---

## 8. Knowledge store, metadata and prompt matching

### 8.1 What it contains **[OFF-2]**

Agent-level metadata customization (descriptions, business terms, synonyms); agent-level data customization (simplified, focused datasets without changing the underlying tables); prompt matching (format assistance and entity matching); join relationships; and SQL expressions. All configurations are scoped to the agent and do not affect Unity Catalog metadata or other assets.

**[OFF-1] [OFF-2]** These practices also improve usability for users who lack direct permissions on the underlying tables, and support quicker iteration when updating instruction versions.

### 8.2 Hiding columns **[OFF-2]**

Hiding removes a column from the agent's context, so Genie won't reference it when generating SQL or answering questions. Useful for unnecessary, duplicate or confusing columns. It does not affect the underlying data or Unity Catalog permissions — the column still exists and stays visible to users with direct table access. Columns can be hidden individually or in bulk.

### 8.3 Prompt matching **[OFF-2]**

Prompt matching lets Genie match the columns and values most relevant to the question and correct spelling issues in prompts. It is on by default and applied automatically to eligible columns as tables are added.

**Worked example:** for *"Show me car sales in Florida for Q1"* against data storing `FL` — without entity matching Genie may emit `WHERE state ILIKE '%Florida%'` (no results); with it, `WHERE state = 'FL'`.

- **Format assistance** provides representative values for all eligible columns, helping Genie understand data types and formatting patterns. Values are generated using the author's data permissions and become part of the agent's shared context.
- **Entity matching** provides curated lists of distinct values for columns where users reference specific entries. **String columns only.** Good candidates: state or country codes, product categories, status codes, department names. Format assistance must be on first; turning it off also disables entity matching. When users filter on an entity-matched column, the filter renders as an editable drop-down of stored values.

**Governance constraints [OFF-2]:** a **row filter** excludes the entire table from prompt matching; a **column mask** excludes only the masked columns. Genie prevents enabling entity matching on tables with row filters or column masks, but **authors must manually disable it for views that reference such tables and for dynamic views**.

**Refresh** prompt matching data when new values have been added or the format of existing values has changed **[OFF-2]**.

### 8.4 Knowledge mining **[OFF-2]**

Genie analyzes Unity Catalog metadata for connected tables and views; primary and foreign keys defined in your schema are automatically saved as join relationships. It also learns from author interactions — when an author thumbs-up a response or downloads query results, it analyzes the query and may suggest new SQL expressions and join relationships for the knowledge store.

---

## 9. Writing the surfaces well

### 9.1 SQL expressions **[OFF-2]**

Use them to provide structured definitions for KPIs and metrics, give Genie explicit context about how to calculate important values, define additional fields such as month or customer segment, and teach filters for business conditions.

Each is defined with a **name**, **code**, **synonyms** and **instructions**:

| Type | Behaviour | Documented example |
|---|---|---|
| **Filter** | Evaluates to a boolean condition | *High-value orders* — `orders.amount > 10000`; synonyms: large orders, big deals, significant orders; instruction notes the $10,000 threshold |
| **Measure** | Aggregation over multiple rows | *Win rate* — `COUNT(CASE WHEN stage = 'Closed Won' THEN 1 END) / NULLIF(COUNT(*), 0)`; synonyms: close rate, conversion rate; instruction notes the result is a decimal 0–1 |
| **Field** | Alters the value of each row | *Deal size* — `CASE WHEN amount < 10000 THEN 'Small' WHEN amount < 50000 THEN 'Medium' ELSE 'Large' END`; synonyms: deal tier, contract size, opportunity size |

### 9.2 Example SQL queries **[OFF-2]**

- Provide the SQL and use **the most typical phrasing of the user's question as the title**, written the way a user would naturally ask. This improves matching.
- Genie can use the example directly for a matching question, or take clues from it for a similar one.
- **Focus on samples that highlight logic unique to your organization and data.**
- Add **usage guidance** explaining when an example is particularly relevant.
- **Parameters:** String, Date, Date and Time, Decimal, Integer (default String). Give each a **comment** describing possible values or limits. If the input value doesn't match the selected type, Genie treats it as the incorrect type, which can lead to inaccurate results.
- **[OFF-6]** Aim for **at least five tested example SQL queries**.
- **[OFF-4]** Any good response can be promoted straight to an example via **Add as instruction**, which opens the save dialog pre-populated with the question and generated SQL.

**Trusted assets [OFF-2]:** example SQL queries and SQL functions that provide verified answers. In chat mode, when the exact text of a parameterized query is used, Genie returns a **verified answer** and users can edit the parameter and rerun. Users need `EXECUTE` on any SQL function used as a trusted asset. **[OFF-5]** Use trusted assets for mission-critical questions where responses must be reliable.

**SQL functions [OFF-2]:** for logic that cannot be captured with a static or parameterized query. Stored in Unity Catalog; scalar or table-valued. Genie cannot view or modify the SQL inside the function, making this suited to logic that should not be surfaced or changed.

### 9.3 Join relationships **[OFF-2]**

Define left and right tables, a join condition (e.g. `accounts.id = opportunity.accountid`, or a SQL expression for complex conditions), and a relationship type: **many to one**, **one to many**, or **one to one**. When multiple joins exist between the same tables, or self-joins are used, Genie automatically generates aliases for the right-hand table to avoid ambiguity.

**[OFF-5]** Prefer defining foreign key constraints in Unity Catalog. Use knowledge-store join relationships when FKs aren't specified, for complex cases such as self-joins, or when you lack permission to modify the underlying tables. Also provide example queries showing standard joins. If none of this resolves the problem, pre-join into a view and use that as the agent's input.

### 9.4 Qualify every column in SQL **[GH-1]**

**Always prefix every column reference with its source name** in SQL snippets and join specs — never emit a bare backtick-quoted column.

```sql
-- ✅ qualified
global_sales_assets_metrics.`Trade Date` = LAST_DAY(global_sales_assets_metrics.`Trade Date`)
MEASURE(global_sales_assets_metrics.`Gross Sales (ex Cash Management)`) / <Avg AUM>

-- ❌ bare (raises the error)
`Trade Date` = LAST_DAY(`Trade Date`)
MEASURE(`Gross Sales (ex Cash Management)`)
```

Unqualified columns raise **`Table name or alias is required for column`** when Genie composes a snippet into a larger statement. Qualifying is always safe and is **required** once multiple tables are joined. Prefix inside subqueries too. Do **not** prefix non-column tokens — string literals, `<placeholder>` markers, `:param_name` parameters, and conceptual CTE-result names stay as-is.

### 9.5 Text instructions

**Canonical structure [GH-1]** — five headers, in this order, omitting any that are empty:

| Header | What goes here |
|---|---|
| `## PURPOSE` | One or two bullets: scope and audience |
| `## DISAMBIGUATION` | Clarification triggers and term-resolution rules |
| `## DATA QUALITY NOTES` | NULL handling, known bad rows, semantics not captured in column descriptions |
| `## CONSTRAINTS` | Hard guardrails: what never to show (PII columns), what not to do |
| `## Instructions you must follow when providing summaries` | Rounding rules, mandatory caveats, date-range statements. **Use this exact heading — do not paraphrase it.** |

Rules **[GH-1]**: one `## Header` per section; dash bullets, one idea each; blank line between sections; no SQL inside bullets; keep the total **under 2,000 characters**; every bullet should reference a concrete asset (table, column, user phrase) or be a specific behavioral rule — vague guidance ("be helpful") is an anti-pattern.

**Example [GH-1]:**

```
## PURPOSE
- Answer questions about order revenue for FY2024 US retail orders.
- Users are merchandising managers — assume retail/e-commerce fluency.

## DISAMBIGUATION
- When the user asks about "customer performance" without a time range, ask them to clarify the period.
- "Q1" means calendar Q1 unless the user says "fiscal Q1".

## DATA QUALITY NOTES
- orders.order_amount is NULL for cancelled rows — filter with is_cancelled = false.

## CONSTRAINTS
- Never show PII columns (customer_email, customer_phone).

## Instructions you must follow when providing summaries
- Round percentages to two decimal places.
- Always state the date range used in the summary.
```

**Content the official docs put here [OFF-2]:** company-specific business information (e.g. fiscal year starts in February, so FY26 runs 1 Feb 2026 – 31 Jan 2027) and formatting rules (always respond in a given language; round decimals to two places by default; omit commas in any column including "Id"/"id"/"_id").

**Justification template before adding a text instruction [GH-1]:**

```
## Text Instruction Justification

- Exact instruction text:
- Why structured surfaces were insufficient:
- Intended global behavior:
- Possible overreach or regression risk:
- How the instruction will be reviewed or validated:
```

**[GH-1]** Do not use text instructions as the default home for guardrails, policies, metric logic, table-selection rules, join rules, filter rules, ranking/windowing rules or long best-practice lists. If the proposed instruction names specific tables, Metric Views, columns, joins, filters, denominators, numerators, aliases, ranking or window logic, encode it structurally or fix the upstream semantic model instead.

### 9.6 Clarification questions **[OFF-1]**

Four components: **trigger condition** ("When users ask about X topic…"), **missing details** ("…but don't include Y details…"), **required action** ("…you must ask a clarification question first…"), **example clarification** ("Please specify…"). Add these at the **end** of your general instructions so Genie prioritizes the behaviour.

### 9.7 Summaries **[OFF-1]**

Add a section at the end of the text instructions headed *"Instructions you must follow when providing summaries"*. Documented examples: respond in a specified language; cite the table and column names used; use bullet points for multi-part summaries; include the date range covered.

**Constraints:** only text instructions affect summary generation — SQL examples and SQL expressions don't. Some customizations aren't available, such as controlling summary length and detail level.

### 9.8 Timezone handling **[OFF-5]**

Genie can't always infer the timezone in the data or the one your analysis needs. Give explicit instructions naming the source timezone, the conversion function and the target timezone. Documented patterns: state that table times are `UTC` and instruct conversion via `convert_timezone('UTC', 'America/Los_Angeles', <timezone-column>)`; and, to define *today* for users in another zone, `date(convert_timezone('UTC', 'America/Los_Angeles', current_timestamp()))`.

---

## 10. Metric Views

**[GH-1]** (canonical deeper rules live in the `databricks-metric-views` skill):

- Treat Metric Views as governed semantic sources.
- **A single fact table is sourced directly** — do **not** build an intermediate base view for it; add dimension joins in the Metric View's `joins` block. Base views are for KPIs combining **multiple fact tables** or needing nested logic the Metric View cannot express.
- Do not attach underlying raw tables unless users also need raw-detail questions.
- Do not duplicate Metric View formulas in snippets or examples unless the example teaches a query *shape*. The formula lives once in the Metric View, referenced via `MEASURE()`.
- If the semantic model is wrong or missing a governed measure, dimension, join or filter, document it as an upstream modeling issue rather than working around it with broad Genie instructions.
- No `SELECT *` against Metric Views in examples or benchmarks.
- Wrap a Metric View query in a CTE before joining it to another source.

**[OFF-1]** Metric views pre-define metrics, dimensions and aggregations, which helps you stay within the object limit, simplifies the data model, and can improve accuracy.

**[OFF-3]** You can **export a Genie Agent as a metric view** from the agent's kebab menu — this creates a metric view based on the data and semantic context configured in the agent, and you can refine the definition with Genie Code before creating it.

**[OFF-5]** For metric calculation problems: define metrics as SQL expressions in the knowledge store; provide example SQL computing each roll-up when metrics aggregate from base tables; explain pre-computed aggregate tables in table comments and specify which further aggregations are valid; and create pre-aggregated views when the required SQL is very complicated.

---

## 11. Readiness assessment **[GH-1]**

Before proposing a live change, score each business question High/Medium/Low across four dimensions:

- **Semantic coverage** — measures, dimensions, filters and time fields exist.
- **Data quality and freshness** — important fields are populated, current, typed and usable.
- **Modelability** — grain and join paths are supported by evidence or user confirmation.
- **GenAI context readiness** — descriptions, synonyms, display names and prompt matching map business language to data.

| Level | Meaning |
|---|---|
| **High** | All required sources, fields, values and join/metric definitions are supported. |
| **Medium** | Answerable with caveats, missing descriptions, uncertain filters, or user-confirmed assumptions. |
| **Low** | Missing source, measure, dimension, time field, join path, or governed metric definition. |

Do not present Low-confidence questions as fully supported. Add data, revise the question, ask for confirmation, or mark the draft with limitations.

---

## 12. Testing, benchmarks and monitoring

### 12.1 Test it yourself **[OFF-1] [OFF-4]**

You should be your agent's first user. Test with realistic questions you expect business users to ask, and **carefully examine the SQL generated**. Click **Show code** to inspect any response's query. With CAN EDIT or greater you can edit the generated SQL, run it, and save it as an instruction via **Add as instruction**. Keep testing and editing until responses are reliable.

### 12.2 Feedback loop **[OFF-4]**

Each response asks **Is this correct?** — users answer **Yes**, **Fix it** (select a common issue or explain, then submit with or without regenerating), or **Request review** (flags for manual review with an optional comment). **The agent's behavior does not change from user feedback alone** — use feedback to identify improvement opportunities or answer users directly. Users with CAN MANAGE can review the exchange, comment, and confirm or correct the response.

### 12.3 Benchmarks **[OFF-4]**

Benchmarks are a set of test questions that score the agent's overall response accuracy. Up to **500 benchmark questions** per agent. Benchmark questions **run as new conversations** — no threaded context — each processed as a new query using the agent's instructions, example SQL and SQL functions.

**Two execution modes,** selected at run time (not per question), applying to every question in the run:

- **Chat mode** (default): accuracy is assessed by comparing Genie's SQL-generated results against a provided **SQL Answer**.
- **Agent mode**: runs with the same multi-step reasoning as Genie's Agent mode; an **LLM judge** grades responses, guided by an optional **evaluation note** which can reference expected content in the generated text report.

**Writing benchmark questions [OFF-4]:** reflect different ways of phrasing the common questions users ask. Optionally include a SQL query whose result set is the correct answer (Unity Catalog SQL functions can serve as gold-standard answers). Only questions that include a SQL Answer can be scored automatically — the rest need manual review. **Most Genie Agents should include between two and four phrasings of the same question**, using the same example SQL. **[OFF-6]** Add at least five benchmark questions based on anticipated user questions. **[GH-1]** If the agent is intended for eval-driven tuning, aim toward a **≥30 valid-item** bar (e.g. 2–4 phrasings per core question).

**Chat mode rating rules [OFF-4]:**

| Condition | Rating |
|---|---|
| Generated SQL exactly matches the provided SQL Answer | Good |
| Result set exactly matches the SQL Answer's result set | Good |
| Same data as the SQL Answer but sorted differently | Good |
| Numeric values round to the same 4 significant digits | Good |
| SQL produces an empty result set or returns an error | Bad |
| Result set includes extra columns versus the SQL Answer | Bad |
| A single-cell result differs from the SQL Answer's single-cell result | Bad |

**5,000-row caveat [OFF-4]:** Chat mode compares up to 5,000 rows per result set. If a result exceeds 5,000 rows and row order differs, truncation can rate a genuinely matching result **Bad**. Write SQL Answers returning fewer than 5,000 rows, or add `ORDER BY` to both sides so row order stays consistent.

**Reviewing runs [OFF-4]:** runs continue when you navigate away; results appear on the **Evaluations** tab with execution status, accuracy and creator. Open a run to compare **Model output** against **Ground truth**, with an explanation for each Bad rating. You can mark results Good/Bad manually and click **Update ground truth** to save a better response as the new ground truth. **Response results are visible in evaluation details for one week**; after that the generated SQL and example SQL remain but the results do not. You can rerun a subset of questions from a previous run to test improvements.

**[GH-1]** Benchmarks should be concrete and hardcoded, not parameterized. Avoid zero-row benchmark SQL unless the benchmark explicitly tests empty results. Keep sample questions user-facing, example SQL instructive and benchmarks evaluative — **do not copy benchmark questions or answer SQL into examples**.

### 12.4 Monitoring **[OFF-4]**

The **Monitor** tab shows every question and answer asked in the agent, filterable by time, rating, user or status. Identifying the questions Genie struggles with tells you where to add instructions. The **Weekly digest** shows weekly message volume, active users and thumbs up/down feedback.

**Conversation visibility [OFF-4]:** with **Reviewable by agent managers**, CAN MANAGE users can open the full exchange; for **Private** conversations they see only the user prompts. Conversations created before conversation sharing was enabled remain Private; newer ones default to reviewable. CAN MANAGE users can permanently delete a conversation for all users.

**[OFF-1]** Audit logs can also be used to monitor Genie Agent feedback and review requests.

### 12.5 User testing **[OFF-1]**

- Set expectations that the tester's job is to help refine the agent.
- Ask them to focus on the specific topic and questions the agent is designed to answer.
- On an incorrect response, encourage adding instructions and clarifications in chat; when the response is right, they should **upvote the final query** to minimize similar errors later.
- Tell users to upvote or downvote using the built-in mechanism, and to share unresolved questions with the authors.
- Consider providing training materials or a written testing/feedback guide.
- Business users must be members of the originating workspace to access the agent.

---

## 13. Deployment, lifecycle and API

### 13.1 Version control **[OFF-1]**

Use **Declarative Automation Bundles** to define, deploy and version-control Genie Agents as code. A bundle repository gives reproducible deployments, change history, and promotion across development, staging and production.

### 13.2 Agent settings worth configuring **[OFF-3]**

**Title** (discoverable in the workspace browser), **default warehouse** (pro or serverless; embeds the configuring author's compute credentials), **tags** (Public Preview; governed tags need ASSIGN), **thumbnail**, **description** (shown when users open the agent; supports Markdown), and **common questions** (optional examples on the chat landing page — author-defined ones appear first, with Genie filling remaining slots with auto-generated questions).

**Other actions [OFF-3]:** **Clone** (copies tables and settings, general instructions, example SQL queries and SQL functions — chat threads and Monitor data are *not* copied); **Export to metric view**; **Assign certification** (Certified / Deprecated / None, via the `system.certification_status` governed tag, requiring ASSIGN on that tag).

### 13.3 Programmatic creation **[OFF-6]**

`serialized_space` is a JSON string defining configuration and data sources, escaped as a string in the request. It contains `version`, `config.sample_questions`, `data_sources.tables` / `.metric_views`, `instructions` (`text_instructions`, `example_question_sqls`, `sql_functions`, `join_specs`, `sql_snippets`), and `benchmarks`.

**Validation rules — invalid JSON is rejected:**

- **Version:** required; use `2` for new agents.
- **IDs:** 32-character lowercase hexadecimal (UUID without hyphens). Required on sample questions, text instructions, example question SQLs, join specs, all three snippet types, and benchmark questions. The docs supply a one-line Python command that generates time-ordered IDs which sort correctly by construction.
- **Sorting:** collections must be pre-sorted and unsorted input is rejected — tables and metric views by `identifier`, `column_configs` by `column_name`, and sample questions, text instructions, example SQLs, join specs, snippets and benchmark questions by `id`; `sql_functions` by the `(id, identifier)` tuple.
- **Uniqueness:** sample-question and benchmark IDs unique across both collections; instruction IDs unique across all instruction types; `(table_identifier, column_name)` unique per agent.
- **Size:** strings ≤ 25,000 characters; repeated fields ≤ 10,000 items; **at most one text instruction per agent**.
- **Join specs:** the `sql` field takes **exactly two elements** — the join condition using backtick-quoted alias references, then a relationship annotation `--rt=FROM_RELATIONSHIP_TYPE_<CARDINALITY>--` with cardinality `MANY_TO_ONE`, `ONE_TO_MANY`, `ONE_TO_ONE` or `MANY_TO_MANY`. Omitting the annotation causes a parsing error. Multi-column joins need a separate join spec each.
- **Other:** table identifiers use the three-level namespace `catalog.schema.table`; each benchmark question needs exactly one answer with format SQL; filter, expression and measure SQL must not be empty.

**API usage practices [OFF-6]:** implement retry logic with exponential backoff (the API does not retry for you); log requests and responses; poll for status every 1–5 seconds until `COMPLETED`, `FAILED` or `CANCELLED`, capping most queries at 10 minutes, with exponential backoff up to one minute between polls; **start a new conversation for each session**, since reusing threads across sessions can reduce accuracy through unintended context reuse; and prune old conversations via the list and delete conversation endpoints to stay under the 10,000-conversation limit.

**[GH-1] CLI reference:**

```bash
# Resolve the warehouse first
databricks experimental aitools tools get-default-warehouse --profile <PROFILE>
databricks warehouses list --profile <PROFILE>   # if none running, ask the user to choose

# Profile sources — one call returns columns, types, sample rows, null counts, row count
databricks experimental aitools tools discover-schema catalog.schema.gold_sales catalog.schema.gold_customers

# Ensure parent_path exists first — create-space fails with "Tree node does not exist" otherwise
databricks workspace mkdirs /Workspace/Users/you@company.com/genie_spaces

# Create
databricks genie create-space --json "{ \"warehouse_id\": \"...\", \"title\": \"...\", \"description\": \"...\", \"parent_path\": \"...\", \"serialized_space\": $(cat genie_agent.json | jq -c '.' | jq -Rs '.') }"

# Inspect / update / delete
databricks genie list-spaces
databricks genie get-space SPACE_ID --include-serialized-space
databricks genie update-space SPACE_ID --json "{\"serialized_space\": $(cat genie_agent.json | jq -c '.' | jq -Rs '.')}"
databricks genie trash-space SPACE_ID
```

**[GH-1] CLI troubleshooting:** `sample_question.id must be provided` → add a 32-char hex `id` to each sample question. `Expected an array for question` → use `"question": ["text"]`, not a bare string. Field shape errors (`START_OBJECT`, `must be sorted by id`, `Unknown field`) → see the skill's `serialized-space.md`. No warehouse available → create a SQL warehouse or provide `warehouse_id`. Wrong or empty answers → add `example_question_sqls` and `text_instructions`.

---

## 14. Troubleshooting playbook **[OFF-5]**

| Symptom | Fix |
|---|---|
| **Misunderstood business jargon** (e.g. "year" means a fiscal year starting in February) | Add instructions that explicitly map your jargon to concepts Genie can understand. |
| **Incorrect table or column usage** | Provide clear, precise descriptions matching users' terminology; add example queries; remove or hide overlapping/unnecessary tables and columns. |
| **Filtering errors** (e.g. matching "California" when the table stores "CA") | Verify the relevant columns have example values and value dictionaries enabled, and refresh values when new data arrives. |
| **Incorrect joins** | Define foreign keys in Unity Catalog; otherwise define join relationships in the knowledge store; add example queries showing standard joins; as a last resort pre-join into a view. |
| **Column comments not syncing from foreign tables** | Edit column metadata in the agent UI, or create materialized views over federated tables and comment those (reusable across agents). |
| **Metric calculation issues** | Define metrics as SQL expressions; add example SQL for each roll-up; document pre-computed aggregates and valid further aggregations in table comments; create pre-aggregated views for very complex SQL. |
| **Incorrect time-based calculations** | Spell out source timezone, conversion function and target timezone in instructions (see §9.8). |
| **Ignoring instructions** | Add example queries using the tables correctly; hide irrelevant columns; create simpler views; remove irrelevant tables and instructions; start a new chat for a clean test. |
| **Performance issues / timeouts** | Check query history for slow queries and optimize the SQL rather than the agent config; encapsulate complex queries in trusted assets or views; shorten example SQL; start a new chat. |
| **Unreliable answers to mission-critical questions** | Use trusted assets to provide verified answers. |
| **Token limit warning** | See below. |
| **Cross-Geo processing not enabled** | Genie is a Designated Service using Databricks Geos for data residency; an account admin must enable cross-Geo processing for affected regions. |
| **Last agent author removed from the workspace** | Queries fail for everyone because the embedded compute credentials are invalid — another user with at least CAN EDIT must reconfigure the agent's SQL warehouse, which embeds their credentials instead. |

**Token limits [OFF-5]:** text instructions and metadata are converted into tokens. As the agent approaches the limit a warning appears; Genie then uses context filtering to prioritize the most relevant tokens, so responses still generate but quality may drop if important context is filtered out. **When the limit is exceeded you can no longer send or receive messages in the agent.** To reduce token count: remove unnecessary columns (create views excluding non-essential fields, or hide columns); streamline column descriptions and avoid restating the column name; edit column metadata in the agent; prune overlapping or redundant example SQL while keeping a diverse range; and simplify instructions.

---

## 15. Anti-patterns **[GH-1]**

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| Both the base view AND the Metric View in the same Agent | Genie sees unaggregated rows and must re-derive aggregation logic | Remove the base view once a Metric View exists on top of it |
| Adding 10 measures at once before testing | Can't isolate which one broke Genie's reasoning | Add and validate one at a time |
| Genie Agent with no description | Multi-agent routing fails silently | Always set an Agent description |
| Complex `CASE` chains in saved example SQL | Increases Genie's reasoning load on similar questions | Simplify to `WHERE` filters; lean on composed measures |
| Bare (unqualified) columns in SQL snippets or example SQL | Raises `Table name or alias is required for column` | Qualify every column (§9.4) |
| Prompt matching blanket-enabled on every column | Wastes context on IDs, hashes, free text and raw measures | Enable selectively, on categorical dimensions and filters users name directly |
| Assuming a governed Metric View source needs no `column_configs` | Format assistance and entity matching have no Metric View equivalent | Still emit `column_configs` enabling them on the relevant categorical dimensions |

**Also from the official docs:** vague text instructions and conflicting guidance across instruction types **[OFF-1]**; using text instructions to define metrics, filters or query patterns expressible in SQL **[OFF-1]**; manufacturing filler examples to hit a count **[GH-1]**; copying benchmark questions or answer SQL into examples **[GH-1]**; and leaving redundant example SQL in place as the agent nears the token limit **[OFF-5]**.

---

## 16. Pre-launch checklist

**Structural checks [GH-1]:**

- [ ] A description that states purpose and scope (required for multi-agent routing).
- [ ] A focused source set, ideally 5 or fewer at first.
- [ ] Descriptions that state business purpose and grain.
- [ ] Hidden ingestion, audit, hash, raw JSON, embedding and sensitive free-text fields.
- [ ] Prompt matching only on useful eligible categorical strings.
- [ ] Joins supported by constraints, naming, row-count checks or user confirmation.
- [ ] No long rulebook-style text instructions.
- [ ] Text instructions only for global behavior that cannot be encoded structurally, with justification.
- [ ] Example SQL that teaches reusable patterns, not memorized test questions.
- [ ] Example SQL parameters with real defaults and descriptions.
- [ ] Benchmarks with ground truth matched to the intended execution mode: checked SQL for deterministic Chat-style questions, evaluation notes for Agent-style multi-step analysis, both when a deterministic question also needs full-response judging.

**Official readiness markers [OFF-6]:** well-annotated Unity Catalog data with clear descriptive comments; user-tested with questions you expect from end users; company-specific context added as instructions, example SQL and functions, with **at least five tested example SQL queries**; and **at least five benchmark questions** based on anticipated user questions.

**Required output when proposing an Agent [GH-1]:** title or draft title; data sources included and why each belongs; per-question readiness confidence and data gaps; important metadata, prompt matching, join, snippet, example, sample question and benchmark choices; benchmark execution target and field strategy; assumptions or user confirmations needed before live creation; the read-only validation performed and any limitations; and any Metric View recommendation made for raw-table sources plus the user's decision.

---

## 17. Sources not yet incorporated

These are referenced by the sources above but were not directly readable when this document was compiled, so nothing here is drawn from them:

- **[GH-1] sibling reference files** — `serialized-space.md` (exact field schemas verified against the Genie API), `diagnose-genie-agent.md`, `optimize-genie-agent.md` (the source of the ≥30 benchmark-item bar cited in §12.3), `genie-agent-cicd.md`, `query-genie-agent.md`, and the `databricks-metric-views` skill. GitHub blocked direct fetches of these paths; they are in the same `references/` folder as `create-genie-agent.md`.
- **Official pages** — Genie Agents concepts, Agent mode and the Agent mode APIs, Analyze files in volumes with a Genie Agent, Genie Agents with dashboards, Embed a Genie Agent, Monitor usage with audit logs and alerts, Metric views, and the Declarative Automation Bundles resources reference.
