# Synapse-to-Databricks Bridge: Executive Implementation Challenges and Extraction Options

> Executive briefing for the interim full-load bridge from Azure Synapse dedicated SQL pool to Azure Databricks through ADLS Gen2.

## 1. Executive summary

The implementation is feasible, but it is not simply a file-copy exercise. The principal risks are:

1. **Source consistency:** separate extraction queries do not automatically read one common source snapshot.
2. **Shared Synapse capacity:** CETAS, Copy Activity partitions, Databricks Sqldw connector requests, direct JDBC/SQL reads and business queries consume the same dedicated-pool resources.
3. **Data compatibility:** CETAS-to-Parquet cannot safely export every source value or type without profiling and approved conversions.
4. **ADLS reliability and security:** identity, firewall, DNS, ACL, immutable-path, orphan-file, throttling and cleanup failures can stop or corrupt the bridge.
5. **Retry safety:** a failed or timed-out operation cannot be repeated blindly into the same folder or Delta target.
6. **Publication consistency:** creating the files or Delta tables does not make a collection of tables visible atomically.
7. **Operational scale:** hundreds of independent objects require metadata-driven orchestration, capacity scheduling, reconciliation, alerting and runbooks.

### Recommended extraction policy

- Use **CETAS as the default bulk-export route** for compatible objects.
- Use **ADF/Synapse Copy Activity as an explicitly approved fallback**, not an automatic retry route, when CETAS compatibility or permission requirements are unacceptable.
- Treat the **Azure Databricks Synapse/Sqldw connector as a controlled optional route**, not as ordinary JDBC and not as the default for all objects. Approve it only after a runtime, security, temporary-storage and full-volume proof of concept.
- Use **direct SQL/JDBC download only as an exception** for small, bounded, independently publishable objects or controlled diagnostics. It should not be the estate-wide full-load design.
- Require every route to use the same source-generation fence, explicit projection/schema contract, immutable attempt path, reconciliation and publication gates.

## 2. What executives should understand

### This is a temporary distributed system

The bridge spans Synapse, an orchestration/control database, ADLS Gen2 and Databricks. There is no transaction covering all four platforms. Each platform can succeed while another fails or loses the acknowledgement. Recovery therefore depends on durable identities, manifests, conditional state changes and platform history—not on a pipeline's final green/red status alone.

### Full load is operationally expensive

Every run reads the complete enabled source inventory, writes a complete transport copy, creates new Delta data and retains previous state for rollback. Cost and duration grow with compressed bytes, file counts, retries, retention and concurrent downstream use—not simply with the number of tables.

### Capacity is shared, not route-specific

At `DW2000c`:

- the documented dynamic resource-class maximum is **32 concurrent queries**;
- a workload group can reach a documented service-level maximum of **48 concurrent queries** only at a minimum **2% request resource grant**; and
- the Copy connector documentation warns that excessive source partition parallelism can exceed the source query limit and cause Synapse throttling.

These are platform ceilings, not recommended operating targets. A small resource grant can increase concurrency while making individual large exports slower. CETAS requests, Copy Activity partitions, Sqldw connector source operations, JDBC partitions, operational queries and any permitted user workload must fit within one tested aggregate source-read budget.

## 3. Extraction-option decision matrix

