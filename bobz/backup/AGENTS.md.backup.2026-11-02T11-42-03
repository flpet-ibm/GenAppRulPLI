# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Overview

**GenApp (PL/I variant)** — IBM General Insurance Application implemented in PL/I. CICS online transactions + batch programs running on z/OS. Built using IBM zAppBuild (DBB/Groovy pipeline).

## Repository Structure

| Directory | Contents |
|-----------|----------|
| `pli/` | 35 PL/I source programs (`.pli`) |
| `plinc/` | 4 PL/I include files (`.inc`) — equivalent of COBOL copybooks |
| `jcl/` | JCL for DB2 DDL, CSD definitions, bind |
| `application-conf/` | zAppBuild build configuration properties |

## Build System

Build is via **IBM zAppBuild** (Groovy/DBB), not a local command. There is no local `make`/`gradle`/`npm`. All compilation happens on z/OS.

- PLI compiler version: `V52` (configured in [`application-conf/PLI.properties`](application-conf/PLI.properties))
- Default compile parms: `PP(INCLUDE('ID(++INCLUDE)'))` for plain; `SYSTEM(CICS),PP(MACRO,CICS)` for CICS programs
- Link edit parms: `MAP,RENT,COMPAT(PM5)`
- CICS programs deploy to `CICSLOAD`; batch programs to `LOAD`
- `pli_compileMaxRC=4` — RC=4 (warnings) is acceptable; link edit must be RC=0

## Testing

- TAZ unit tests run via `TazUnitTest.groovy` triggered by `runzTests=true` in [`application-conf/application.properties`](application-conf/application.properties)
- Test recording files (`.rec`) are binary — committed as binary in `.gitattributes`
- Test case programs: `**/testcase/*.pli` (set `pli_testcase=true` via [`application-conf/file.properties`](application-conf/file.properties))
- TAZ playback uses `EQAPPLAY`, parms: `STOP=E,REPORT=XML`

## Encoding — CRITICAL

All mainframe source files are stored in UTF-8 in Git and automatically converted to EBCDIC (IBM-1047) when transferred to z/OS. Controlled by [`.gitattributes`](.gitattributes):
- `.pli`, `.inc`, `.jcl`, `.properties`, `.cbl`, `.bms`, `.asm` → `zos-working-tree-encoding=ibm-1047 git-encoding=utf-8`
- `.rec`, `.plbck` → `binary` (never convert)

**Do not change line endings or encoding** of source files manually.

## PL/I Code Patterns

### Include files
- Programs use **both** `%INCLUDE` (compile-time macro preprocessor) and `EXEC SQL INCLUDE`:
  - `%INCLUDE LGCMAREA;` — COMMAREA structure (all CICS programs)
  - `%INCLUDE SSMAP;` — BMS map structures (menu programs LGTESTP1–4, LGTESTC1)
  - `%INCLUDE LGCMARER;` — Business rule request area (LGAPOL01 → LGAPBR01)
  - `EXEC SQL INCLUDE SQLCA;` + `EXEC SQL INCLUDE LGCMAREA;` — DB2 programs
- Include search path: `plinc/*.inc` (configured in [`application-conf/application.properties`](application-conf/application.properties) as `pliincludeSearch`)

### COMMAREA convention
- **All CICS programs** receive `COMM_AREA_PTR` as the procedure parameter: `Proc(COMM_AREA_PTR) Options(Main)`
- Menu programs additionally receive `DFHEIPTR`: `Proc(DFHEIPTR,COMM_AREA_PTR) Options(Main,Reentrant)`
- `CA_REQUEST_ID` (6 chars) encodes the operation: e.g. `'01IMOT'` = Inquire Motor policy
- `CA_RETURN_CODE` = `'00'` for success, `'98'` for bad COMMAREA length, `'99'` for DB2/system errors
- COMMAREA is validated for minimum length at program entry; abend `LGCA` if EIBCALEN=0

### Error handling pattern
- Every program declares a `WS_EYECATCHER` initialized to `'PROGNAMExxxxxx-WS'` for dump identification
- Error messages written to CICS TS queue (not ABEND) via internal `WRITE_ERROR_MESSAGE` procedure
- DB2 errors: check `SQLCODE` after every SQL statement; non-zero → write error and `EXEC CICS RETURN`

### Program naming convention
```
LG + operation(A=Add,D=Delete,I=Inquire,U=Update) + entity(C=Customer,P=Policy) + backend(DB=DB2,VS=VSAM,US=Combined) + 01
```
- `LGACDB01` = **L**ife**G**eneral + **A**dd **C**ustomer + **D**B2 + instance 01
- `LGAPOL01` = Add Policy (orchestrator); calls `LGAPDB01` and optionally `LGAPBR01` (business rules)
- `LGTESTP1-4` = Menu programs (Motor, Endowment, House, Commercial)
- `LGTESTC1` = Customer menu

### Batch programs
- `LGBATCH` / `LGBATDB2` — non-CICS; use `Proc Options(Main)` without COMMAREA parameter
- Files declared with `DCL filename FILE RECORD INPUT SEQUENTIAL ENV(FB RECSIZE(n))`
- `LGBATCHF` is a subprogram called via static `CALL` (not CICS LINK)

## DB2 Tables

| Table | Programs that write | Programs that read |
|-------|--------------------|--------------------|
| CUSTOMER | LGACDB01, LGACUS01 | LGBATCH, LGBATDB2, LGICDB01 |
| CUSTOMER_SECURE | LGACDB02 | — |
| POLICY | LGACUS01, LGAPDB01 | LGDPDB01, LGIPDB01 |
| ENDOWMENT/HOUSE/MOTOR/COMMERCIAL/CLAIM | LGAPDB01 | LGIPDB01 |
| MOTOR | LGMOTOUT (read/write) | — |

## CICS Program Call Architecture

```
Menu (LGTESTP1-4 / LGTESTC1)
  └─ CICS LINK ──► Policy orchestrator (LGAPOL01 / LGDPOL01 / LGIPOL01 / LGUPOL01)
                      └─ CICS DYNAMIC_LINK ──► DB2 backend (LGAPDB01 / LGDPDB01 / etc.)
                      └─ CICS LINK ──► Business rules (LGAPBR01) [optional]
  └─ CICS LINK ──► Customer orchestrator (LGACUS01 / LGICUS01 / LGUCUS01)
                      └─ CICS DYNAMIC_LINK ──► DB2 backend (LGICDB01 / LGUCDB01)
```

- `LGAPVS01`, `LGIPVS01`, etc. = VSAM counterparts (parallel to DB2 backends)
- `LGACDB01` also calls `LGACVS01` (CICS LINK) and `LGACDB02` (customer security, CICS LINK)

## Scanned Database

Program metadata is stored in `.bobz/local-settings.json` (`databaseLocation` field). Use `execute_sql_query` for cross-program analysis (call chains, SQL table access, dependencies).
