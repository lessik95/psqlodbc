# SQL Server Linked Server LOB compatibility experiment

This branch investigates SQL Server 2025 `MSDASQL` + psqlODBC 18.00.0005 returning error 7341 for long PostgreSQL `text`/`xml`/`json` values.

## Evidence

- Direct ODBC `GetChars()` (SequentialAccess): 100,000 ASCII characters returned successfully in 25 chunks.
- COM `ADODB.Connection` using `Provider=MSDASQL`: 100,000 ASCII characters returned successfully.
- SQL Server `OPENQUERY`: 7341 for the same `repeat('X', 100000)::text` expression.
- psqlODBC MyLog on the failing Linked Server path: `SQLDescribeCol` reports `SQL_WLONGVARCHAR` (-10) with ColumnSize -4 -> 0; `SQLBindCol` uses `SQL_C_WCHAR` (-8), BufferLength=0; `SQLFetch` then warns "Fetched item was truncated"; no subsequent `SQLGetData` seen in the supplied trace.

## Hypothesis (not proven)

For result sets whose observed longest value is already known, returning a positive ColumnSize may prevent SQL Server's MSDASQL access path from making a zero-length character binding. The experimental patch changes only that metadata path for the Unicode driver when called from SQL Server. It does NOT fix a possible `MSDASQL` stream binding bug and may therefore still fail.

## SQL Server tests

Use a **separate test DSN and Linked Server** with the freshly compiled DLL, `Max LongVarChar=-4`, `Text as LongVarChar=true`, `Unknowns as LongVarChar=true`, `LFConversion=0`. Do not replace the production DSN.

```sql
-- Known-size single ASCII text: expect DATALENGTH=200000 for NVARCHAR
SELECT DATALENGTH(payload) AS bytes_received
FROM OPENQUERY([BOX-DWH-PATCH-TEST],
  'SELECT repeat(''X'', 100000)::text AS payload');

-- Mixed Unicode: assert no truncation and inspect value
SELECT DATALENGTH(payload) AS bytes_received
FROM OPENQUERY([BOX-DWH-PATCH-TEST],
  'SELECT repeat(''Ж曲'', 100000)::text AS payload');

-- Multiple rows with different lengths: avoid relying on result-set maximum only
SELECT DATALENGTH(payload) AS bytes_received
FROM OPENQUERY([BOX-DWH-PATCH-TEST],
  'SELECT repeat(''X'', n)::text AS payload
   FROM (VALUES (5), (100000), (1000000)) AS q(n)');

-- Real table: xml/json/text including >1MiB
SELECT *
FROM OPENQUERY([BOX-DWH-PATCH-TEST],
  'SELECT * FROM "SRV"."t_struct" LIMIT 100');
```

Also compare UTF-8 bytes/hashes with PostgreSQL for representative rows including Cyrillic and non-1251 characters. A successful build or successful single-row query alone does not validate a production fix.

## Expected investigative follow-up

If `SQLBindCol(BufferLength=0)` remains in the trace after this change, collect an ODBC manager trace / instrument `SQLDescribeCol`, `SQLColAttribute`, `SQLFetch`, and `SQLGetData`, then revisit the driver/bridge boundary before making more changes.


## Revision 2 (diagnostic)

The first build compiled successfully but a Linked Server test still produced error 7341.
A later psqlODBC trace for `repeat('X',100000)::text` had `adtsize_or=100000`,
`column_size=-4`, `SQLBindCol BufferLength=0`, and `SQLFetch` truncation.
The trace PID differed from the SQL Server process PID. This suggests (but does
not prove) that the driver is hosted out-of-process, where `isSqlServr()` would
be false.

Revision 2 **removes the `isSqlServr()` guard**. It activates only for Unicode
drivers explicitly configured with `MaxLongVarcharSize=-4`, and only when
a positive longest-result value is known. Look for this diagnostic marker
with `MyLog=2`:

```
SQLSERVER_LOB_EXPERIMENT_V2: type=25 longest=100000
```

Interpret results in this order:

1. **Marker present**, SQLDescribeCol size=100000, `OPENQUERY` works:
   supports the metadata hypothesis for this query; test more cases.
2. **Marker present**, SQLDescribeCol size=100000, still error 7341:
   metadata alone was insufficient; inspect SQLBindCol and stream behavior.
3. **Marker absent**, still size=-4: either the patched DLL was not loaded,
   the size was not known at the time, or another condition excluded the
   code path. Compare installed DLL with the CI artifact and inspect the log.

