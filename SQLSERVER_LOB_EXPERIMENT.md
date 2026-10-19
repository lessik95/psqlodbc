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
