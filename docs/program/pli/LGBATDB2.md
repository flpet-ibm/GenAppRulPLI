## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
- [3. Outputs](#3-outputs)
- [4. Processing Logic](#4-processing-logic)
  - [4.1 Program Initialization and Setup](#41-program-initialization-and-setup)
  - [4.2 Customer Data Retrieval](#42-customer-data-retrieval)
  - [4.3 Report Generation and Output Processing](#43-report-generation-and-output-processing)
  - [4.4 Termination](#44-termination)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
- [7. Constraints](#7-constraints)
  - [7.1 Data Constraints](#71-data-constraints)
  - [7.2 Sequencing and Flow Constraints](#72-sequencing-and-flow-constraints)
  - [7.3 Interface Constraints](#73-interface-constraints)
  - [7.4 Error Handling Constraints](#74-error-handling-constraints)
- [8. Error Handling](#8-error-handling)
  - [8.1 Db2 SQLCODE / SQLSTATE Diagnostic Display](#81-db2-sqlcode--sqlstate-diagnostic-display)
  - [8.2 Absence of Post-SELECT Error Checking](#82-absence-of-post-select-error-checking)
  - [8.3 No PL/I ON-Condition Handlers](#83-no-pli-on-condition-handlers)
  - [8.4 No Return-Code or Status Propagation](#84-no-return-code-or-status-propagation)
  - [8.5 Program Termination](#85-program-termination)
- [9. Examples](#9-examples)
  - [9.1 Example 1: Customer 1 found in DB2](#91-example-1-customer-1-found-in-db2)
  - [9.2 Example 2: Names containing blanks](#92-example-2-names-containing-blanks)
  - [9.3 Example 3: Customer 1 not in the table](#93-example-3-customer-1-not-in-the-table)

## 1. Purpose

LGBATDB2 is a batch (non-CICS) test driver for the customer reporting subroutine LGBATCHF. It looks up one fixed customer, customer number 1, in the DB2 CUSTOMER table. It reads that customer's name, date of birth, address, phone numbers and email address into a shared communication area. It then passes the customer number and that area to LGBATCHF, which builds a formatted 80-character report line. The program displays that line as the result of the run. It also displays the SQL return code and state at startup. Together these give a quick end-to-end check of the DB2 retrieval and the reporting logic outside the online environment.

## 2. Inputs

- **Program parameters and console input**
  - The program is a batch main procedure (`Options(Main)`). It takes no parameters and reads nothing from the console, a file, or the environment.

- **Hard-coded driver value**
  - `DB2_CUSTOMERNUMBER_INT`
    - The program sets this to the constant 1 before it processes the record.
    - It is the only selection key. It is the customer number used in the `WHERE CUSTOMERNUMBER = :DB2_CUSTOMERNUMBER_INT` clause of the `CUSTOMER` table lookup.
    - Changing which customer is reported means changing this value in the source.

- **External data source: Db2 `CUSTOMER` table**
  - The `SELECT` reads one customer row. It returns these columns into the communication area customer request fields:
    - `CA_FIRST_NAME` comes from `FIRSTNAME`.
    - `CA_LAST_NAME` comes from `LASTNAME`.
    - `CA_DOB` comes from `DATEOFBIRTH`.
    - `CA_HOUSE_NAME` comes from `HOUSENAME`.
    - `CA_HOUSE_NUM` comes from `HOUSENUMBER`.
    - `CA_POSTCODE` comes from `POSTCODE`.
    - `CA_PHONE_MOBILE` comes from `PHONEMOBILE`.
    - `CA_PHONE_HOME` comes from `PHONEHOME`.
    - `CA_EMAIL_ADDRESS` comes from `EMAILADDRESS`.
  - The Db2 environment and the `CUSTOMER` table contents determine what these fields hold.

- **External program dependency**
  - `LGBATCHF` is loaded with `fetch` and called with `CUSTIN_CUSTNUMBER`, `COMM_AREA_PTR` and `CUSTRPT_LINE`.
    - Its behavior and the report text it builds depend on the external module.
    - `CUSTIN_CUSTNUMBER` is loaded from `DB2_CUSTOMERNUMBER_INT` and passed in as the customer number.
    - `CUSTRPT_LINE` is the 80-character line that `LGBATCHF` fills in and the program then displays.
    - The communication area, addressed through `COMM_AREA_PTR` over `BATCH_COMM_AREA`, carries the Db2-populated customer fields to `LGBATCHF`.

- **Shared communication area structure**
  - The `COMM_AREA` layout comes from the included `LGCMAREA` definition.
  - The program does not set or read the other communication area fields (`CA_CUSTOMER_NUM`, `CA_NUM_POLICIES`, `CA_POLICY_NUM`, `CA_PAYMENT`) before calling `LGBATCHF`. Their contents are whatever the buffer holds or `LGBATCHF` sets.

- **Db2 runtime status data**
  - The SQL communication area (`SQLCA`, with `SQLCODE` and `SQLSTATE`) is supplied by Db2. The program displays these values before the lookup.

## 3. Outputs

- **Printed messages (console output via DISPLAY)**
  - Startup diagnostics
    - `'LGBATDB2 SQLCode : ' || SQLCODE` is displayed after the subroutine is fetched. It shows the SQL return code in the SQLCA at that point, before any SQL statement has run.
    - `'LGBATDB2 SQLState: ' || SQLSTATE` is displayed next. It shows the SQL state in the SQLCA at that point.
  - Customer report line
    - `CUSTRPT_LINE` is the main result of the batch run. After the reporting subroutine fills it, the program displays it.
    - It holds an 80-character line in the form "Customer <number> is <first name> <last name>".
    - The subroutine also displays the same text with the prefix `LGBATCHF WAS HERE... `.

- **Report line and its inputs**
  - Report line
    - `CUSTRPT_LINE` is returned by the reporting subroutine and displayed by this program.
  - Customer identification
    - `DB2_CUSTOMERNUMBER_INT` is set to 1. It is the key for the CUSTOMER table lookup.
    - `CUSTIN_CUSTNUMBER` is loaded from `DB2_CUSTOMERNUMBER_INT`. It is passed to the subroutine and appears as the customer number in the report line.
  - Customer name fields
    - `CA_FIRST_NAME` is filled from FIRSTNAME. The subroutine uses its text up to the first blank in the report line.
    - `CA_LAST_NAME` is filled from LASTNAME. The subroutine uses its text up to the first blank in the report line.

- **Data returned from the database into the communication area**
  - These fields are filled by the SELECT from the CUSTOMER table and are passed by pointer to the reporting subroutine.
  - Personal details
    - `CA_DOB` comes from DATEOFBIRTH.
  - Address details
    - `CA_HOUSE_NAME` comes from HOUSENAME.
    - `CA_HOUSE_NUM` comes from HOUSENUMBER.
    - `CA_POSTCODE` comes from POSTCODE.
  - Contact details
    - `CA_PHONE_MOBILE` comes from PHONEMOBILE.
    - `CA_PHONE_HOME` comes from PHONEHOME.
    - `CA_EMAIL_ADDRESS` comes from EMAILADDRESS.
  - The subroutine does not use these fields in the report line. They are populated and available in the communication area.

- **Communication area fields declared but not set by this program**
  - `CA_CUSTOMER_NUM`, `CA_NUM_POLICIES`, `CA_POLICY_NUM` and `CA_PAYMENT` belong to the shared communication area layout.
  - The program never assigns them. They do not affect the report line or any other output.

- **Other side effects and return values**
  - The program has no file writes and no API calls. It does not write to the database, and the only database access is the single SELECT.
  - The program returns no explicit value. It ends with a plain `RETURN`.
  - The SQL result codes (SQLCODE and SQLSTATE) after the SELECT are not checked or displayed. If the customer is not found, the communication area and the report line are built from whatever the fields already hold.
  - The subroutine call is a dynamic fetch of `LGBATCHF`, and the call passes the customer number, the communication area pointer and the report line by reference.

## 4. Processing Logic

### 4.1 Program Initialization and Setup
- Pointer and Memory Mapping
  - Maps the pointer `COMM_AREA_PTR` to the local storage buffer `BATCH_COMM_AREA` using the `ADDR` built-in function to provide backing storage for the `COMM_AREA` structure.
- Subroutine Loading
  - Loads the external reporting procedure `LGBATCHF` dynamically into memory using the `FETCH` statement.
- Status Logging
  - Displays initial Db2 diagnostic status (`SQLCODE` and `SQLSTATE`) to standard system output (`SYSOUT`).
- Key Assignment
  - Sets the binary integer host variable `DB2_CUSTOMERNUMBER_INT` to the literal value `1`, designating the target customer identifier for processing.

### 4.2 Customer Data Retrieval
- Db2 Query Execution
  - Executes an embedded singleton `SELECT` statement against the `CUSTOMER` table in the Db2 relational database.
  - Matches table records where the `CUSTOMERNUMBER` column equals the host variable `:DB2_CUSTOMERNUMBER_INT`.
- Database Field Mapping
  - Retrieves customer demographic and contact data directly into the corresponding fields of the `CA_CUSTOMER_REQUEST` structure within `COMM_AREA`:
    - `FIRSTNAME` into `:CA_FIRST_NAME`
    - `LASTNAME` into `:CA_LAST_NAME`
    - `DATEOFBIRTH` into `:CA_DOB`
    - `HOUSENAME` into `:CA_HOUSE_NAME`
    - `HOUSENUMBER` into `:CA_HOUSE_NUM`
    - `POSTCODE` into `:CA_POSTCODE`
    - `PHONEMOBILE` into `:CA_PHONE_MOBILE`
    - `PHONEHOME` into `:CA_PHONE_HOME`
    - `EMAILADDRESS` into `:CA_EMAIL_ADDRESS`

### 4.3 Report Generation and Output Processing
- Argument Preparation
  - Assigns the numeric customer key from `DB2_CUSTOMERNUMBER_INT` to the 10-digit numeric picture host variable `CUSTIN_CUSTNUMBER`.
- Subroutine Invocation
  - Calls external subroutine `LGBATCHF`, passing three parameters:
    - Input customer identifier (`CUSTIN_CUSTNUMBER`)
    - Pointer to the populated communication area (`COMM_AREA_PTR`)
    - Target character buffer (`CUSTRPT_LINE`)
  - `LGBATCHF` formats the retrieved customer details into a formatted 80-character line.
- Report Line Display
  - Outputs the generated text line stored in `CUSTRPT_LINE` to standard system output via the `DISPLAY` statement.

### 4.4 Termination
- Flow Completion
  - Returns control cleanly to the operating system or invoking batch job step via the `RETURN` statement.

## 5. Paragraphs

- **LGBATDB2 (main procedure body)**
  - Purpose: Entry point of the batch (non-CICS) DB2 test driver. It reads customer 1 from the CUSTOMER table and displays the report line that the LGBATCHF subroutine builds.
  - Operations:
    - Points `COMM_AREA_PTR` at the 32500-byte `BATCH_COMM_AREA`. The based `COMM_AREA` structure (the `CA_*` fields) therefore overlays that buffer.
    - Dynamically loads the external routine `LGBATCHF` with `fetch`.
    - Displays `SQLCODE` and `SQLSTATE`, using the SQLCA included at the top of the program, before any SQL has run.
    - Sets `DB2_CUSTOMERNUMBER_INT` to 1, calls `PROCESS_RECORD`, then returns.
  - Control flow: Straight-line code with no conditions or loops. The only branch is the call to `PROCESS_RECORD`.
  - External interactions:
    - Loads the LGBATCHF module at run time.
    - Writes the SQLCODE and SQLSTATE messages to the console or SYSOUT with `DISPLAY`.

- **PROCESS_RECORD**
  - Purpose: Handles one customer. It gets the customer data from DB2, has the reporting subroutine format it, and displays the result.
  - Operations:
    - Calls `GET_CUSTOMER_INFO` to fill the customer fields of the commarea.
    - Moves `DB2_CUSTOMERNUMBER_INT` into the picture field `CUSTIN_CUSTNUMBER`.
    - Calls `LGBATCHF(CUSTIN_CUSTNUMBER, COMM_AREA_PTR, CUSTRPT_LINE)`. The subroutine builds the 80-character report line from the customer number and the commarea.
    - Displays `CUSTRPT_LINE`.
  - Control flow: Sequential. It makes two calls, one to the internal `GET_CUSTOMER_INFO` and one to the external `LGBATCHF`. There is no error checking of SQLCODE after the lookup.
  - External interactions:
    - Calls the external subroutine `LGBATCHF`.
    - Writes the report line with `DISPLAY`.
    - Reaches DB2 indirectly through `GET_CUSTOMER_INFO`.

- **GET_CUSTOMER_INFO**
  - Purpose: Reads the customer's details from DB2 into the commarea customer request fields.
  - Operations:
    - Runs a singleton `EXEC SQL SELECT ... INTO` against the CUSTOMER table, filtered by `CUSTOMERNUMBER = :DB2_CUSTOMERNUMBER_INT`.
    - Copies the nine selected columns into these host variables:

      | Column | Host variable |
      |---|---|
      | FIRSTNAME | `CA_FIRST_NAME` |
      | LASTNAME | `CA_LAST_NAME` |
      | DATEOFBIRTH | `CA_DOB` |
      | HOUSENAME | `CA_HOUSE_NAME` |
      | HOUSENUMBER | `CA_HOUSE_NUM` |
      | POSTCODE | `CA_POSTCODE` |
      | PHONEMOBILE | `CA_PHONE_MOBILE` |
      | PHONEHOME | `CA_PHONE_HOME` |
      | EMAILADDRESS | `CA_EMAIL_ADDRESS` |

  - Control flow: A single SQL statement. The procedure does not test SQLCODE for not-found or error conditions. If the SELECT fails, the commarea fields stay unchanged and processing continues.
  - External interactions: Reads from the DB2 CUSTOMER table. The commarea fields are written through the based `COMM_AREA` structure overlaid on `BATCH_COMM_AREA`.

## 6. Dependencies

- Database (Db2)
  - CUSTOMER table
    - Read with a singleton SELECT in GET_CUSTOMER_INFO.
    - Columns read: FIRSTNAME, LASTNAME, DATEOFBIRTH, HOUSENAME, HOUSENUMBER, POSTCODE, PHONEMOBILE, PHONEHOME and EMAILADDRESS.
    - The row is chosen with the predicate CUSTOMERNUMBER = :DB2_CUSTOMERNUMBER_INT.
  - Db2 SQL environment
    - The program is precompiled with embedded SQL (EXEC SQL), so it needs the Db2 precompiler or coprocessor.
    - At run time it needs an active Db2 connection and a plan or package that contains this program.
    - It does not check SQLCODE after the SELECT. It only displays SQLCODE and SQLSTATE before the SELECT runs.
  - SQLCA
    - Included with EXEC SQL INCLUDE SQLCA.
    - Provides the SQLCODE and SQLSTATE values that the program displays.

- External programs and subroutines
  - LGBATCHF
    - Declared as ENTRY(PIC '9999999999', PTR, CHAR(80)).
    - Loaded dynamically with the FETCH statement, then called from PROCESS_RECORD.
    - Takes the customer number (CUSTIN_CUSTNUMBER), the pointer to the communication area (COMM_AREA_PTR) and the output report line (CUSTRPT_LINE).
    - Builds the formatted customer report line from the communication area data.
    - Its load module must be available in the runtime library concatenation (STEPLIB or equivalent).

- Include members and copybooks
  - LGCMAREA (plinc/LGCMAREA.inc)
    - Defines the COMM_AREA structure, which is based on COMM_AREA_PTR.
    - The program expands it inline.
    - COMM_AREA supplies the host variables for the SELECT INTO: CA_FIRST_NAME, CA_LAST_NAME, CA_DOB, CA_HOUSE_NAME, CA_HOUSE_NUM, CA_POSTCODE, CA_PHONE_MOBILE, CA_PHONE_HOME and CA_EMAIL_ADDRESS.
    - It also defines the shared fields that LGBATCHF reads. These include CA_CUSTOMER_NUM, CA_NUM_POLICIES, CA_POLICY_NUM and CA_PAYMENT.
  - SQLCA
    - The standard Db2 communication area include described above.

- Input parameters and data sources
  - Hard-coded customer key
    - DB2_CUSTOMERNUMBER_INT is set to 1 in the main procedure.
    - It is the only input that selects which customer is processed.
    - The program has no external file, parameter, or environment input.
  - Working storage shared with the subroutine
    - BATCH_COMM_AREA is a 32,500-character buffer.
    - COMM_AREA_PTR points to it through ADDR.
    - It is the storage area that holds the Db2 results and that is passed to LGBATCHF.
    - DB2_IN_INTEGERS holds the host variables DB2_CUSTOMERNUMBER_INT, CUSTIN_CUSTNUMBER and CUSTRPT_LINE.

- Output destinations
  - System console or SYSOUT through the DISPLAY statement
    - Shows the SQLCODE and SQLSTATE lines.
    - Shows the 80-character report line (CUSTRPT_LINE) that LGBATCHF returns.

- Runtime and platform
  - Enterprise PL/I for z/OS, with OPTIONS(MAIN) making the program a batch (non-CICS) main entry point.
  - Language Environment runtime services, which support the dynamic FETCH of LGBATCHF and the DISPLAY output.
  - A batch job (JCL) that runs the program under a Db2 attachment such as DSN or IKJEFT01 and provides the load libraries and Db2 libraries.

## 7. Constraints

### 7.1 Data Constraints

**Fixed Target Customer**
- The program hard-codes `DB2_CUSTOMERNUMBER_INT = 1` before executing the database lookup; no other customer number can be processed in a single run
- The SQL `WHERE CUSTOMERNUMBER = :DB2_CUSTOMERNUMBER_INT` consequently restricts the SELECT to exactly one row — the row whose `CUSTOMERNUMBER` equals `1`

**Single-Row Result Expectation**
- The SQL uses a singleton `SELECT … INTO`, which assumes the WHERE clause returns at most one row; Db2 raises SQLCODE -811 if more than one row matches, and SQLCODE +100 if no row is found — both are unhandled, meaning any deviation from exactly one matching row silently produces incorrect or absent data in the host variables

**Host Variable Field Sizes (data truncation / overflow constraints)**
- `CA_FIRST_NAME` is declared `CHAR(10)` — FIRSTNAME values longer than 10 characters are silently truncated on retrieval
- `CA_LAST_NAME` is declared `CHAR(20)` — LASTNAME values longer than 20 characters are silently truncated
- `CA_DOB` is declared `CHAR(10)` — DATEOFBIRTH must fit in 10 characters
- `CA_HOUSE_NAME` is declared `CHAR(20)` — HOUSENAME must fit in 20 characters
- `CA_HOUSE_NUM` is declared `CHAR(4)` — HOUSENUMBER must fit in 4 characters
- `CA_POSTCODE` is declared `CHAR(8)` — POSTCODE must fit in 8 characters
- `CA_PHONE_MOBILE` is declared `CHAR(20)` — PHONEMOBILE must fit in 20 characters
- `CA_PHONE_HOME` is declared `CHAR(20)` — PHONEHOME must fit in 20 characters
- `CA_EMAIL_ADDRESS` is declared `CHAR(100)` — EMAILADDRESS values longer than 100 characters are silently truncated
- `CUSTRPT_LINE` is declared `CHAR(80)` — the output report line produced by `LGBATCHF` is bounded to exactly 80 characters; any formatted content that would exceed 80 characters is truncated by the fixed declaration
- `CUSTIN_CUSTNUMBER` is declared `PIC '9999999999'` (10 digits) — the customer number passed to `LGBATCHF` must be representable as a 10-digit unsigned integer; the assignment from `DB2_CUSTOMERNUMBER_INT` (FIXED BIN(31)) is constrained to values 0–9,999,999,999

**Nullable Column Restriction**
- None of the nine columns retrieved have indicator variables declared; if any of FIRSTNAME, LASTNAME, DATEOFBIRTH, HOUSENAME, HOUSENUMBER, POSTCODE, PHONEMOBILE, PHONEHOME, or EMAILADDRESS is nullable in the CUSTOMER table and contains a NULL, Db2 raises SQLCODE -305, which is unhandled, causing the program to abnormally terminate

**COMMAREA Size**
- `BATCH_COMM_AREA` is declared `CHAR(32500)`, matching the total size of the `COMM_AREA` union; the pointer `COMM_AREA_PTR` is set to address this fixed-size block; the structure must not exceed 32,500 bytes or storage corruption occurs

### 7.2 Sequencing and Flow Constraints

**Mandatory Execution Order**
- `COMM_AREA_PTR` must be set to `ADDR(BATCH_COMM_AREA)` before any reference to the based `COMM_AREA` structure; this is done at the top of the main body, and must precede the call to `PROCESS_RECORD`
- `LGBATCHF` must be `fetch`ed (dynamically loaded) before it can be called via `CALL LGBATCHF(…)`; the `fetch` statement is issued in the main body before `CALL PROCESS_RECORD`, enforcing load-before-call ordering
- Within `PROCESS_RECORD`, `GET_CUSTOMER_INFO` must complete before `CUSTIN_CUSTNUMBER` is assigned and before `LGBATCHF` is called, because the COMMAREA fields populated by the SQL SELECT are the input to the reporting subroutine

**Single-Pass Processing**
- The program calls `PROCESS_RECORD` exactly once and then executes `RETURN`; there is no loop, file input, or iteration mechanism, so the program is constrained to processing a single fixed customer per invocation

### 7.3 Interface Constraints

**LGBATCHF Call Signature**
- `LGBATCHF` is declared `ENTRY(PIC '9999999999', PTR, CHAR(80))`; callers must pass exactly three arguments in this order: a 10-digit picture customer number, a pointer to the COMMAREA, and an 80-character output line; mismatched argument types or counts produce undefined behaviour
- The COMMAREA pointer passed as the second argument must address a valid, fully initialised 32,500-byte area; passing an uninitialised or incorrectly sized area yields unpredictable results in the subroutine

**No CICS Environment**
- The program is declared `Options(Main)` without `SYSTEM(CICS)` or `REENTRANT`; it is a batch (non-CICS) program and must not be run under CICS; CICS-specific services (e.g., `EXEC CICS`) are unavailable and would fail at runtime

### 7.4 Error Handling Constraints

**No SQLCODE Checking After the SELECT**
- The `DISPLAY` calls at the start of the main body show the initial SQLCA values but occur before the SQL SELECT is executed; no SQLCODE check is performed after the SELECT inside `GET_CUSTOMER_INFO`; the program therefore has no mechanism to detect or recover from SQL errors (e.g., table not found, no row, multiple rows, NULL indicator missing)
- The program provides no ON-unit or condition handler for any PL/I or Db2 exception, meaning any runtime error results in an uncontrolled termination

## 8. Error Handling

### 8.1 Db2 SQLCODE / SQLSTATE Diagnostic Display

- After the `FETCH LGBATCHF` call and before any business logic executes, the program issues two `DISPLAY` statements that emit the current `SQLCODE` and `SQLSTATE` values to standard output
  - This serves as a passive diagnostic mechanism: the values are unconditionally printed regardless of whether they indicate success or failure, giving the batch operator visibility into the initial Db2 connection state at program startup
  - There is no conditional branch on these values; the program does not halt or alter its path based on what is displayed here

### 8.2 Absence of Post-SELECT Error Checking

- The `GET_CUSTOMER_INFO` procedure executes a singleton `SELECT … INTO` against the `CUSTOMER` table but performs no explicit check of `SQLCODE` after the statement completes
  - If the row is not found (`SQLCODE +100`), or if any other Db2 error occurs (negative `SQLCODE`), the program takes no corrective action and does not notify the operator of the failure
  - There is no `WHENEVER` directive (SQL error-handling directive) declared anywhere in the program to intercept non-zero `SQLCODE` values automatically
  - This represents a gap in error handling: Db2 errors from the SELECT are silently ignored and execution continues into `PROCESS_RECORD`, which will work with whatever (potentially uninitialised or stale) data is in the communication area

### 8.3 No PL/I ON-Condition Handlers

- The program declares no `ON` condition handlers (e.g., `ON ERROR`, `ON CONVERSION`, `ON OVERFLOW`, `ON UNDEFINEDFILE`) anywhere in its scope
  - Runtime exceptions such as data conversion errors or arithmetic overflow would propagate to the PL/I default system action, which typically results in a program abend with a system dump rather than a controlled error message

### 8.4 No Return-Code or Status Propagation

- After calling `LGBATCHF`, the program displays the formatted report line unconditionally without inspecting any return indicator from the subroutine
  - There is no check of `CA_RETURN_CODE` in the communication area after the call, so any error signalled by `LGBATCHF` through that standard channel goes undetected

### 8.5 Program Termination

- The program terminates normally via a `RETURN` statement at the main procedure level after `PROCESS_RECORD` completes
  - There is no abnormal termination path (no `STOP RUN` equivalent, no explicit abend call, no condition signal) triggered by any error state; the program always exits cleanly from the perspective of the PL/I runtime, even when Db2 or subroutine errors have occurred

## 9. Examples

### 9.1 Example 1: Customer 1 found in DB2

**Input data**

CUSTOMER table row where `CUSTOMERNUMBER = 1`:

| Column | Value |
|---|---|
| FIRSTNAME | `Andrew` |
| LASTNAME | `Pandy` |
| DATEOFBIRTH | `1965-09-14` |
| HOUSENAME | `Pandy Mansion` |
| HOUSENUMBER | `1` |
| POSTCODE | `PO1 1AB` |

The program takes no run-time input. It sets `DB2_CUSTOMERNUMBER_INT = 1` itself.

**Expected output (SYSOUT)**

```
LGBATDB2 SQLCode : 0
LGBATDB2 SQLState: 00000
LGBATCHF WAS HERE... Customer 0000000001 is Andrew Pandy
Customer 0000000001 is Andrew Pandy
```

**Explanation**

- The first two `DISPLAY` lines run before any SQL statement, so they show the initial SQLCA values (SQLCODE 0, SQLSTATE `00000`).
- `GET_CUSTOMER_INFO` runs the `SELECT ... INTO` against `CUSTOMER`. It loads the row into the commarea fields `CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_DOB`, `CA_HOUSE_NAME`, `CA_POSTCODE` and the others.
- `CUSTIN_CUSTNUMBER` is a 10-digit picture field. Moving the integer 1 into it gives `0000000001`.
- `LGBATCHF` receives the number, the commarea pointer and the 80-character line. It keeps each name up to the first blank and builds `Customer 0000000001 is Andrew Pandy`.
- `LGBATCHF` displays the line with its own `LGBATCHF WAS HERE...` prefix. `LGBATDB2` then displays `CUSTRPT_LINE`, padded to 80 characters.

### 9.2 Example 2: Names containing blanks

**Input data**

CUSTOMER row where `CUSTOMERNUMBER = 1`:

| Column | Value |
|---|---|
| FIRSTNAME | `Mary Ann` |
| LASTNAME | `Van Der Berg` |

**Expected output (report line)**

```
Customer 0000000001 is Mary Van
```

**Explanation**

- `INDEX(...,' ')` finds the first blank in each name field. Only the text before that blank goes into the report line.
- `Mary Ann` becomes `Mary`, and `Van Der Berg` becomes `Van`.
- The line is built as `'Customer ' || number || ' is ' || first || ' ' || last`.

### 9.3 Example 3: Customer 1 not in the table

**Input data**

The `CUSTOMER` table has no row with `CUSTOMERNUMBER = 1`.

**Expected output (report line, approximate)**

```
Customer 0000000001 is
```

The exact text depends on what is left in the commarea.

**Explanation**

- The `SELECT` returns SQLCODE +100 (no row found). The program does not check SQLCODE after the select and does not display it.
- The `INTO` host variables are not loaded. `BATCH_COMM_AREA` is not initialized in this program, so the name fields hold whatever was already in that storage.
- The program still calls `LGBATCHF` and displays the line as usual. In the best case the names are blank and the line is `Customer 0000000001 is`, with an empty first name and an empty last name.

---

Generated by IBM Bob Premium Package for Z
