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