| Situation | CETAS | Copy Activity | Databricks Sqldw connector | Direct SQL/JDBC |
|---|---|---|---|---|
| Large compatible full-table/view export | **Preferred** | Acceptable after performance testing | Technically strong; controlled optional route | Avoid as default |
| Many independent objects | **Preferred with governed scheduler** | Use selectively; count partitions as source queries | Requires Databricks orchestration, staging cleanup and aggregate cap | Avoid estate-wide fan-out |
| Values contain CETAS-to-Parquet unsafe characters | Do not use unchanged | **Preferred fallback after compatibility test** | Do not assume it avoids the issue; prove the complete staged read | Possible exception, with explicit schema and load controls |
| LOB value can exceed CETAS 1 MB limit | Do not use unchanged | **Preferred fallback** if connector test passes | Use only after boundary-value testing | Possible for bounded exceptions; test memory/fetch behaviour |
| Need managed pipeline monitoring and connector retry telemetry | Limited; custom orchestration required | **Strong fit** | Databricks job telemetry; staging lifecycle remains custom | Custom implementation required |
| Need a bulk path controlled from Databricks | Separate source and Databricks phases | Possible through pipeline invocation | **Strong technical fit after proof of concept** | Weak for large volumes |
| Source cannot grant CETAS external-object permissions | Not suitable under current security decision | **Use Copy Activity** with approved identity | Requires its own Synapse and storage permissions | Possible if SQL `SELECT` permission and network path are approved |
| Small lookup/reference table | Suitable but may be operationally heavy | Suitable | Operationally heavy | **Reasonable exception** |
| No deterministic JDBC partition column | CETAS preferred | Copy without excessive partitioning | Bulk staging avoids a client-defined JDBC partition scheme | Avoid parallel JDBC; single connection may be slow |
| Strong restartability from immutable files required | **Strong fit when attempt paths are governed** | **Strong fit when attempt paths are governed** | Temporary files are not a release manifest; add governed attempt state | Weak unless it first writes an audited immutable staging copy |
| Unity Catalog-governed temporary storage is mandatory | Supported through governed destination design | Supported through governed sink design | Poor fit: UC external locations are not supported as `tempDir` | Not applicable unless separately staged |
| Need one common source generation across tables | Requires fence | Requires same fence | Requires same fence | Requires same fence; JDBC does not solve consistency |

## 4. CETAS: advantages, disadvantages and decision rules

### Advantages

- Executes a T-SQL `SELECT` and exports the result to external storage in parallel.
- Uses Synapse's distributed engine instead of pulling every row through one client process.
- Well suited to repeatable bulk extraction from materialised tables or simple presentation views.
- Produces Parquet directly for efficient Databricks ingestion.
- Supports an explicit projection, allowing approved casts, column renaming and deterministic ordering.
- Produces source request identifiers that can be correlated with Synapse DMVs and orchestration records.
- Keeps the Databricks load phase separate from the source extraction phase, improving restartability.

### Disadvantages and constraints

- CETAS creates external-table metadata in Synapse and files in ADLS; both lifecycles must be managed.
- It requires `ALTER SCHEMA`/`CREATE TABLE` plus source `SELECT`; the login also needs `ADMINISTER BULK OPERATIONS`, `ALTER ANY EXTERNAL DATA SOURCE` and `ALTER ANY EXTERNAL FILE FORMAT`. `ALTER ANY EXTERNAL DATA SOURCE` is highly privileged because it can expose database-scoped credentials.
- LOB values larger than **1 MB** are not supported.
- For Parquet/ORC export, values containing pipe, quotation-mark or carriage-return/newline characters can cause errors or rejected records. Profiling only declared data types is insufficient; actual content must be assessed.
- The output file count is engine-controlled. It must not be assumed to be one file, a fixed file count, or a particular size.
- `ORDER BY` does not provide an ordered CETAS output.
- CETAS always creates a non-partitioned external table even when the source is partitioned; application-level chunking requires a separately governed design.
- A failure or cancellation can leave partial files. Microsoft documents only a one-time cleanup attempt, so orphaned data must be detected and retained/removed by state-aware cleanup.
- External data is not covered by the source database backup; the customer owns consistency between Synapse metadata and storage.
- A new export must use a unique empty path. Reusing a previous or partially populated path risks mixing attempts.

### Use CETAS when

- the source projection and actual values pass compatibility profiling;
- the object is medium or large and benefits from source-native parallel export;
- the security owner accepts the required CETAS permissions and managed-identity/storage model;
- an immutable attempt-specific ADLS path is available;
- the run is protected by the source-generation fence; and
- operations can monitor request status, files, counts and metadata cleanup.

### Do not use CETAS unchanged when

- a LOB can exceed 1 MB;
- the exported Parquet data can contain the documented unsafe characters and no approved safe conversion exists;
- the required output type is unsupported;
- the source projection requires unsafe unrestricted SQL stored in metadata;
- the security team rejects the required external-data-source/file-format privileges;
- the destination path is not guaranteed to be new and empty; or
- the source generation can change during extraction.

An approved cast may make an object CETAS-compatible, but a cast is a data contract change. It requires precision, truncation, null, encoding and downstream-consumer tests; it must not be introduced silently merely to make CETAS succeed.

## 5. Copy Activity: advantages, disadvantages and decision rules

