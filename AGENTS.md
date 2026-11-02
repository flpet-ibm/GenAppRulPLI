# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Workspace Type

**Application platform: IBM Z**
**Detected languages:** Enterprise PL/I for z/OS (CICS, Db2, batch), JCL
**Project:** GenApp (PL/I variant), the IBM General Insurance application. It is built with IBM zAppBuild (DBB/Groovy).

## Repository Layout

| Directory | Purpose |
|-----------|---------|
| `pli/` | PL/I programs. Most have no sequence numbers in columns 73-80. |
| `plinc/` | PL/I include files (`.inc`, the equivalent of COBOL copybooks). They DO carry sequence numbers in columns 73-80. |
| `jcl/` | Db2 DDL, bind and CSD definition jobs |
| `Resources/` | CICS CSD definitions |
| `application-conf/` | zAppBuild properties |
| `.bobz/` | Scanner settings, expanded sources, impact analysis reports |

The BMS source for mapset `SSMAP` and the `LGAPBR01` business rules program are NOT in this repo. `plinc/SSMAP.inc` is only the PL/I symbolic map.

## Build, Lint and Test

There is no local build, lint or test runner. Everything runs on z/OS through zAppBuild. Use `get_diagnostics` (Z Open Editor) for local syntax checks.

- Compiler `V52`. Compile RC up to 4 is accepted; link edit must be RC=0 (`application-conf/PLI.properties`).
- Compile parms: `PP(INCLUDE('ID(++INCLUDE)')),SOURCE`. CICS programs also get `SYSTEM(CICS),PP(MACRO,CICS)`.
- Link edit parms are `MAP,RENT,COMPAT(PM5)`. CICS programs deploy as `CICSLOAD`, all others as `LOAD`.
- `userbuild-run-user-build` builds a single program. Ask the user whether to do a full upload first. A full upload is required the first time, and again when include files change.
- A change to any include requires an impact build or full build, not a single-file build. Almost every program includes `LGCMAREA` or `LGPOLICY`.
- Tests are TAZ unit tests. `runzTests=true` is set in `application-conf/application.properties`, and `application-conf/TazUnitTest.properties` configures them. Test case programs match `**/testcase/*.pli` (`pli_testcase=true` in `application-conf/file.properties`).
- To run a single test, use `edt_run_tests` on the one test file. For a pipeline run, point `TazUnitTest.groovy` at one `*.bzucfg`.
- Playback uses `EQAPPLAY` with `STOP=E,REPORT=XML`. `.rec` and `.plbck` files are binary, so never edit them as text.
- No `testcase/` or `*.bzucfg` files are currently in the repo.

## Encoding and Line Endings (critical)

- Sources are UTF-8 in Git and convert to EBCDIC IBM-1047 on transfer (`.gitattributes`: `zos-working-tree-encoding=ibm-1047 git-encoding=utf-8`).
- Keep line endings LF only. Never introduce CRLF.
- In `.inc` files, keep the columns 73-80 sequence numbers when editing, and number any new lines consistently.

## PL/I Conventions

### Includes
- Use `%INCLUDE NAME;` with no path and no extension. The search path is `plinc/*.inc` (`pliincludeSearch`). A quoted path breaks the DBB scanner.
- Db2 programs use `EXEC SQL INCLUDE SQLCA;` and `EXEC SQL INCLUDE LGCMAREA;`.

### COMMAREA (`plinc/LGCMAREA.inc`)
- It is the only data contract between programs. Layout: 6-byte `CA_REQUEST_ID`, 2-digit `CA_RETURN_CODE`, 10-digit `CA_CUSTOMER_NUM`, then `CA_REQUEST_SPECIFIC` (`CHAR(32482)`), 32500 bytes in total.
- Every branch of the union must keep the overall size unchanged. When adding a field, shrink the branch's filler field by the same number of bytes.
- CICS programs: `Proc(COMM_AREA_PTR) Options(Main)`. Menu programs: `Proc(DFHEIPTR,COMM_AREA_PTR) Options(Main,Reentrant)`.
- Return codes: `'00'` OK, `'98'` COMMAREA too short, `'99'` Db2 or system error. Set the code, then `EXEC CICS RETURN` immediately.
- `CA_REQUEST_ID` encodes the operation, for example `'01IMOT'` = inquire motor.

