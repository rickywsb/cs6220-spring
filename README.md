CREATE GTT (with GLOBAL) ─────┐
                             │
     Create template table   ▼
         + register in catalog: pg_global_temp_tables
                             │
         ┌───────────────────┴──────────────────────────────┐
         │                                                  │
First access (SELECT/INSERT/...)                       SET pgtt.enabled = off
         │                                                  │
Create session-local temp table in pg_temp schema       Access template directly
         │
Reroute all future accesses to this table


1. Goal
The purpose of this investigation is to assess whether the pgtt extension should be included in the list of supported extensions for Amazon RDS PostgreSQL. We begin by evaluating whether similar global temporary table (GTT) functionality already exists in standard RDS PostgreSQL. Since PostgreSQL does not natively support global temporary tables that persist across sessions while maintaining per-session data isolation, we investigate whether incorporating pgtt—which emulates this behavior—is beneficial for RDS environments where typical TEMP tables are limited to session scope and must be redefined each time.

2. Background and Functional Comparison
The pgtt extension introduces an emulation layer for global temporary tables, which are absent in core PostgreSQL. While PostgreSQL supports session-level TEMP tables, these must be redefined in every session and are invisible across sessions. By contrast, pgtt allows users to define a "global" temporary table once, and then instantiate a private, session-local temp table from that definition transparently. This model reduces DDL repetition and enables consistent table definitions across multiple sessions or clients.

Internally, when pgtt.enabled is set to true and the extension is loaded (either via session_preload_libraries, ALTER DATABASE SET, or session-local LOAD), the extension intercepts any CREATE TEMPORARY TABLE command containing the keyword GLOBAL or comment /*GLOBAL*/. Instead of creating a true temporary table, it creates an unlogged "template" table in the pgtt_schema namespace and records its metadata in a catalog table called pg_global_temp_tables. On first access to the table (such as through a SELECT or INSERT), the extension automatically creates a regular session-local temporary table in the pg_temp schema based on the template. All subsequent accesses are transparently rerouted to this new temporary table.

For example, after defining a GTT like:

CREATE /*GLOBAL*/ TEMPORARY TABLE test_tt (id int, lbl text) ON COMMIT PRESERVE ROWS;
A session can access it via:

SELECT * FROM test_tt;
which is automatically rerouted to the session’s private instance in pg_temp.test_tt, even if the user queries pgtt_schema.test_tt.

This behavior provides a convenient abstraction layer that aligns with Oracle-style GTT usage. Compared to traditional RDS PostgreSQL, where temporary table definitions must be repeated in each session or managed externally, pgtt offers a centralized schema and lifecycle management system for session-isolated temp data. 



Example: Transparent Rerouting

LOAD 'pgtt';

CREATE /*GLOBAL*/ TEMPORARY TABLE test_tt (id int, lbl text) ON COMMIT PRESERVE ROWS;

INSERT INTO test_tt VALUES (1, 'one'), (2, 'two'), (3, 'three');

-- All three variants below return the same data due to rerouting
SELECT * FROM test_tt;
SELECT * FROM pg_temp.test_tt;
SELECT * FROM pgtt_schema.test_tt;
To bypass rerouting and inspect the template table directly:

SET pgtt.enabled TO off;
SELECT * FROM pgtt_schema.test_tt;  -- returns 0 rows
SET pgtt.enabled TO on;
Design Details

Table creation: The extension registers GTTs in pgtt_schema.pg_global_temp_tables with metadata including schema, name, and ON COMMIT behavior.
Renaming / Dropping: Both operations are blocked if the temporary table has been instantiated in the session. Operations succeed only when inactive.
Schema relocation: By default, pgtt tables are not relocatable, but advanced configuration allows it for use cases requiring fixed schema names.
pg_dump / restore: Template tables and catalog metadata are included in dumps, allowing full restoration of the GTT setup.
This model allows applications to use global temporary tables in PostgreSQL with consistent session-local isolation, while maintaining persistence and visibility for administrative tasks.


2. Background and Functional Comparison
A Global Temporary Table (GTT) is a database object that allows all sessions to share the same table definition (schema), while maintaining session-local visibility of the table’s data. This means multiple users or connections can access the same table name and structure, but each sees only their own data. Unlike regular temporary tables in PostgreSQL—which must be explicitly created in every session and discarded afterward—a GTT is defined once and reused across sessions, reducing overhead and ensuring schema consistency. This model is widely supported in Oracle and other enterprise databases, and is especially useful in OLTP systems where temporary data is processed per user but must follow a shared structure (e.g., staging data, session caches, or ETL pipelines).