### Advantages

- Provides a managed orchestration activity with pipeline/activity run IDs, metrics, retries, timeouts and monitoring.
- Supports Azure and self-hosted integration runtimes and can accommodate private-network connectivity patterns.
- Can write Parquet to ADLS Gen2 without requiring the CETAS external-table DDL path.
- Supports source partition options and controlled parallel reads for large objects.
- Is the primary fallback for CETAS-incompatible values/types when an end-to-end connector test proves no truncation, skipped rows or semantic change.
- Separates route-specific settings—DIUs, `parallelCopies`, partition policy and integration runtime—from the table's logical schema contract.

### Disadvantages and constraints

- Adds another capacity and failure domain: integration runtime, connector version, activity queues and pipeline limits.
- DIUs and `parallelCopies` are different controls. More DIUs do not automatically mean more source-query parallelism, and more parallel copies can overload Synapse.
- DIUs do not apply to self-hosted integration runtime; node CPU, memory, network, concurrent-job limit and node count become operational concerns.
- A partitioned Copy operation creates multiple source queries. One activity must not be counted as one unit of Synapse pressure.
- Poor partition keys create skew: one partition runs much longer while others finish, extending the fence duration.
- Too many partitions can cause Synapse queueing or throttling. Microsoft's connector guidance specifically warns against excessive degree of copy parallelism.
- Connector schema inference, automatic mapping or fault-tolerance settings can rename, coerce, truncate or skip data unless explicitly disabled/governed.
- Retry settings can duplicate or mix output if the same sink path is reused.
- Copy can be more expensive and harder to tune because source capacity, integration-runtime capacity and ADLS throughput all interact.

### Use Copy Activity when

- CETAS is rejected because of the 1 MB LOB limit, unsafe content, unsupported representation or unacceptable CETAS privileges;
- the organisation needs managed pipeline monitoring and approved integration-runtime connectivity;
- the exact ordered projection and explicit sink schema are enforced;
- the object has a tested partition key or is small enough to copy without partition fan-out;
- no row skipping, truncation or schema drift is permitted; and
- the activity writes to a new immutable attempt path and passes the same reconciliation gates as CETAS.

### Do not use Copy Activity when

- it is being selected automatically after any CETAS failure without re-assessing compatibility and semantics;
- the connector will infer the schema or apply fault tolerance that skips bad rows;
- partition ranges are overlapping, incomplete, mutable or highly skewed;
- the aggregate number of Copy partitions plus CETAS, Sqldw, JDBC and operational queries exceeds the tested source budget;
- the integration runtime lacks proven bandwidth, high availability or operational ownership; or
- the sink path can contain files from an earlier attempt.

## 6. Azure Databricks Synapse/Sqldw connector

The Azure Databricks Synapse connector is distinct from an ordinary JDBC download. Databricks uses the Synapse SQL endpoint for control and query submission while the bulk transfer uses temporary ADLS Gen2 staging. Current syntax uses `format("sqldw")` on Databricks Runtime 11.3 LTS and above; `format("com.databricks.spark.sqldw")` is the older form.

### What it offers

- Can read a Synapse table through `dbTable` or a view/query through `query`.
- Uses a storage-staged bulk path instead of transporting the complete result set through one JDBC client stream.
- Lets a Databricks workflow control extraction, validation and Delta writing in one orchestration environment.
- Avoids the need for a client-selected JDBC `partitionColumn` merely to obtain parallel bulk movement.
- Supports dedicated SQL pools; it is not a connector for every Synapse component.

### Material limitations and risks