### Length constants are duplicated
- `WS_MOTOR_LEN` and `WS_FULL_MOTOR_LEN` (and similar constants) are declared in `plinc/LGPOLICY.inc` AND locally in some programs (`LGUPOL01`, `LGUCUS01`). Update every copy together.
- `WS_MOTOR_LEN` (65) is smaller than the `CA_MOTOR` structure. `premium` and `accidents` are moved to the COMMAREA separately in `LGIPDB01`, so a new trailing field needs an explicit assignment.
- Db2 CHAR host variables must match the column length exactly.
- A nullable column needs an indicator variable on SELECT. Otherwise SQLCODE -305 occurs on existing rows.

### Error handling
- Every program declares `WS_EYECATCHER`, initialised to `'PROGNAMExxxxxx-WS'`, for dump identification.
- Errors go to a CICS TS queue via the internal `WRITE_ERROR_MESSAGE` procedure (and `LGSTSQ`), not to an ABEND. The exception is `EIBCALEN=0`, which abends `LGCA`.
- Check `SQLCODE` after every SQL statement.

### Naming
`LG` + operation (A/D/I/U) + entity (C=Customer, P=Policy) + backend (`DB`=Db2, `VS`=VSAM, `OL`/`US`=orchestrator) + `01`.
Menus are `LGTESTP1-4` (Motor, Endowment, House, Commercial) and `LGTESTC1` (Customer). Batch programs are `LGBATCH`, `LGBATDB2`, `LGMOTOUT` and `LGBATCHF`, a static-CALL submodule.

## Architecture

```
Menu (LGTESTP1-4 / LGTESTC1) -- CICS LINK --> Orchestrator (LGxPOL01 / LGxCUS01)
Orchestrator -- CICS LINK --> Db2 backend (LGxPDB01 / LGxCDB01) or VSAM backend (LGxPVS01)
LGAPOL01 -- CICS LINK --> LGAPBR01 (optional business rules, not in repo)
LGACDB01 -- CICS LINK --> LGACVS01, LGACDB02 (customer security)
```

- Orchestrators contain no SQL. `LGAPOL01` links to literal program names, while `LGIPOL01` and `LGDPOL01` link through a variable.
- Update policy flow: `LGUPOL01` -> `LGUPDB01` (Db2). It also links `LGUPVS01`.
- Customer numbers come from a CICS named counter (`GENACUSTNUM`, pool `GENA`). If it is unavailable, `LGACDB01` silently falls back to customer 0.

## Db2 Tables (from scanner and source)

| Table | Written by | Read by |
|-------|-----------|---------|
| CUSTOMER | LGACDB01, LGACUS01, LGUCDB01 | LGICDB01, LGBATCH, LGBATDB2 |
| CUSTOMER_SECURE | LGACDB02 | - |
| POLICY | LGAPDB01, LGACUS01, LGDPDB01 | LGIPDB01 |
| ENDOWMENT, HOUSE, MOTOR, COMMERCIAL, CLAIM | LGAPDB01 (MOTOR is also updated by LGUPDB01) | LGIPDB01 (MOTOR also by LGMOTOUT) |

The scanner misses some UPDATE statements, for example `UPDATE MOTOR` in `LGUPDB01`. Confirm with `grep` before trusting a scanner "no access" result.

## Scanned Database

`.bobz/local-settings.json` (`databaseLocation`) points to the scanner database. Query it with `execute_sql_query` first for call chains, table access and include usage. Its include (copybook) data is incomplete for PL/I, so use `grep` for `%INCLUDE`. Scanner line numbers refer to expanded source (`.bobz/expanded/`), not the original files.

## Technical Documentation

**Location:** `application-conf/README.md` (zAppBuild property reference) and `.bobz/impact-analysis/<change>/IMPACT-ANALYSIS.md`.

| Subject | Documentation | Purpose |
|---------|--------------|---------|
| Build properties | `application-conf/README.md` | Which build parameters can be overridden per file |
| MOTOR table change | `.bobz/impact-analysis/motor-add-nullable-column-*/IMPACT-ANALYSIS.md` | Impact of adding a nullable MOTOR column: Db2, COMMAREA, BMS, VSAM and batch |
| MOTOR programs (`LGAPDB01`, `LGIPDB01`, `LGUPDB01`, `LGMOTOUT`, `LGTESTP1`, `LGUPOL01`, `LGUCUS01`) and includes (`LGCMAREA`, `LGPOLICY`, `SSMAP`) | the same report | Per-program change points and risks |

**Auto-update rules:**
1. When code changes, check whether the mapping above covers it, and flag documentation that is now out of sync.
2. When analysing a mapped program, read its documentation first.
3. Add new programs or documents to the table when they appear.

## Coding Standards

No coding standard skill is available in this workspace. If one is added, apply it to all generated or modified PL/I and JCL.