PostgreSQL does not natively support global temporary tables. Its temporary tables are entirely session-scoped, meaning both the definition and data exist only for the lifetime of a session. This limitation can introduce overhead in applications that require repeated creation of identical temporary tables, or in systems migrating from Oracle where GTTs are extensively used.

The pgtt extension fills this gap by emulating global temporary table behavior within PostgreSQL’s extension framework. It separates the definition of the table from the data, storing the former as a persistent "template" table and dynamically creating session-local temporary instances at runtime. This allows applications to interact with GTTs as if they were natively supported.


The extension supports common table features such as LIKE clauses and AS SELECT ... WITH DATA. These allow the user to initialize a GTT with structure or data during creation, with the data applying only to the current session. However, pgtt does not support ON COMMIT DROP, which is available in native PostgreSQL temp tables. Instead, it strictly supports ON COMMIT DELETE ROWS and ON COMMIT PRESERVE ROWS, mirroring the Oracle behavior where the table persists and only the rows are conditionally cleared.

Importantly, the use of pgtt also reduces catalog bloat. In systems that create and drop large numbers of temporary tables, repeated DDL operations can clutter PostgreSQL’s internal catalogs, degrading performance over time.


Users can create indexes and constraints on GTTs, with the exception of foreign key constraints. Attempting to declare referential integrity relationships on GTTs will result in an error. This restriction is intentional and designed to mimic enterprise RDBMS systems like Oracle and DB2, where temporary tables are isolated and do not enforce foreign keys to permanent tables.



Summary of What This Tells Us
Loading the extension has almost no measurable performance cost, which makes it viable to preload in environments where it may be used conditionally.
Runtime performance of pgtt is comparable to native PostgreSQL temporary tables, even though pgtt uses rerouting and internal logic. This validates its efficiency and makes it suitable for production environments where Oracle-style GTT semantics are required.
The rerouting mechanism does not add significant latency, and in fact performs slightly better in high-connection scenarios due to the absence of repeated DDL.


Security Analysis
While the pgtt extension provides powerful functionality for emulating global temporary tables, it introduces several security and operational concerns, particularly in managed environments such as Amazon RDS where user privilege separation and auditability are critical. The following concerns should be considered before adopting the extension:

1. Requires session_preload_libraries, which depends on superuser privileges

In order for the extension to intercept temporary table creation and enable rerouting, pgtt must be loaded at session startup using session_preload_libraries. However, in RDS PostgreSQL, only the rds_superuser role has permission to modify this setting via the DB parameter group. This means that:

Regular users cannot dynamically enable pgtt in their sessions.
The extension must be preloaded globally across all sessions, even those not using GTTs, slightly increasing the attack surface.
In multi-tenant or hosted environments, globally preloading extensions that alter query execution logic is generally discouraged due to the potential for unexpected cross-session behavior and reduced control granularity.

2. Data is stored in unlogged tables — no crash recovery

The underlying “template” table used by pgtt is an unlogged table, and the dynamically created session-local temp tables are standard PostgreSQL temporary tables, which are also unlogged. This means:

If the server crashes unexpectedly, any in-flight data stored in GTTs is lost.
While this may be acceptable for typical temporary workloads, it violates durability guarantees expected in PCI/GDPR-compliant environments.
Users must be clearly informed that data in GTTs is never recoverable after crash.
Although this behavior aligns with native PostgreSQL TEMP tables, the presence of persistent schema-level definitions in pgtt may mislead users into assuming partial durability.

3. Query rerouting can interfere with logical replication or auditing

pgtt works by rewriting queries after parsing, redirecting them from the pgtt_schema table to the corresponding pg_temp version. This rerouting is internal and not visible in the final query plan or audit logs. Consequently:

Logical decoding tools like AWS DMS or Debezium may misinterpret the query, as the table OID referenced does not match what is seen in the WAL stream.
Auditing and query logging systems might log the original query (SELECT * FROM pgtt_schema.tt) rather than what was actually executed (SELECT * FROM pg_temp.tt), reducing traceability.
This undermines the predictability and transparency required in regulated environments.