- The Databricks documentation is archived and explicitly says the legacy configurations might not be updated and are not officially endorsed or tested. A new enterprise implementation must not assume long-term strategic support.
- A dedicated `abfss://...` `tempDir` is mandatory. A Unity Catalog external location cannot be used directly as that `tempDir`.
- Both Databricks and Synapse need authorised access to the temporary storage. The design therefore has three network/security paths: Databricks-to-Synapse, Databricks-to-ADLS and Synapse-to-ADLS.
- The connector does not delete its temporary files automatically. Databricks recommends periodic cleanup, so cleanup must be fenced against active jobs and governed by recorded attempt state.
- Temporary connector output is not automatically a trusted release manifest. Publication still requires attempt identity, source/target counts, schema checks and Delta commit evidence.
- It does not provide a transactionally consistent snapshot across 600 independent view queries. The same source-generation fence remains mandatory.
- Each concurrent connector read still creates work in the dedicated pool and competes with CETAS, Copy Activity, direct JDBC and business queries.
- Query pushdown is limited. The archived documentation states that string, date and timestamp expressions are not pushed down by the connector; explicit source queries should be used where source-side evaluation is required.
- Authentication may involve managed identity, service principal or forwarded storage credentials. Forwarding an account key carries additional risk and requires encrypted JDBC; managed identity or approved service-principal authentication is preferred.
- Runtime, access-mode and network compatibility must be proven on the exact Databricks compute configuration. Do not infer compatibility from a notebook syntax example.

### Use the Sqldw connector when

- Databricks-controlled bulk extraction provides a clear operational benefit;
- a proof of concept passes on the selected Databricks Runtime and compute/access mode;
- Security approves the Synapse and temporary-storage identities and all three network paths;
- a dedicated, non-UC `tempDir` and state-aware cleanup process are approved;
- the view's full-volume schema and boundary values pass end-to-end testing;
- the scheduler counts each active connector source operation within the aggregate Synapse budget; and
- the object uses the same source fence, reconciliation and publication controls as every other route.

### Do not use the Sqldw connector when

- the organisation requires a currently strategic, fully supported default route for all 600 objects;
- Unity Catalog external locations are mandatory for every storage path;
- temporary storage cannot be isolated, audited and cleaned safely;
- the implementation forwards storage account keys without explicit security approval;
- the selected Databricks compute/network model has not passed an end-to-end test; or
- it is assumed to solve cross-view consistency, retries or source-capacity limits automatically.

### Decision

The Sqldw connector is technically stronger for bulk movement than ordinary JDBC, but it should be a **controlled optional route**, not the estate-wide default. Before production approval, run a representative proof of concept covering large, small, wide, LOB, temporal and special-character objects; authentication; private networking; failures; retries; temporary-file cleanup; and measured Synapse impact.

## 7. Direct SQL/JDBC download from Synapse

“Downloading using SQL DB” is interpreted here as Databricks or another client executing SQL over JDBC/ODBC against the Synapse dedicated-pool endpoint and receiving the rows directly.

### What it offers

- Simple implementation for a small table or diagnostic query.
- Requires normal query permissions rather than CETAS external-object permissions.
- Can push projection and filters into Synapse.
- Spark JDBC can parallelise a read by using `partitionColumn`, bounds and `numPartitions`.

### Why it is not the recommended bulk route

- Without partition options, a Spark JDBC read commonly has one partition/connection and becomes a throughput bottleneck.
- With partition options, `numPartitions` also determines the maximum concurrent JDBC connections. Those connections create parallel Synapse queries and consume the same source concurrency budget.
- The JDBC partition column must be numeric, date or timestamp and should distribute data evenly. Bounds determine partition stride; they do not filter the source data.
- Multiple JDBC partition queries do not create a cross-query snapshot. A source fence or immutable generation remains mandatory.
- Rows travel through the client/Spark network path rather than being exported directly by Synapse to storage. Long reads are exposed to connection resets, query timeout, fetch-size and executor failure.
- A failed direct read does not automatically leave a durable, independently auditable source extract. If the job writes straight to Delta, source extraction and target commit become harder to separate and retry safely.
- Schema/type inference can produce different decimal, timestamp, binary or string behaviour from the approved Parquet contract.
- Large result sets can overrun driver/executor memory if the implementation accidentally collects data or uses insufficient parallelism.
- Direct connectivity requires firewall/private endpoint routing, DNS, TLS, token/secret handling and driver lifecycle management from Databricks compute to the Synapse SQL endpoint.

### Appropriate uses

- Small reference tables with a bounded size and independent publication contract.
- Schema discovery, count queries, health checks and controlled diagnostics.
- A temporary exception when CETAS and Copy Activity are unavailable, provided the exception has explicit size, duration, concurrency and retry limits.

### Inappropriate uses

- The default route for the complete full-load inventory.
- Large tables without a stable, evenly distributed partition column.
- High `numPartitions` values chosen merely to increase speed.
- Any design that performs `collect()`, pandas conversion or driver-side file generation for large results.
- Any direct-to-final-Delta path that bypasses immutable attempt identity, count reconciliation and publication isolation.