This build is *not* side-by-side installable under a different ODBC driver
name: the normal installer registers `PostgreSQL Unicode(x64)`. Back up DSN
configuration and uninstall the previous experiment before installing v2.


## Revision 3 (SQL Server error 7356)

The V2 build passed single-row 100,000-character text, 1,589,499-character XML,
Unicode, JSONB and a mixed-length TEXT result. However, this real query:

```sql
SELECT * FROM OPENQUERY([BOX-DWH],
  'SELECT * FROM "CFG".t_service_config_var ORDER BY fk_id;');
```

failed with SQL Server error 7356, `DBCOLUMNFLAGS_ISLONG` changing from
128 (prepare) to 0 (execution) for PostgreSQL TEXT `fc_description`.

The psqlODBC MyLog confirmed that in the prepare path no rows had been fetched,
`adtsize_or_longestlen=-3`, `ColumnSize=-4`, and `SQL_WLONGVARCHAR=-10`.
During execution `longest=31`, and the original
`pgtype_attr_to_concise_type()` code reclassified the same PostgreSQL TEXT
column as `SQL_WVARCHAR=-9` because 31 <= MaxVarcharSize=255.

V3 keeps the ODBC long type stable for TEXT and unrecognized PostgreSQL types
(including json/jsonb) when the Unicode driver uses `MaxLongVarcharSize=-4`
and the respective `TextAsLongVarchar` or `UnknownsAsLongVarchar` setting is
enabled. It also treats unconstrained `varchar` as a long type with dynamic
length for this experimental configuration. The positive observed `ColumnSize`
behavior of V2 remains intact.

**Validation required on non-production SQL Server**: rerun the real
`t_service_config_var` query with 8 columns, short TEXT values, small JSONB,
100k TEXT, >1.5M XML, Unicode, NULL/empty, mixed lengths, and unconstrained
VARCHAR. A previously loaded old DLL in a running SQL Server instance can
continue to run despite replacing the on-disk driver; do not restart production
SQL Server solely for this experiment without an approved change window.


## Revision 4 (persistent 7356 under V3)

V3 MyLog definitively shows the V3 driver is loaded (source line 1128 for
`pgtype_attr_column_size`). Both prepare and execute report
`SQL_WLONGVARCHAR=-10` for short TEXT, but prepare reports
`ColumnSize=-4` / `ColDef=0`, whereas execution reports `31` /
`ColDef=31`. Short JSONB similarly retains `SQL_LONGVARCHAR=-1` yet
changes length from unknown to `10`. This appears consistent with
MSDASQL changing its OLE DB `DBCOLUMNFLAGS_ISLONG` classification
based on size even though the ODBC SQL type is stable.

V4 is a separate metadata-only hypothesis: with Unicode driver,
`MaxLongVarcharSize=-4`, and a known positive observed length, the
driver reports at least **8192 characters** for PostgreSQL TEXT,
unconstrained VARCHAR, XML, JSON (OID 114), and JSONB (OID 3802). The
8192 minimum is chosen to exceed both OLE DB character LOB thresholds
(4000 wide / 8000 narrow). Longer values retain their observed lengths.
It does not change field content, streaming, or error handling.

Diagnostic MyLog marker:

```
SQLSERVER_LOB_EXPERIMENT_V4: type=25 observed=31 reported=8192
```

**Important**: This is an experimental workaround, not a confirmed fix.
Retest all 13 original cases, especially short TEXT, JSONB, real 8-column
table, and unconstrained VARCHAR. In addition, observe potential
SQL Server 7347 (expected length mismatch), 7356 (other metadata
flags), and 7341 (fetch truncation). If V4 fails, investigate MSDASQL
OLE DB metadata rather than blindly increasing sizes. Do not deploy
unvalidated code to production.


## V4 validation result: failed short LOB values

Windows standard x64 CI passed, but **real SQL Server/MSDASQL tests did not**:
7/13 PASS. Tests 2 (short TEXT), 6 (short JSONB), 8–11 (real
`CFG.t_service_config_var`) and 13 (short unconstrained VARCHAR) failed
with SQL Server **7341**. Tests 1, 3–5, 7, and 12 passed.

MyLog definitively shows the V4 implementation executing:

```text
SQLSERVER_LOB_EXPERIMENT_V4: type=25 observed=31 reported=8192
SQLDescribeCol column_size=8192, SQL_WLONGVARCHAR=-10
SQLGetData first request BufferLength=0 -> expected warning
SQLGetData second request BufferLength=64 -> 62 UTF-16 bytes
SQLSERVER_LOB_EXPERIMENT_V4: type=3802 observed=10 reported=8192
SQLGetData second request BufferLength=11 -> 10 ANSI bytes
```