### Preliminary Performance Assessment

The following performance observations are based on benchmark results published in the official `pgtt` documentation. While they offer useful insight into expected behavior, these results should be independently validated in production-like environments before making adoption decisions.

#### 1. Minimal Overhead When Loaded But Not Used

Benchmark tests using `pgbench` in a TPC-B–like scenario indicate that simply loading the `pgtt` extension does not introduce meaningful overhead if it is not actively used.

- **Without extension**: 862 TPS  
- **With extension loaded**: 852 TPS

The performance difference is approximately 1.1%, suggesting that preloading `pgtt` via `session_preload_libraries` introduces negligible cost in typical workloads where GTTs are not invoked. This supports its safe inclusion in shared environments, assuming proper access controls are in place.

#### 2. Comparable Performance to Native TEMP Tables

The benchmark also compares runtime performance between PostgreSQL native temporary tables and `pgtt`-based Global Temporary Tables:

- **Using native TEMP tables**: 285 TPS  
- **Using `pgtt` GTTs**: 292 TPS

Despite its use of rerouting and dynamic temporary table creation, `pgtt` slightly outperforms native TEMP tables in this scenario. This marginal gain (~2.3%) may be due to reduced DDL overhead, since `pgtt` defines each table only once and instantiates the session-local copy on first access. Regular TEMP tables, by contrast, may incur more catalog and locking overhead when repeatedly created and dropped.

#### Summary of What This Tells Us

While the `pgtt` extension introduces two mechanisms that could theoretically impact performance—session-level library loading and runtime query rerouting—the benchmark results suggest that these mechanisms have little to no negative effect in practice. In fact, under certain conditions, `pgtt` may even slightly improve throughput due to reduced catalog churn and session-level reuse of temporary structures.

- Loading the extension has almost no measurable performance cost, making it viable to preload in environments where it may be used conditionally.
- Runtime performance of `pgtt` is comparable to, and sometimes slightly better than, native PostgreSQL temporary tables.
- The rerouting mechanism does not introduce significant latency and may provide optimization benefits in high-concurrency environments.

> _Note: These results are drawn from the `pgtt` extension’s own performance documentation. While promising, they should be considered preliminary. Independent benchmarking under representative production conditions is still recommended to validate these results and assess performance impact in real-world workloads._


Following the security analysis, it is equally important to evaluate the potential performance impact of introducing the pgtt extension. In managed environments like Amazon RDS, where session_preload_libraries and query rerouting mechanisms may introduce latency or affect scalability, understanding runtime behavior is critical. This section examines whether these mechanisms cause measurable overhead during execution, and whether pgtt performs comparably to native PostgreSQL temporary tables under typical transactional workloads. The goal is to provide data-driven insight into whether the extension is suitable for performance-sensitive deployments.





pg_global_temp_tables achieves Oracle-style GTT behavior by defining a view over a table-returning function and attaching INSTEAD OF triggers to simulate DML operations. The key idea is that the view remains accessible under a schema-qualified name, while the actual data is stored in a native TEMP table created dynamically per session. This allows application SQL (e.g., SELECT * FROM schema.temp_table) to work without modification—similar to Oracle—while isolating session data using standard PostgreSQL mechanisms. However, this architecture introduces more moving parts and depends heavily on trigger logic.

Despite its cleverness, pg_global_temp_tables has only 18 stars, 4 watchers, and 5 forks on GitHub. Its last update was over a year ago, and the project does not appear to be actively maintained. While it solves a real problem, its reliance on function-view-trigger plumbing makes it harder to manage, and less transparent at runtime than pgtt.


