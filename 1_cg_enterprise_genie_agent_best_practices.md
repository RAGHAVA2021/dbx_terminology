# Enterprise Best Practices for Designing, Building, Testing, and Operating Databricks Genie Agents

**Document type:** Enterprise design and quality guide  
**Source basis:** Databricks-authored GitHub guidance and official Azure Databricks/Databricks product documentation only  
**Prepared:** 19 September 2026  
**Terminology:** Genie Agents were formerly known as Genie Spaces.

---

## 1. Purpose

This document consolidates Databricks guidance for creating a focused, governed, accurate, testable, and maintainable Genie Agent. It covers the lifecycle from requirements and source selection through semantic design, testing, approval, deployment, monitoring, and continuous refinement.

This is a source-based consolidation. It does not add independent architecture rules or unsupported recommendations. Where the referenced sources disagree, the difference is recorded explicitly rather than silently reconciled.

## 2. Authoritative references

The guide is based on the following sources:

1. [Create Genie Agent — Databricks Agent Skills GitHub reference, commit `e0af9245`](https://github.com/databricks/databricks-agent-skills/blob/e0af9245dd88c9d2c1c6a2de9f7133beac0cdb2d/skills/databricks-genie-agents/references/create-genie-agent.md)
2. [Curate an effective Genie Agent — Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/genie-agents/best-practices)
3. [Create and manage a Genie Agent — Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/genie-agents/set-up)
4. [Tune Genie Agent quality — Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/genie-agents/tune-quality)
5. [Test and monitor a Genie Agent — Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/genie-agents/monitor)
6. [Unity Catalog metric views — Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/uc-semantics/metric-views/)
7. [Declarative Automation Bundles resources — Azure Databricks](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/bundles/resources)

### 2.1 Source-role distinction

| Source | Role in this guide |
|---|---|
| Official Azure Databricks product documentation | Current product behavior, limits, permissions, configuration, quality tuning, testing, monitoring, and bundle resource fields |
| Databricks Agent Skills GitHub reference | Detailed design-time workflow, approval gates, profiling rules, readiness assessment, design priorities, SQL rules, static checks, anti-patterns, and CLI workflow |

### 2.2 Source versions and conflict-resolution rule

The GitHub file was included in release commit `e0af9245dd88c9d2c1c6a2de9f7133beac0cdb2d` on **18 September 2026 at 08:34:01 UTC**. That file-level release date does not establish when each individual statement was last changed. Git blame attributes both the 30-object statement and the no-`column_configs` default statement to commit `1c435ff8` on **28 August 2026 at 05:27:05 UTC**. The official Azure Databricks best-practices and quality-tuning pages used in those conflicts show **11 September 2026** as their last-updated date.

For a direct conflict, this guide applies the following rule:

1. Compare the last-change date of the specific conflicting statement, not merely the containing file's latest release date.
2. Use the value from the more recently modified source when both statement-level dates are available.
3. If an official webpage does not display a date, use the official webpage value.
4. Retain the superseded statement in a dated source note so that the decision remains traceable.

## 3. Executive principles

The referenced sources consistently establish the following principles:

1. Treat Genie as a new data analyst that requires clear, domain-specific context.
2. Define a specific audience, business purpose, and bounded question set.
3. Start small and keep each Agent focused on a business domain or subdomain.
4. Use concise, well-documented, simplified datasets.
5. Prefer governed and structured semantics over broad natural-language instructions.
6. Use Metric Views when reusable business metrics require centralized definitions.
7. Do not invent KPIs, joins, fiscal rules, default filters, or business terminology.
8. Configure table and column metadata, synonyms, hidden fields, and prompt matching deliberately.
9. Use verified SQL expressions and example SQL for business logic and complex query patterns.
10. Use text instructions only for global behavior that cannot be expressed structurally.
11. Test with realistic business questions, inspect generated SQL, and maintain benchmarks.
12. Obtain explicit approval before creating or updating a live Genie Agent when following the GitHub design workflow.
13. Version and promote the Agent using Declarative Automation Bundles when managing it as code.
14. Continue monitoring questions, feedback, reviews, and benchmark results after release.

## 4. Consolidated source-based lifecycle

The Databricks GitHub reference defines a gated design workflow. The official product documentation adds product configuration, testing, deployment, and operational practices. The table below consolidates those sources; it is not an independently published Databricks release standard.

| Stage | Expected result | Gate or control |
|---|---|---|
| 1. Gather requirements | Audience, purpose, real questions, known sources, terminology, KPI rules, and benchmark intent | Do not profile, write configuration, or run CLI creation commands until the minimum requirements are supplied |
| 2. Discover or confirm data | Focused source set mapped to business questions | Explain why each source belongs |
| 3. Check Metric View need | Decision to use an existing Metric View, create one separately, or continue with raw tables | Pause for user decision when reusable KPIs suggest a governed Metric View |
| 4. Inspect and profile | Metadata, grain, freshness, quality, useful values, sensitive fields, and relationships | Read-only and bounded investigation |
| 5. Inspect semantic sources | Measures, dimensions, filters, joins, time semantics, synonyms, and formats understood | Do not duplicate governed logic |
| 6. Assess readiness | High, Medium, or Low confidence for every real business question | Do not treat unresolved Low-confidence questions as supported |
| 7. Design Agent context | Description, sources, metadata, prompt matching, joins, SQL expressions, examples, functions, instructions, questions, and benchmarks | Structured context before text instructions |
| 8. Review draft | Static health checks and anti-pattern review completed | All gaps and assumptions exposed |
| 9. Obtain approval | Full proposed configuration reviewed | Do not create or update the live Agent without explicit approval under the GitHub workflow |
| 10. Create and configure | Agent created with the approved sources, warehouse, context, and permissions | Respect product requirements and access controls |
| 11. Test and evaluate | Realistic questions, reviewed SQL, benchmarks, and business-user testing | Correct errors before broader release |
| 12. Deploy and version | Reproducible configuration and controlled environment promotion | Use Declarative Automation Bundles where the Agent is managed as code |
| 13. Monitor and refine | Usage, feedback, review requests, benchmark results, and context improvements | Treat curation as iterative |

## 5. Requirements gate

Before profiling data or designing the Agent, collect the following:

1. **Target audience:** The intended users and their level of domain fluency.
2. **Agent purpose:** The business area, domain, or subdomain that the Agent covers.
3. **Three to five real business questions:** Concrete questions users will ask, not generic demonstrations.
4. **Known data sources:** Catalog, schema, table, view, or Metric View identifiers, or keywords to use for discovery.
5. **Terminology and KPI definitions:** Business vocabulary, metric definitions, fiscal conventions, default filters, and other domain rules.
6. **Benchmark intent:** Whether the Agent will be evaluated in Chat mode, Agent mode, or not initially benchmarked.

If any minimum requirement is missing, ask for it instead of assuming it. In particular, do not invent:

- Business definitions
- Join relationships
- KPI formulas
- Fiscal calendars
- Default filters
- Metric logic

## 6. Scope and Agent sizing

### 6.1 Focus by business domain

An Agent should serve a particular topic and audience. The sources advise organizing Agents by business domain or subdomain rather than by individual report. If a domain is too broad, divide it into subdomain-focused Agents.

The Agent description must state its purpose and scope. The GitHub reference also identifies the description as required for routing when a supervisor or multi-agent system delegates work to the appropriate Agent.

The GitHub reference also recommends assigning domain or subdomain tags to the Agent and its underlying tables or Metric Views for discoverability and observability. It identifies mirroring the hierarchy in Unity Catalog, such as one schema per domain or subdomain, as optional.

### 6.2 Start small

Both the official best-practices page and GitHub reference recommend starting with approximately five or fewer data objects. Include only the sources and columns needed to answer the supported questions. Expand iteratively based on testing, monitoring, and actual user needs.

### 6.3 Resolved source conflict: maximum number of data objects

The sources state different maximums:

| Source | Source date | Stated limit |
|---|---:|---:|
| Official Azure Databricks best-practices page | 11 September 2026 | Up to 50 tables, views, or Metric Views per Genie Agent |
| Databricks Agent Skills GitHub `create-genie-agent.md` — exact line last changed in commit `1c435ff8` | 28 August 2026 | Hard limit of 30 tables, views, or Metric Views per Genie Agent |

Applying the conflict-resolution rule in section 2.2, this guide uses the newer official value of **50 tables, views, or Metric Views as the current maximum**. The older GitHub value of 30 is retained in the table for traceability. The common design guidance remains unchanged: begin with five or fewer focused objects and keep the source set well below the maximum.

## 7. Technical prerequisites and access model

According to the official setup documentation:

- Agent data must be registered in Unity Catalog.
- A Genie Agent requires a pro or serverless SQL warehouse.
- Databricks recommends a serverless SQL warehouse for optimal performance.
- The creator or editor needs the Databricks SQL workspace entitlement.
- The creator or editor needs `CAN USE` on a suitable SQL warehouse.
- The creator or editor needs `SELECT` on the data used by the Agent.
- Creating or editing requires at least `CAN EDIT`; creators receive `CAN MANAGE` on Agents they create.
- Partner-powered AI features must be enabled at account and workspace levels.
- Each Agent supports up to 200,000 conversations.
- Each conversation supports up to 10,000 messages.

### 7.1 Separate compute and data identities

The official documentation describes two access paths:

- **Compute:** Queries use the warehouse credentials embedded by the author who configured the warehouse. Consumers do not require direct warehouse access.
- **Data:** Unity Catalog authorization is evaluated using each end user's identity. Users see only data for which they have permission, and their queries are attributed to them.

Consumers require the appropriate entitlement, at least `CAN VIEW` or `CAN RUN` on the Agent, and `SELECT` on the attached data objects. Questions involving inaccessible data can return an empty response.

Unity Catalog row filters and column masks continue to apply per user when the Agent is shared.

### 7.2 Attached-source boundary

An Agent uses only data sources explicitly attached under its Sources configuration. Referencing a table or Unity Catalog function only in instructions or metadata does not make it queryable. Access to every attached source remains governed by the end user's Unity Catalog permissions.

### 7.3 Dashboard filter behavior

When an existing Genie Agent is linked to a dashboard, dashboard filters do not carry into the Agent's chat context. The official best-practices page states that dashboard filters work as expected with Genie Agents created automatically when a dashboard is published.

## 8. Data discovery and profiling

### 8.1 Read-only constraint

The GitHub design workflow limits investigation to bounded, read-only SQL, including:

- `SELECT`
- `WITH`
- `SHOW`
- `DESCRIBE`
- `EXPLAIN`
- `information_schema`

It explicitly prohibits mutation of Unity Catalog objects or data during this workflow, including `CREATE`, `ALTER`, `DROP`, `INSERT`, `UPDATE`, `DELETE`, `MERGE`, and `COPY INTO`.

### 8.2 Phased inspection

Profile in phases:

1. **Structure:** Confirm each object, comments, columns, data types, constraints, and narrowly selected sample rows.
2. **Quality and usage:** Examine nulls, empty strings, constants, distinct counts, casing inconsistencies, boolean values stored as strings, sensitive or noisy fields, and usage or lineage signals where available.
3. **Relevant columns:** Focus detailed profiling on dates, filters, categorical values, join keys, and candidate measures that affect Genie quality.
4. **Readiness:** Map findings back to the three to five real business questions.

### 8.3 Required signals for tables and standard views

Identify:

- Row count
- Business grain
- Freshness and date range
- Measures and dimensions
- Likely filters
- Data-quality caveats
- Sensitive or noisy fields
- Candidate joins
- Evidence supporting joins, such as constraints, naming, row-count checks, query history, or user confirmation

### 8.4 Required signals for Metric Views

Identify:

- Governed measures
- Dimensions and filters
- Joins
- Time dimensions
- Display names and synonyms
- Formatting
- Comments
- Valid `MEASURE()` query patterns
- Any upstream semantic gap

### 8.5 Apply profiling results

Use findings to:

- Hide ingestion and ETL metadata, all-null columns, audit fields, raw blobs or JSON, embeddings, hashes, secrets, tokens, and sensitive free text when they are not appropriate for end-user questions.
- Record material null, constant, casing, or type caveats in data-quality notes only when Genie needs the information.
- Use actual profiled values for example parameters, benchmark literals, and sample wording.
- Use query history as evidence for joins, examples, sample questions, and benchmarks when it is available.
- Ask the user to confirm unsupported formulas, joins, fiscal rules, calendar rules, and default filters.

## 9. Metric View decision and semantic governance

Metric Views are the Unity Catalog implementation for centrally defining business metrics. They separate measures from dimensions used to group, filter, and aggregate results, allowing a metric to be defined once and queried at runtime.

### 9.1 Decision gate

Before profiling raw sources in depth, determine whether the intended questions center on reusable KPIs such as revenue, margin, or conversion rate.

- If they do, the GitHub workflow requires pausing and recommending a governed Metric View first.
- State that the benefits are consistent definitions, reduced duplicated SQL, and improved Genie reasoning.
- Continue with raw tables only when the user declines or the questions require raw-detail access.
- Creating or altering the Metric View is outside the `create-genie-agent.md` workflow and is handled through the separate Databricks Metric View process.

### 9.2 Use Metric Views as governed sources

- For a Metric View with a single fact table, the GitHub reference directs authors to source the fact table directly and add dimension relationships in the Metric View's `joins` block rather than creating an intermediate base view.
- Use a base view when KPIs combine multiple fact tables or require nested logic that the Metric View cannot express directly.
- Prefer the Metric View's semantic metadata over duplicated formulas in the Agent.
- Do not attach the underlying raw table merely to duplicate the Metric View's aggregation logic; include raw data only when raw-detail questions require it.
- Do not reproduce a governed formula as an SQL expression or example merely to redefine it.
- If a measure, dimension, join, or filter is missing or incorrect, record an upstream semantic-model gap rather than hiding the problem in broad text instructions.
- Do not use `SELECT *` against a Metric View in examples or benchmarks.
- When combining a Metric View result with another source, the GitHub reference instructs wrapping the Metric View query in a CTE before joining.

## 10. Readiness assessment

Assess every real business question across four dimensions:

1. **Semantic coverage:** Required measures, dimensions, filters, and time fields exist.
2. **Data quality and freshness:** Important fields are populated, current, correctly typed, and contain usable values.
3. **Modelability:** Grain and join paths are supported by evidence or user confirmation.
4. **GenAI context readiness:** Descriptions, synonyms, display names, and prompt-matching choices connect business language to the data.

### 10.1 Confidence levels

| Level | Meaning | Required treatment |
|---|---|---|
| High | Required sources, fields, values, joins, and metric definitions are supported | May be presented as supported |
| Medium | Answerable with caveats, missing descriptions, uncertain filters, or explicitly confirmed assumptions | Document caveats and confirmations |
| Low | A source, measure, dimension, time field, join path, or governed definition is missing | Add data, revise the question, obtain confirmation, or state the limitation; do not present it as fully supported |

Do not proceed to the final design with unresolved Low-confidence questions presented as supported capabilities.

## 11. Structured-context design order

The GitHub reference defines the following priority order:

1. Agent description
2. Existing Metric View semantic metadata
3. Focused data-source selection
4. Table, Metric View, and column descriptions
5. Synonyms and display names
6. Format assistance and entity matching
7. Hidden fields
8. Supported join specifications for raw tables
9. SQL expressions for reusable filters, fields, or measures not already governed
10. Example SQL for representative complex query patterns
11. Trusted Unity Catalog SQL functions
12. Text instructions as a last resort
13. Sample questions and benchmarks

This ordering places governed and structured context ahead of free-form text.

## 12. Table and column configuration

### 12.1 Descriptions and synonyms

Provide precise descriptions that explain business purpose, meaning, and grain. Agent-specific descriptions and synonyms do not overwrite Unity Catalog metadata.

Synonyms should map natural business language to the relevant columns or SQL expressions. Review any AI-generated descriptions before accepting them.

### 12.2 Configure and review `column_configs`

The GitHub reference emphasizes creating `column_configs` explicitly during serialized Agent creation. It is the serialized Agent surface used for per-column:

- Descriptions
- Synonyms
- Format assistance
- Entity matching
- Exclusion of hidden columns

Even when the source is a governed Metric View, format assistance and entity matching remain Agent-level column settings. The current official documentation separately states that prompt matching is enabled automatically for eligible columns when tables are added; authors can manage those settings per column.

### 12.3 Resolved source conflict: prompt-matching defaults

The sources describe different default behavior:

| Source | Source date | Statement |
|---|---:|---|
| Official Azure Databricks quality-tuning page | 11 September 2026 | Prompt matching is enabled by default, and eligible columns receive format assistance and entity matching automatically when tables are added |
| Databricks Agent Skills GitHub `create-genie-agent.md` — exact lines last changed in commit `1c435ff8` | 28 August 2026 | An Agent created without `column_configs` ships without per-column descriptions, synonyms, format assistance, entity matching, or hidden-field settings |

Applying the conflict-resolution rule in section 2.2, this guide uses the newer official behavior: prompt matching is enabled automatically for eligible columns as tables are added. Authors should review and manage the per-column settings. The older GitHub statement is retained for traceability, while its recommendation to configure `column_configs` explicitly remains applicable to deliberate serialized Agent design.

### 12.4 Hide irrelevant columns

Hide unnecessary, duplicate, technical, confusing, or sensitive columns from the Agent's context. Hiding a column does not delete it, change Unity Catalog metadata, or replace Unity Catalog permissions.

### 12.5 Review suggested workspace queries

When sources are added, Genie can search for relevant popular workspace queries associated with those sources. Review each suggestion and accept only relevant queries. Suggested queries require appropriate query access, are limited to attached sources, and may be unavailable when no relevant query has run or the curator lacks access.

## 13. Prompt matching

Prompt matching helps Genie align user wording and misspellings with actual columns and stored values. It is enabled automatically for eligible columns according to the newer official documentation. Authors can manage the per-column settings, and the GitHub design workflow recommends representing deliberate serialized choices through `column_configs`.

### 13.1 Components

- **Format assistance:** Supplies representative values to help Genie recognize data types and formatting patterns.
- **Entity matching:** Supplies curated distinct values for categorical string columns that users are likely to name directly.

The official tuning documentation states that entity matching can cover up to 120 columns; each column can include up to 1,024 distinct values, with each value up to 127 characters.

### 13.2 Selection rules

The GitHub reference instructs enabling these settings selectively for useful categorical dimensions and filters. Do not blanket-enable them for identifiers, hashes, free text, latitude/longitude, or raw measures.

Examples identified by the official tuning documentation as suitable for entity matching include:

- State or country codes
- Product categories
- Status codes
- Department names

Entity matching supports string columns and requires format assistance to be enabled.

### 13.3 Security and policy behavior

Representative values are generated using the author's data permissions and become part of the Agent's shared context. The official documentation states:

- A row filter excludes the entire table from prompt matching.
- A column mask excludes the masked column from prompt matching.
- Agent authors must disable entity matching for views that reference row-filtered or masked tables and for dynamic views.

Refresh stored prompt-matching values when new values appear or formats change.

## 14. Join relationships

Define relationships so Genie can produce accurate joins. For each relationship, supply:

- Left and right objects
- Qualified join condition
- Relationship type: many-to-one, one-to-many, or one-to-one

Add a relationship only when constraints, naming, row-count checks, query history, or user confirmation supports it. Do not invent relationships.

## 15. Choosing the correct instruction surface

| Need | Preferred surface |
|---|---|
| Reusable business metric, filter, or calculated field | SQL expression, unless already governed by a Metric View |
| Complex or ambiguous question pattern | Verified example SQL query |
| Logic that requires a reusable registered function | Unity Catalog SQL function |
| Business meaning of a table or column | Table or column description |
| User vocabulary | Synonym or display name |
| Mapping user-entered values to stored categorical values | Prompt matching |
| Raw-table relationship | Join specification |
| Global ambiguity handling, summary formatting, or caveat | Concise text instruction |

### 15.1 SQL expressions

SQL expressions provide structured definitions for:

- Filters
- Measures
- Fields or dimensions

Use them for reusable business concepts that are not already governed by a Metric View. Supply a name, SQL expression, synonyms, and specific usage instructions.

### 15.2 Example SQL

Use example SQL for complex, multi-part, or ambiguous questions. The title should use the normal phrasing a business user would use. Examples should teach representative query shapes rather than reproduce benchmark answers.

Cover only the distinct shapes the supported questions require, such as:

- Simple aggregation
- Grouping by a dimension
- Time filtering or window logic
- Ratio or `MEASURE()` composition
- Ranking
- CTE followed by a join

There is no fixed minimum number of SQL expressions, examples, or benchmarks. Size them by coverage rather than quota.

### 15.3 Parameterized examples

For every parameter, provide:

- A description or comment
- A supported type
- A real default value derived from the data

The official documentation lists String, Date, Date and Time, Decimal, and Integer as supported parameter types. An input that does not match its configured type can lead to inaccurate results.

### 15.4 Trusted assets

The official documentation identifies parameterized example queries and Unity Catalog SQL functions as trusted assets. When Genie uses the exact parameterized query text in Chat mode, it can return a verified answer. Users need `EXECUTE` on any attached SQL function.

## 16. SQL quality rules

### 16.1 Qualify column references

The GitHub reference requires every column in SQL expressions and join specifications to be prefixed with its source name or an explicit alias, including references inside subqueries. Unqualified columns can fail when Genie composes an expression into a larger statement.

Do not prefix string literals, parameter markers, placeholders, or conceptual names that are not source columns.

### 16.2 Validate SQL before saving

Validate example SQL, benchmark SQL, expressions, and joins using bounded read-only execution or `EXPLAIN` when possible. Do not add benchmark SQL without first checking it.

### 16.3 Keep examples and benchmarks separate

- Sample questions are user-facing discovery aids.
- Example SQL teaches reusable solution patterns.
- Benchmarks evaluate accuracy.

Do not copy benchmark questions or their answer SQL into examples. Benchmarks should be concrete rather than parameterized. Avoid zero-row benchmark SQL unless empty results are the intended test.

## 17. Text instructions

### 17.1 Last-resort rule

Use text instructions only for global context that cannot be represented through sources, Metric View semantics, metadata, synonyms, prompt matching, joins, SQL expressions, example SQL, functions, or an upstream semantic-model correction.

Do not use a long text rulebook to define table-selection logic, joins, KPI formulas, filters, ranking rules, or window logic when a structured surface can carry the rule.

### 17.2 Characteristics of acceptable instructions

- Clear and specific
- Free from conflicts with SQL expressions and examples
- Limited and focused
- Applicable globally
- Explicit about when clarification is required and what question should be asked

Too many instructions can reduce effectiveness, particularly in longer conversations.

### 17.3 Recommended organization from the GitHub reference

When needed, use the following headings in this order and omit empty sections:

```markdown
## PURPOSE

## DISAMBIGUATION

## DATA QUALITY NOTES

## CONSTRAINTS

## Instructions you must follow when providing summaries
```

The GitHub reference further specifies:

- One idea per bullet
- Blank lines between sections
- No SQL in text-instruction bullets
- Concrete asset references or precise behavioral rules
- Total text kept under 2,000 characters

The official best-practices page also requires the exact heading `Instructions you must follow when providing summaries` when customizing summary behavior.

### 17.4 Clarification behavior

A clarification instruction should define:

1. The trigger condition
2. The missing details
3. The required action to ask a question first
4. The exact or representative clarification question

Place clarification guidance at the end of the general instructions so Genie can prioritize the behavior.

### 17.5 Instruction justification

The GitHub reference requires documenting the following when proposing or editing text instructions:

```markdown
## Text Instruction Justification

- Exact instruction text:
- Why structured surfaces were insufficient:
- Intended global behavior:
- Possible overreach or regression risk:
- How the instruction will be reviewed or validated:
```

## 18. Product and design limits

The referenced sources state the following limits. The 50-object maximum is the result of the statement-level dated conflict resolution in section 6.3; the other limits come from the official setup, tuning, and monitoring documentation.

| Configuration area | Limit |
|---|---:|
| Attached tables, views, or Metric Views — current maximum selected in section 6.3 | 50 per Agent |
| Instructions | 100 per Agent |
| Knowledge-store snippets | 200 per Agent |
| Benchmark questions | 500 per Agent |
| Conversations | 200,000 per Agent |
| Messages | 10,000 per conversation |

For the instruction limit, every example SQL query, SQL function, and the entire general-instructions block each count as one instruction.

For the knowledge-store limit, table descriptions, join relationships, and SQL expressions share the 200-snippet capacity. The official tuning page separately notes that column descriptions and prompt-matching settings do not count toward this snippet limit.

## 19. Sample questions

Common questions appear on the Agent's landing page. Author-defined questions take priority; when fewer are supplied than the interface can display, Genie can fill the remaining positions with generated questions.

Choose questions that reflect the Agent's defined audience, scope, and realistic supported workflows. They are not substitutes for benchmarks.

## 20. Testing and benchmarks

### 20.1 Test as the first user

Before release:

- Ask realistic questions expected from business users.
- Review the natural-language result, result table, visualization, and generated SQL.
- Correct generated SQL where necessary.
- Save corrected and reusable patterns as instructions or examples.
- Test variations in wording.

Genie is nondeterministic, so repeated identical prompts can sometimes produce different outputs. Verified examples can improve consistency.

### 20.2 Benchmark modes

| Mode | Evaluation basis |
|---|---|
| Chat mode | Compares generated SQL results with the result of a supplied SQL answer |
| Agent mode | Uses Agent mode's multi-step reasoning and an LLM judge; an optional evaluation note can guide grading |

Benchmark questions run as new conversations and do not inherit threaded chat context.

### 20.3 Benchmark construction

- Cover frequently asked and business-critical questions.
- Use checked SQL ground truth for deterministic Chat-mode questions.
- Use evaluation notes for Agent-mode analysis.
- Include both when a deterministic question also requires judging the complete response.
- Cover relevant sources, filters, measures, joins, time logic, answer shapes, evidence, and synthesis.
- Use multiple realistic phrasings where appropriate.

The GitHub reference does not impose a minimum at creation. For later evaluation-driven optimization, it points toward at least 30 valid benchmark items, while the official product documentation allows up to 500 benchmark questions per Agent.

### 20.4 Run and review evaluations

Users with at least `CAN EDIT` can run all benchmarks or a selected subset. Review model output against ground truth, inspect explanations for incorrect results, and manually mark results Good or Bad when required.

The official monitoring documentation states that detailed benchmark response results remain visible for one week; generated SQL and example SQL remain afterward.

## 21. Business-user testing and feedback

After author testing:

- Recruit business users who represent the defined audience.
- Ask them to test only the intended topic and supported questions.
- Set the expectation that testing helps refine the Agent.
- Encourage built-in positive and negative feedback.
- Encourage review requests for disputed responses.
- Capture unresolved questions for curators.

User feedback alone does not automatically change Agent behavior. Authors and managers should review feedback and apply appropriate context changes deliberately.

## 22. Monitoring and continuous improvement

Use the Monitor experience to review questions, feedback, and review requests according to the permissions and conversation-sharing behavior documented by Databricks.

For incorrect answers:

1. Inspect the generated SQL.
2. Determine whether the cause is missing metadata, terminology, a relationship, a semantic definition, an example pattern, or an upstream model gap.
3. Edit the SQL when appropriate.
4. Save a reusable correction as an example or other suitable structured context.
5. Rerun the relevant benchmark subset.

Genie Code can propose context changes after a failure or benchmark run. Review proposed changes and accept only the context that should be retained.

### 22.1 Knowledge-mining recommendations

The official quality-tuning page states that primary and foreign keys defined in the schema can be saved automatically as join relationships in the Agent. Genie can also analyze author interactions, including an author's positive feedback or result download, and suggest SQL expressions or additional join relationships. Review every proposed knowledge-store change before retaining it.

## 23. Deployment and version control

The official best-practices documentation recommends Declarative Automation Bundles to define, deploy, version, and promote Genie Agents through development, staging, and production.

### 23.1 Bundle requirements recorded in official documentation

- Genie Agents are represented under the `genie_spaces` bundle resource key.
- Genie Agent bundle resources require the direct deployment engine.
- Official documentation states support was added in Databricks CLI version 1.3.0.
- The resource includes fields such as `description`, `file_path`, `lifecycle`, `parent_path`, `permissions`, `serialized_space`, `title`, and required `warehouse_id`.
- When `file_path` is supplied, it takes precedence over `serialized_space`.
- The serialized definition contains the Agent's sources, instructions, and sample questions.

### 23.2 Promotion objective

The official best-practices page identifies the purpose of bundle-based management as:

- Reproducible deployments
- Change history
- Version control
- Promotion across development, staging, and production

## 24. Approval and change controls

The GitHub workflow contains explicit pauses:

1. **Requirements gate:** Do not begin data profiling or configuration work without the minimum audience, purpose, real questions, and known-source information.
2. **Metric View gate:** Pause when reusable KPIs indicate that a governed Metric View should be considered.
3. **Live-change gate:** Present the full draft and obtain explicit approval before running create or update operations.

The design review should expose:

- Proposed title and description
- Included sources and reason for inclusion
- Per-question readiness level
- Data and semantic gaps
- Metadata and synonym choices
- Hidden columns
- Prompt-matching choices
- Join definitions and supporting evidence
- SQL expressions
- Example SQL and parameters
- SQL functions
- Text instructions and justification
- Sample questions
- Benchmarks and evaluation mode
- Required user confirmations
- Read-only validation performed and limitations
- Metric View recommendation and the recorded decision

## 25. Static health checks

Before proposing a live change, verify:

- [ ] The Agent has a clear description of purpose and scope.
- [ ] The audience and business domain are explicit.
- [ ] The initial source set is focused, ideally five or fewer objects.
- [ ] The total source set does not exceed the current 50-object maximum.
- [ ] Each source maps to one or more real business questions.
- [ ] Every table or Unity Catalog function referenced by the Agent is explicitly attached.
- [ ] Descriptions explain business purpose and grain.
- [ ] Unnecessary technical, audit, ingestion, raw, embedding, and sensitive fields are hidden.
- [ ] Synonyms map actual business vocabulary to the correct data.
- [ ] Automatically enabled prompt-matching settings have been reviewed for eligible columns.
- [ ] `column_configs` explicitly records the per-column choices required by the serialized design.
- [ ] Prompt matching is enabled only on useful eligible categorical strings.
- [ ] Prompt-matching security conditions have been reviewed.
- [ ] Domain or subdomain tags have been considered for the Agent and its sources.
- [ ] Joins are supported by evidence or user confirmation.
- [ ] Reusable KPIs have been assessed for Metric View governance.
- [ ] Governed Metric View formulas are not duplicated unnecessarily.
- [ ] SQL expressions and joins use qualified column references.
- [ ] Example SQL teaches reusable query patterns.
- [ ] Every example parameter has a type, description, and real default.
- [ ] SQL examples, expressions, joins, and benchmark answers were checked with read-only execution or `EXPLAIN` where possible.
- [ ] Text instructions are concise, global, non-conflicting, and justified.
- [ ] Sample questions, examples, and benchmarks have separate purposes.
- [ ] Every business question has a readiness rating.
- [ ] Low-confidence questions are resolved or explicitly limited.
- [ ] Benchmark ground truth matches the selected evaluation mode.
- [ ] Dashboard-linked use cases account for the documented filter-context behavior.
- [ ] The full draft has been presented for approval.

## 26. Source-defined anti-patterns

| Anti-pattern | Source-defined problem | Source-defined correction |
|---|---|---|
| Adding the base view and its Metric View without a raw-detail requirement | Genie can see unaggregated rows and re-derive governed aggregation logic | Remove the base source when the Metric View covers the required questions |
| Adding many measures before testing | Makes it difficult to identify which change affected reasoning | Add and validate incrementally |
| Omitting the Agent description | Breaks or weakens multi-agent routing | Always set purpose and scope in the description |
| Using complex `CASE` chains in saved examples | Increases reasoning burden for similar questions | Simplify patterns and use governed or composed measures where applicable |
| Using bare column references | Can fail when Genie composes SQL into a larger statement | Qualify columns with a source name or alias |
| Enabling prompt matching on every column | Uses context on IDs, hashes, free text, and raw measures | Enable it selectively for useful categorical filters and dimensions |
| Assuming automatic prompt-matching defaults require no review | Defaults might not represent the intended categorical dimensions, filters, or exclusions | Review eligible columns and manage their settings; record deliberate serialized choices in `column_configs` |
| Putting metrics, filters, joins, and query rules into long text instructions | Natural-language guidance is less structured and can conflict | Prefer Metric Views, metadata, joins, SQL expressions, examples, and functions |
| Conflicting text and SQL guidance | Produces less predictable behavior | Align all instruction surfaces |
| Treating user feedback as an automatic training update | Feedback alone does not change the Agent | Curators must review and apply appropriate context changes |
| Referencing an unattached source only in instructions or metadata | Genie does not query sources that are not explicitly attached | Attach every required data object or function under Sources |
| Assuming dashboard filters carry into an existing linked Agent | Existing linked Agents do not inherit dashboard filter context | Design and test the Agent without assuming those filters are present |

## 27. Definition of a review-ready draft

The GitHub reference requires the proposed output to include:

- Genie Agent title or draft title
- Included sources and justification
- Readiness confidence for every real question
- Data and semantic gaps
- Metadata, prompt matching, join, SQL expression, example, sample-question, and benchmark choices
- Benchmark mode and field strategy when benchmarks are included
- Assumptions and confirmations required before live creation or update
- Read-only validation completed and any limitations
- Metric View recommendation and the user's decision

## 28. Consolidated source-based release-readiness checklist

The following checklist consolidates the referenced guidance for use before broader release. It is not an independently named Databricks release standard.

- Technical prerequisites and permissions are satisfied.
- The Agent is focused on a defined audience and domain.
- Sources and columns are limited to what the supported questions require.
- Business semantics are represented through the most structured available surface.
- Metadata, synonyms, hidden fields, and prompt matching have been reviewed.
- Relationships and SQL have been validated.
- Unsupported or Low-confidence questions are not presented as supported.
- Realistic author testing has been completed.
- Benchmarks appropriate to the intended mode have been run and reviewed where evaluation is in scope.
- Business-user testing and feedback collection have been planned or completed.
- The approved configuration is versioned and promoted through the chosen deployment process.
- Monitoring ownership and iterative refinement are established through the product's review and monitoring capabilities.

## 29. CLI workflow from the GitHub reference

The GitHub reference documents the following CLI sequence:

1. Resolve the SQL warehouse before create or update.
2. Use the auto-detected default warehouse unless the user specifies another.
3. If no running warehouse is available, list warehouses and ask the user to choose.
4. Use schema discovery and bounded queries for profiling.
5. Ensure the parent workspace path exists before Agent creation.
6. Create the Agent from a local serialized JSON definition.
7. Use list and get operations to inspect Agents.
8. Use update only after the approved configuration is ready.
9. Use the documented trash operation for deletion.

The same GitHub reference reiterates that creation and update must not occur before explicit approval.

## 30. Traceability matrix

| Practice area | GitHub create guide | Official best practices | Setup | Tune quality | Test/monitor | Metric Views | Bundles |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Requirements and approval gates | ✓ |  |  |  |  |  |  |
| Start small and stay focused | ✓ | ✓ |  |  |  |  |  |
| Data-object product limit | ✓ | ✓ | ✓ |  |  |  |  |
| Attached-source boundary |  |  | ✓ |  |  |  |  |
| Dashboard filter behavior |  | ✓ |  |  |  |  |  |
| Read-only profiling | ✓ |  |  |  |  |  |  |
| Readiness assessment | ✓ |  |  |  |  |  |  |
| Metadata and synonyms | ✓ | ✓ | ✓ | ✓ |  |  |  |
| Prompt matching | ✓ | ✓ |  | ✓ |  |  |  |
| Joins and SQL expressions | ✓ | ✓ |  | ✓ |  |  |  |
| Example SQL and functions | ✓ | ✓ | ✓ | ✓ | ✓ |  |  |
| Text instructions as last resort | ✓ | ✓ |  | ✓ |  |  |  |
| Metric governance | ✓ | ✓ |  |  |  | ✓ |  |
| Benchmarks and evaluation | ✓ | ✓ |  | ✓ | ✓ |  |  |
| Permissions and sharing |  |  | ✓ | ✓ | ✓ |  | ✓ |
| Monitoring and feedback |  | ✓ |  |  | ✓ |  |  |
| Knowledge-mining recommendations |  |  |  | ✓ |  |  |  |
| Version-controlled deployment |  | ✓ |  |  |  |  | ✓ |

---

## 31. Final enterprise summary

A quality Genie Agent is not created merely by attaching tables. The referenced guidance recommends a focused business purpose, confirmed semantics, well-documented and simplified sources, deliberate column configuration, structured business logic, validated SQL, realistic evaluation, explicit approval within the GitHub workflow, controlled deployment, and ongoing monitoring.

The consolidated sequence used in this guide is:

Define the audience and questions → choose focused governed sources → profile and assess readiness → encode structured semantics → add minimal global instructions → validate with realistic questions and benchmarks → obtain approval → deploy reproducibly → monitor and refine.

Every business definition must be supported by governed metadata, verified SQL, source evidence, or explicit user confirmation. Unsupported assumptions must not be presented as established business logic.