### Decision

Direct JDBC is technically possible, but it does not remove the major challenges. It moves parallelism and failure handling from Synapse/ADF into Spark, adds database connections and still requires source consistency, schema governance, capacity limits and idempotent publication. Treat it as an exception route, not a peer bulk route.

## 8. ADLS Gen2 write challenges

| Challenge | Executive impact | Required control |
|---|---|---|
| RBAC and hierarchical namespace ACLs | Identity appears authorised but cannot traverse or write the path | Grant least-privileged role plus required directory traversal/write ACLs; test with the actual runtime identity |
| Firewall, private endpoint and DNS | Intermittent or complete inability to write/read staging | Prove Synapse-to-ADLS and Databricks-to-ADLS routes, DNS and endpoint approvals before performance testing |
| Wrong endpoint/protocol or external data-source configuration | CETAS connection failures or data written to the wrong place | Govern the exact `abfs[s]`/HTTPS endpoint, container and prefix as deployment configuration |
| Reused/non-empty output path | Files from different attempts or generations are mixed | Allocate a new immutable leaf path for every mapping attempt; reject a non-empty path |
| Partial/orphan files after failure | A folder exists but is not a valid completed extract | Trust the audited request/activity and validation state, not folder existence; quarantine and clean by recorded disposition |
| Unknown file count and size | Downstream jobs may suffer from excessive files, skew or unexpected large files | Measure file count/bytes per object; never assume a fixed CETAS file count |
| Folder/file scale | PolyBase queries can fail or experience JVM memory pressure | Keep prefixes narrow; Microsoft recommends no more than 30,000 files per HDFS folder and documents a 33,000 maximum under the stated high-concurrency condition |
| ADLS throttling/HTTP 500 or 503 responses | Longer extraction, retries and missed delivery window | Monitor throttling, cap aggregate concurrency, retry with exponential backoff and preserve fence margin |
| Cross-region traffic | Higher latency, cost and another failure boundary | Co-locate platforms where possible; approve measured cross-region exceptions |
| Unmasked sensitive data | Staging can expose data that consumer-table policies would hide | Deny business access; use separate write/read/cleanup identities, encryption, audit and classification |
| Schema and encoding drift | Databricks cannot decode or interprets values differently | Use an explicit ordered projection and Spark schema; test text, decimal, date/time and special characters |
| Premature lifecycle deletion | Active attempts or rollback evidence can be lost | Use audit-state-aware cleanup; lifecycle policy is only a conservative backstop |
| Soft delete and retention | Storage remains billed after logical deletion | Include soft-delete retention and failed attempts in the cost model |
| Credential/identity rotation | Previously working exports fail unexpectedly | Use managed identity where approved; monitor credential, role, ACL and endpoint changes |
| Concurrent cleanup and reads | Loader sees missing files or inconsistent attempt contents | Cleanup only terminal, expired attempts under a cleanup lease and validated path guard |
| Sqldw temporary-file accumulation | Storage cost grows and abandoned extracts become difficult to distinguish from live work | Use a dedicated connector container/prefix, recorded attempt ownership and state-aware cleanup after a conservative age threshold |

## 9. Parallelism and Synapse-capacity limitations

### Capacity rules

1. **One shared budget:** count every active CETAS statement, Copy source partition, Sqldw connector source operation, JDBC partition and operational query.
2. **Ceiling is not target:** do not configure 32 or 48 merely because documentation lists those maxima.
3. **Resource grant trade-off:** lower grants allow more concurrency but may slow large scans/exports and extend the source fence.
4. **Queueing matters:** a submitted query is not necessarily running. Measure queue duration separately from execution time.
5. **Largest objects dominate:** schedule large objects early and use smaller objects to fill tested spare capacity.
6. **Copy Activity is multi-dimensional:** total source pressure is approximately concurrent activities multiplied by effective source partitions, plus CETAS/Sqldw/JDBC/other queries.
7. **Retry reserve:** leave capacity for bounded retries and mandatory objects; do not saturate the pool with first attempts.
8. **Protect upstream and consumers:** the extraction window must not destabilise existing Synapse transformations or permitted user workloads.

### Recommended performance ladder