| Feature / Criteria                          | `pgtt` (pg_temp_template)               | `pg_global_temp_tables`                           | Native `pg_temp` Tables              |
|--------------------------------------------|----------------------------------------|--------------------------------------------------|--------------------------------------|
| **Global Table Definition**                | ✅ Stored as persistent template table  | ✅ Achieved via view/function/trigger            | ❌ Redefined per session             |
| **Session-local Data Isolation**           | ✅ Yes (via rerouted temp instance)     | ✅ Yes (via session-created temp table)          | ✅ Yes                               |
| **Requires `session_preload_libraries`**   | ✅ Yes                                  | ❌ No                                            | ❌ No                                |
| **Compatible with Oracle-style GTTs**      | ✅ Closely mimics syntax & semantics    | ✅ Query syntax compatible with Oracle           | ❌ No                                |
| **Query Rerouting Mechanism**              | ✅ Internal rerouting at access         | ❌ View + triggers manage session scoping        | ❌ Not applicable                    |
| **Performance Overhead**                   | ⬤ Minimal (~1%)                        | ⬤ Moderate (trigger overhead)                   | ✅ Very low                          |
| **Crash Recovery**                         | ❌ No (uses unlogged template tables)   | ❌ No (temp tables not crash-safe)               | ❌ No                                |
| **Logical Replication Compatibility**      | ⚠️ Potential issues with rerouting      | ✅ Safe (no parser hooks)                        | ✅ Fully compatible                  |
| **Foreign Key Support**                    | ❌ Not supported                        | ❌ Not supported                                 | ✅ Supported                         |
| **DDL Simplicity**                         | ✅ CREATE /*GLOBAL*/ TEMP TABLE         | ❌ Requires multiple steps and functions         | ✅ Standard SQL                      |
| **Schema-qualified Access**                | ✅ Always enabled via search_path       | ✅ Fully supported via view layer                | ❌ No (`pg_temp` not schema-qualified) |
| **RDS Compatibility**                      | ⚠️ Limited (needs preload setting)      | ✅ Works without superuser                       | ✅ Fully supported                   |
| **Maintenance Activity**                   | ✅ Actively maintained (v4.2, 2025)      | ❌ Inactive (18 stars, no recent commits)        | ✅ Core PostgreSQL                   |





1. Choose pgtt

Pros:

Offers robust emulation of Oracle-style GTTs with true schema-qualified access and session-local data.
Simplifies migrations by preserving application-level query syntax.
Actively maintained with recent performance, documentation, and compatibility fixes (v4.2 as of June 2025).
Cons:

Requires session_preload_libraries, limiting use in Amazon RDS unless applied by a privileged role.
Involves internal rerouting, which may cause complications in environments using logical replication or strict audit logging.
Adds system complexity due to its reliance on catalog metadata and internal hooks.
Not suitable in multi-tenant shared environments without strong access controls.
Recommended when:

You are migrating from Oracle and need compatibility with CREATE GLOBAL TEMPORARY TABLE semantics.
Your environment supports session_preload_libraries (e.g., self-managed PostgreSQL or RDS with admin control).
You need centralized GTT definitions with automatic per-session isolation.
2. Defer pgtt Adoption

Pros:

Reduces operational and security complexity, especially in managed cloud environments.
Maintains full compatibility with PostgreSQL’s logical replication and audit tools.
Leverages well-understood native behavior without introducing additional abstractions.
Allows future reconsideration if RDS eventually supports session preload extensions at finer granularity.
Cons:

Requires application logic to explicitly define and manage TEMP tables in each session.
Misses out on centralizing temporary table definitions for reuse.
May result in catalog bloat under heavy use of ad hoc temporary tables.



To evaluate both passive and active performance implications of the pgtt extension, a series of pgbench tests were conducted in a TPC-B–like scenario. These tests measure transactions per second (TPS) under various configurations.

Overhead of Loading the Extension (Not Used)

Without loading the extension:

number of transactions actually processed: 51741
latency average = 23.201 ms
tps = 862.165341 (excluding connections establishing)
With pgtt loaded but not used:

number of transactions actually processed: 51171
latency average = 23.461 ms
tps = 852.599010 (excluding connections establishing)
Observation: Loading the extension via session_preload_libraries introduces only a 1.1% TPS drop, indicating negligible overhead when pgtt is not actively used.

Performance: Regular TEMP Table vs. pgtt GTT

Using a regular temporary table:

number of transactions actually processed: 17153
latency average = 70.058 ms
tps = 285.514186 (excluding connections establishing)
Using a pgtt-based global temporary table:

number of transactions actually processed: 17540
latency average = 68.495 ms
tps = 292.028832 (excluding connections establishing)
Observation: When actively used, pgtt performs slightly better than native PostgreSQL TEMP tables in this scenario, with a ~2.3% TPS improvement and lower average latency. This may be due to reduced catalog activity and session reuse, as pgtt avoids repeated DDL by reusing the same template definition across transactions.