Thus, the string is read correctly by the ODBC driver on the second call,
yet the SQL Server consumer rejects the row with 7341. This suggests a
problem in the OLE DB bridge's handling of metadata/row access, **not a
proven psqlODBC data truncation**.

**Do not deploy V4 in production.** Do not increase fabricated sizes
further without capturing the MSDASQL-side status or a tightly scoped
reproduction. The next diagnostic may use a session-scoped SQL Server
trace flag 7300 (if sysadmin privileges are available) and a boundary
test of actual lengths just below/equal/above 8192.


## V5 follow-up (2026-10-20)

V4 boundary probes using a single TEXT value yielded:

| Length in characters | Result |
|---:|---|
| 8191, 8192, 8193, 16384 | SQL Server 7341 |
| 20000, 24576, 28672, 30720, 32000, 32767, 32768, 65536, 100000 | PASS |

A **mixed two-row result set**, containing 31-character and 32768-character
TEXT values in the same column, succeeded for both rows. SQL Server reported
62 and 65536 bytes respectively. Therefore the long-value handling of the
**entire result set** appears relevant, not merely each row's actual length.

V5 is an experimental test: replace V4's 8192-character minimum returned
`ColumnSize` with **32768 characters** for short opted-in TEXT, unconstrained
VARCHAR, XML, JSON, and JSONB. Keep V3's stable ODBC long-type change,
preserve `MaxLongVarcharSize=-4` opt-in and report actual longest length
for results longer than the floor. New MyLog marker:
`SQLSERVER_LOB_EXPERIMENT_V5: type=25 observed=31 reported=32768`.

The size is a **reported metadata lower bound, not an actual
value truncation limit**. It may incur a large memory/performance cost
on wide/multicolumn queries and is not an established MSDASQL contract.
Even though a mixed real result with 32768-char max length worked, there is
no guarantee that *fabricating* a 32768-char maximum for standalone 31 chars
will work. Test all 13 diagnostic SQL cases on the isolated second server,
especially short TEXT/JSONB, unconstrained VARCHAR, actual DWH table and
multi-row stress; do not deploy this experimental build in production.


## V5: 12/13 SQL Server diagnostic tests passed

V5 was tested against SQL Server 2025 Enterprise -> MSDASQL ->
psqlODBC Unicode x64 -> PostgreSQL 16.2. Results:

- PASS: 100k TEXT, short TEXT (31 chars), empty/NULL TEXT,
  mixed short/long TEXT, 1,589,499-char XML, large JSONB,
  actual `CFG.t_service_config_var` columns, entire real table,
  short and long unconstrained VARCHAR.
- FAIL: **small JSONB** (`jsonb_build_object('a', 'X')`),
  SQL Server error 7341.

In `pgtype_attr_to_concise_type`, TEXT and XML use
`ansi_to_wtype(conn, SQL_LONGVARCHAR)` and become
`SQL_WLONGVARCHAR=-10` on a Unicode connection. However, JSON and JSONB
(OIDs 114 and 3802) take the default unknown-type branch, which returns
`SQL_LONGVARCHAR=-1` even on a Unicode connection. This difference is
supported by earlier MyLog (JSONB SQLGetData with SQL_C_CHAR).

## V6: targeted JSON/JSONB Unicode long mapping hypothesis

For these exact PostgreSQL OIDs, under the existing opt-in
`CC_is_in_unicode_driver(conn)`, `MaxLongVarcharSize=-4`, and
`UnknownsAsLongVarchar=1`, use
`ansi_to_wtype(conn, SQL_LONGVARCHAR)` to return `SQL_WLONGVARCHAR=-10`.
All other unknown PostgreSQL OIDs retain their V5 behavior.

V5's empirical 32768-character metadata floor is unchanged.
MyLog marker:
`SQLSERVER_LOB_EXPERIMENT_V6: json_oid=3802 sql_type=SQL_WLONGVARCHAR`

**Not validated in SQL Server yet.** Important checks after installing on the
non-production SQL Server: original 13-test suite, especially short JSONB
(test 6), small PostgreSQL JSON (OID 114), large JSONB, JSON containing
non-ASCII text, real `CFG.t_service_config_var` table, and short/long TEXT.
This opt-in change can affect provider conversion and metadata buffers.
Do not install in production without regression/performance testing.
