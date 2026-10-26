# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Workspace Type

**Application platform: IBM Z**
**Detected languages:** Enterprise PL/I for z/OS (CICS + DB2 online, batch), JCL, CICS CSD definitions.
GenApp (General Insurance Application) PL/I variant. Sources: `pli/` (programs), `plinc/` (include files = copybooks), `jcl/` (DB2 DDL, bind, CSD), `Resources/` (CSD definitions), `application-conf/` (zAppBuild config).

## Build / Test

No local build. Compilation happens on z/OS via IBM zAppBuild (DBB/Groovy); there is no make/gradle/npm and no lint step.
- Config: [`application-conf/PLI.properties`](application-conf/PLI.properties) (compiler `V52`, `pli_compileMaxRC=4`, `pli_linkEditMaxRC=0`, link parms `MAP,RENT,COMPAT(PM5)`), build order in [`application-conf/application.properties`](application-conf/application.properties).
- CICS programs compile with `SYSTEM(CICS),PP(MACRO,CICS)` and deploy as `CICSLOAD`; others deploy as `LOAD`.
- For single-program build/verify from the IDE use the `userbuild-run-user-build` tool (ask user full vs. single-file upload first) and `get_diagnostics` for local syntax checks.
- Tests: TAZ unit tests via `TazUnitTest.groovy` (`runzTests=true`). PL/I test programs live in `**/testcase/*.pli` (`pli_testcase=true` in [`application-conf/file.properties`](application-conf/file.properties)). `.rec` recordings are binary — never edit as text. No testcase directory currently exists in the repo.

## Encoding / Line Endings — CRITICAL

Git stores UTF-8; files convert to EBCDIC IBM-1047 on z/OS transfer ([`.gitattributes`](.gitattributes)). `.rec`/`.plbck` are binary. LF line endings only. Do not change encoding or line endings.

## PL/I Source Conventions

- Cols 73–80 hold sequence numbers in `plinc/*.inc` and in many (not all) `pli/*.pli` files. Preserve them; keep new lines consistent with the file's existing style (some programs have none).
- Includes: `%INCLUDE NAME;` (no path/extension; resolved through `pliincludeSearch` = `plinc/*.inc`) for COMMAREA/maps (`LGCMAREA`, `LGPOLICY`, `SSMAP`, `LGCMARER`); `EXEC SQL INCLUDE` for SQLCA/DCLGEN members.
- `plinc/LGCMAREA.inc`: `COMM_AREA` is a `UNION` over a fixed `CHAR(32500)` buffer; header is 18 bytes and `CA_REQUEST_INITIALISER` is `CHAR(32482)`. Any new request-specific structure must fit within 32482 bytes. Changing COMMAREA/`LGPOLICY` layouts affects every program that includes them.
- CICS entry: `Proc(COMM_AREA_PTR) Options(Main)`; BMS menu programs (`LGTESTP1-4`, `LGTESTC1`) and `LGDPDB01` use `Proc(DFHEIPTR,COMM_AREA_PTR) Options(Main,Reentrant)`. Batch programs (`LGBATCH`, `LGBATDB2`) use `Proc Options(Main)` with no COMMAREA.
- `CA_REQUEST_ID` (6 chars, e.g. `'01IMOT'`) selects the operation. `CA_RETURN_CODE`: `'00'` ok, `'98'` bad COMMAREA length, `'99'` DB2/system error. Abend `LGCA` when `EIBCALEN=0`.
- Each program has a `WS_EYECATCHER` (`'PROGNAME…-WS'`). Errors go to a CICS TS queue via a `WRITE_ERROR_MESSAGE` procedure (not ABEND); check `SQLCODE` after every SQL statement, on error write message and `EXEC CICS RETURN`.
- DB2 CHAR host variables must match column lengths exactly; INTEGER columns use `FIXED BIN(31)`.
- Statistics: most online programs link `LGSTSQ` (TS queue / named-counter updates) on success and failure paths.

## Program Naming / Call Flow

`LG` + op (`A`dd/`D`elete/`I`nquire/`U`pdate) + entity (`C`ustomer/`P`olicy) + layer (`DB`=DB2, `VS`=VSAM, `OL`/`US`=orchestrator) + `01`.

```
LGTESTP1-4 / LGTESTC1 (menus)
  └─ LINK literal ─► orchestrators LGxPOL01 / LGxCUS01
        └─ LINK (usually program name held in a variable; literal in LGAPOL01/LGUPOL01) ─► DB2 backend LGxPDB01 / LGxCDB01
              └─ LINK ─► VSAM counterpart LGxPVS01 / LGxCVS01
```
- LGAPOL01 → LGAPDB01 (+ optional `LGAPBR01` only when `BUSINESS_RULES='Y'`; flag defaults `'N'`, the MOVE to `'Y'` is commented out; `LGAPBR01` source is not in this repo).
- LGACDB01 / LGACUS01 also LINK `LGACVS01` and `LGACDB02` (customer security, table `CUSTOMER_SECURE`).
- `LGBATCHF` is a subprogram (internal proc `LGBATCF`) called statically by `LGBATCH`, not via CICS LINK.
- Customer numbers come from CICS named counter `GENACUSTNUM` (pool `GENA`); `LGSETUP` initializes the counters.
- Policy-type tables (`ENDOWMENT`, `HOUSE`, `MOTOR`, `COMMERCIAL`, `CLAIM`) are inserted only by `LGAPDB01`; `POLICY` is inserted by `LGAPDB01` and `LGACUS01`; `CUSTOMER` by `LGACDB01` and `LGACUS01`.

## Scanned Database

Scanner DB path is in `.bobz/local-settings.json` (`databaseLocation`). Use `execute_sql_query` (call `get_sql_schema` with `local` first) for call chains, SQL table access and include dependencies before reading source. Expanded source is cached under `.bobz/expanded/`.

**Prefer the local metadata database over source searching** (`grep`, `glob`, reading `pli/*.pli`) whenever you need to understand:
- program dependencies (LINK/CALL chains, callers and callees, include usage),
- Db2 data usage (which programs access which tables, and with which statement type: SELECT, INSERT, UPDATE, DELETE, cursor),
- data flows (where a variable or field is defined, read and written).

Query the database first (`execute_sql_query`, `get_program_dependencies`, `get_paragraphs`) and only read source for the specific programs and line ranges the query points to. Use source search as a fallback or to verify, and when you do, say which result came from which. Known gaps seen so far, so cross-check before concluding "no usage":
- The table-access results can miss statements (e.g. the `UPDATE MOTOR` in `LGUPDB01` was not returned); confirm with a source search of the SQL verb and table name.
- Include joins for PL/I `%INCLUDE` members (`LGCMAREA`, `LGPOLICY`, `SSMAP`) can return no rows; use source search for those.
- Scanner line numbers refer to expanded source (`.bobz/expanded/`), so they differ from the line numbers in `pli/`.

## Data Dictionary / Docs / Standards

- No `bobz/DD.json` data dictionary exists yet; the `data-dictionary-management` skill would create one when COBOL/PL/I files are analyzed.
- `docs/` is empty; no technical documentation mapping exists.
- No coding standards skill is configured for this workspace; use the PL/I conventions above.
- [`zapp.yaml`](zapp.yaml) is a default generated file with `**` search for syslib/maclib/proclib; narrow `locations` (e.g. `plinc`) if resolution is slow.