For CETAS, begin with:

```text
8 -> 12 -> 16 -> 24
```

Test 32 only if 24 improves throughput without unacceptable queueing, source pressure, ADLS throttling or long-tail duration. Do not target 48 without a deliberately configured workload group, verified classifier, tested 2% grant and explicit acceptance that individual queries may slow.

For Copy Activity, begin conservatively and change only one dimension at a time:

```text
Concurrent activities:       2 -> 4 -> 8 -> 12
parallelCopies per activity: 1 -> 2 -> 4
```

Stop increasing when aggregate throughput flattens, p95 duration worsens, Synapse queueing rises, the integration runtime saturates, ADLS throttles or source-fence margin becomes unsafe.

For the Sqldw connector, start with a separate, lower-concurrency proof-of-concept lane:

```text
Concurrent connector reads: 2 -> 4 -> 8
```

Increase only after confirming the actual Synapse request pattern, queue time, staging-file behaviour, Databricks utilisation and cleanup safety. Do not run the CETAS, Copy and Sqldw ladders independently at their individual maxima; the aggregate governor is authoritative.

### Metrics executives should expect

- Total compressed GB exported per minute.
- Queue time and execution time by route and size category.
- p50/p95 completion time and the last mandatory-object completion time.
- Concurrent Synapse requests and workload-group/resource-class assignment.
- Copy Activity effective DIUs, `parallelCopies`, partitions and integration-runtime saturation.
- Sqldw connector reads, Synapse requests, temporary bytes/files, staging age and cleanup backlog.
- JDBC connections/partitions when an exception route is used.
- ADLS file count, bytes, throttling and failed writes.
- Retry volume and additional fence time.
- Impact on upstream materialisation and permitted Synapse consumers.

## 10. Cross-cutting implementation challenges

| Challenge | What can go wrong | Executive decision/control |
|---|---|---|
| Source-generation fence | A new upstream refresh overlaps extraction, producing mixed generations | Make every writer participate in the interlock or use immutable generation-specific source tables |
| Consistency-group assignment | One failed table blocks a very large business group | Business and architecture owners approve grouping and availability blast radius |
| Metadata quality | Wrong source/target, schema, route or policy is applied at scale | Versioned mapping and contracts; fail closed on incomplete approval |
| Type compatibility | Timestamps, `time`, decimals, LOBs or special characters change meaning or fail | Explicit temporal/type contract and representative boundary tests |
| Identity separation | One highly privileged identity can extract, publish and delete | Separate Synapse export, staging read, Databricks write, governance and cleanup identities |
| Retry/idempotency | Timeout causes duplicate files or Delta rows | New attempt path for re-extraction; stable logical Delta transaction identity for same-data retry |
| Reconciliation | Pipeline reports success despite missing, duplicated or unreadable data | Independently compare source audit, readable Parquet and committed Delta counts plus schema/business checks |
| Publication | Users see a partial multi-table release | Atomic table replace only for independent tables; release pointer for consistency groups |
| Retention/rollback | `VACUUM` or lifecycle deletes the previous usable release | Versioned retention contract, cost approval and tested rollback |
| Observability | Operators cannot distinguish slow, queued, failed, duplicated or inconsistent state | One correlated dashboard, numeric alerts and exercised runbooks |
| Delivery window | A few large or skewed objects miss the user-availability target | Size-based scheduling, measured critical path, retry reserve and early warning threshold |
| Cost | Full loads create transport, candidate, current and retained historical copies | Measure compressed bytes and write amplification; approve retention and compute budgets |

## 11. Executive risks and mitigations

| Risk | Likelihood without control | Business consequence | Primary mitigation |
|---|---|---|---|
| Mixed source generation | High | Inconsistent reports and joins | Enforced cooperative fence or immutable source generation |
| Synapse saturation | High | Upstream delay and missed reporting window | Aggregate concurrency cap and workload isolation |
| CETAS-incompatible object discovered late | Medium–High | Object omitted or bridge delayed | Early content profiling and approved Copy fallback |
| ADLS network/permission failure | Medium | Complete extraction stoppage | Pre-production route/identity tests and monitoring |
| Partial files treated as success | Medium | Missing or corrupt Delta target | Attempt manifest and independent validation |
| Blind retry after timeout | Medium | Duplicate data or inconsistent state | History/state reconciliation before retry |
| Large consistency-group failure | Medium | Previous release remains active for broad domain | Blast-radius approval and group-level readiness |
| Retention misconfiguration | Medium | Rollback unavailable | Explicit table-property contract and post-publication verification |
| Direct JDBC over-parallelisation | Medium | Synapse queueing/throttling and unstable extraction | Exception-only use and capped `numPartitions` |
| Sqldw connector support or runtime incompatibility | Medium | Planned bulk route fails after build has started | Treat as optional; prove exact runtime/network/authentication configuration before assignment |
| Sqldw temporary-data leakage or accumulation | Medium | Security exposure and unplanned storage cost | Dedicated restricted staging area, audit, retention and state-aware cleanup |
| Sensitive staging exposure | Medium | Security/compliance incident | Least privilege, private connectivity, audit and cleanup |

## 12. Recommended implementation position

1. **Approve CETAS as the primary route**, subject to object-level compatibility evidence.
2. **Approve Copy Activity as the governed fallback**, with no implicit route switching.
3. **Approve the Sqldw connector only as a controlled optional route**, conditional on exact-runtime proof, explicit security approval, dedicated temporary storage, cleanup controls and measured capacity impact. Do not assign all 600 objects to it by default.
4. **Do not approve direct JDBC as the estate-wide route.** Allow it only through a documented exception with object size, concurrency, timeout, schema and retry limits.
5. **Approve a conservative initial concurrency level**, then increase only from measured full-volume tests.
6. **Require one aggregate Synapse source-read governor** across CETAS, Copy Activity, Sqldw, direct JDBC and operational workloads.
7. **Require immutable ADLS attempt paths** and prohibit retries into an existing path.
8. **Require independent source/staging/Delta reconciliation** before publication.
9. **Keep the previous approved release available** until rollback retention expires.
10. **Treat missing telemetry as indeterminate**, not as proof that no error occurred.
11. **Do not commit to the user-ready time until a production-volume rehearsal proves the p95 critical path with retry margin.**

## 13. Go-live questions for executives

- Has Security approved the CETAS privilege model and the Copy, Sqldw and JDBC identities?
- Has every object been assigned an approved extraction route based on actual data profiling?
- If Sqldw is assigned, has the exact Databricks Runtime, compute/access mode, private network, authentication and `tempDir` design passed a full-volume proof of concept?
- What tested aggregate Synapse concurrency cap protects upstream and user workloads?
- What is the measured p95 critical path, including queueing and one bounded retry?
- Which consistency groups can remain on yesterday's release if one mandatory object fails?
- Has the organisation accepted the availability impact of the largest group?
- Are ADLS private connectivity, DNS, RBAC and ACL tests complete for every runtime identity?
- Are staging and retained Delta storage costs approved?
- Can operations identify and recover an ambiguous CETAS, Copy, Sqldw, JDBC or Delta outcome without duplicating data?
- Has rollback been tested after the configured retention and automated optimization settings are applied?

## Official references

- [Microsoft: CREATE EXTERNAL TABLE AS SELECT (CETAS)](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-table-as-select-transact-sql)
- [Microsoft: Dedicated SQL pool memory and concurrency limits](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/memory-concurrency-limits)
- [Microsoft: Copy and transform data in Azure Synapse Analytics](https://learn.microsoft.com/en-us/azure/data-factory/connector-azure-sql-data-warehouse)
- [Microsoft: Copy Activity performance optimization features](https://learn.microsoft.com/en-us/azure/data-factory/copy-activity-performance-features)
- [Azure Databricks archive: Query data in Azure Synapse Analytics using the Synapse/Sqldw connector](https://learn.microsoft.com/en-us/azure/databricks/archive/connectors/synapse-analytics)
- [Apache Spark: JDBC data source options](https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html)
- [Microsoft: Create ADLS Gen2 storage with hierarchical namespace](https://learn.microsoft.com/en-us/azure/storage/blobs/create-data-lake-storage-account)
- [Microsoft: Azure Storage lifecycle management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
- [Microsoft: ADLS Gen2 access control model](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-access-control-model)
- [Microsoft: Azure Storage scalability and performance targets](https://learn.microsoft.com/en-us/azure/storage/common/scalability-targets-standard-account)
