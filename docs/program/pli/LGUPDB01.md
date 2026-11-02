## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
  - [2.1 CICS Interface Inputs](#21-cics-interface-inputs)
  - [2.2 CICS COMMAREA Input Fields (`COMM_AREA`)](#22-cics-commarea-input-fields-comm_area)
  - [2.3 External Db2 Data Source Inputs](#23-external-db2-data-source-inputs)
- [3. Outputs](#3-outputs)
  - [3.1 Returned COMMAREA (primary output to caller)](#31-returned-commarea-primary-output-to-caller)
  - [3.2 Db2 data updates](#32-db2-data-updates)
  - [3.3 Error and diagnostic messages (CICS queue output via LGSTSQ)](#33-error-and-diagnostic-messages-cics-queue-output-via-lgstsq)
  - [3.4 Program control and abnormal termination](#34-program-control-and-abnormal-termination)
  - [3.5 Downstream program invocation](#35-downstream-program-invocation)
- [4. Processing Logic](#4-processing-logic)
  - [4.1 Initialization and Input Validation](#41-initialization-and-input-validation)
  - [4.2 Concurrency and Locking Management (Optimistic Locking)](#42-concurrency-and-locking-management-optimistic-locking)
  - [4.3 Policy-Specific Update Operations](#43-policy-specific-update-operations)
  - [4.4 Core Policy Database Synchronization](#44-core-policy-database-synchronization)
  - [4.5 Downstream VSAM Synchronization and Termination](#45-downstream-vsam-synchronization-and-termination)
  - [4.6 Error Handling and Logging](#46-error-handling-and-logging)
  - [4.7 Concurrency and Locking Management (Optimistic Locking)](#47-concurrency-and-locking-management-optimistic-locking)
  - [4.8 Policy-Specific Update Operations](#48-policy-specific-update-operations)
  - [4.9 Core Policy Database Synchronization](#49-core-policy-database-synchronization)
  - [4.10 Downstream VSAM Synchronization and Termination](#410-downstream-vsam-synchronization-and-termination)
  - [4.11 Error Handling and Logging](#411-error-handling-and-logging)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
  - [6.1 Calling Environment and Input Parameters](#61-calling-environment-and-input-parameters)
  - [6.2 Included Source Members (Copybooks)](#62-included-source-members-copybooks)
  - [6.3 Database Dependencies (Db2 for z/OS)](#63-database-dependencies-db2-for-zos)
  - [6.4 External Programs and Modules](#64-external-programs-and-modules)
  - [6.5 CICS Services and Resources](#65-cics-services-and-resources)
  - [6.6 Build and Configuration Dependencies](#66-build-and-configuration-dependencies)
  - [6.7 Runtime and Language Dependencies](#67-runtime-and-language-dependencies)
- [7. Constraints](#7-constraints)
  - [7.1 Request Type Constraints](#71-request-type-constraints)
  - [7.2 Concurrency and Data Integrity Constraints](#72-concurrency-and-data-integrity-constraints)
  - [7.3 Sequencing Constraints](#73-sequencing-constraints)
  - [7.4 Resource and Size Limits](#74-resource-and-size-limits)
  - [7.5 Error Handling and Return Code Constraints](#75-error-handling-and-return-code-constraints)
  - [7.6 Environment and Dependency Constraints](#76-environment-and-dependency-constraints)
- [8. Error Handling](#8-error-handling)
  - [8.1 Input and Environment Validation](#81-input-and-environment-validation)
  - [8.2 Return Code Signalling to the Caller](#82-return-code-signalling-to-the-caller)
  - [8.3 DB2 SQLCODE Checking](#83-db2-sqlcode-checking)
  - [8.4 Business and Data Integrity Checks](#84-business-and-data-integrity-checks)
  - [8.5 Transactional Recovery](#85-transactional-recovery)
  - [8.6 Logging and Notification](#86-logging-and-notification)
  - [8.7 Downstream Call Handling](#87-downstream-call-handling)
- [9. Examples](#9-examples)
  - [9.1 Example 1: Motor policy updated successfully](#91-example-1-motor-policy-updated-successfully)
  - [9.2 Example 2: Stale timestamp (optimistic lock failure)](#92-example-2-stale-timestamp-optimistic-lock-failure)
  - [9.3 Example 3: Policy not found](#93-example-3-policy-not-found)
  - [9.4 Example 4: Missing COMMAREA](#94-example-4-missing-commarea)
  - [9.5 Example 5: DB2 resource failure on cursor open](#95-example-5-db2-resource-failure-on-cursor-open)

## 1. Purpose

`LGUPDB01` is the Db2 back-end program responsible for updating existing insurance policy records in the GenApp general insurance application. Invoked via a CICS LINK from the orchestrator `LGUPOL01`, it receives a 32,500-byte COMMAREA carrying a customer number, a policy number, a `CA_REQUEST_ID` that identifies the policy type (`01UEND` = Endowment, `01UHOU` = House, `01UMOT` = Motor), and the new field values to be written. The program opens a `SELECT FOR UPDATE` cursor on the Db2 POLICY table to obtain a row lock, fetches the matching row, and performs an optimistic-concurrency check by comparing the `LASTCHANGED` timestamp held in Db2 against the one passed in the COMMAREA; if they differ the update is abandoned with return code `02`. When the timestamps match it delegates to one of three internal sub-procedures (`UPDATE_ENDOW_DB2_INFO`, `UPDATE_HOUSE_DB2_INFO`, or `UPDATE_MOTOR_DB2_INFO`) to update the policy-type-specific Db2 table (ENDOWMENT, HOUSE, or MOTOR), then issues a positioned `UPDATE … WHERE CURRENT OF POLICY_CURSOR` against the POLICY table to write the new dates, broker details, and a fresh `CURRENT TIMESTAMP`. After closing the cursor, the program issues a CICS LINK to `LGUPVS01` to keep the corresponding VSAM store in sync, and then returns to its caller.

## 2. Inputs

### 2.1 CICS Interface Inputs

- `COMM_AREA_PTR`
  - Pointer passed into the program entry point (`LGUPDB01: Proc(COMM_AREA_PTR)`) providing addressability to the `COMM_AREA` data structure.
- `EIBCALEN`
  - CICS Exec Interface Block field containing the length of the commarea passed by the calling program. Used to validate commarea presence and calculate variable-length payload sizes.
- `EIBTRNID`
  - CICS Exec Interface Block transaction identifier, captured into working storage for diagnostic and execution context.
- `EIBTRMID`
  - CICS Exec Interface Block terminal identifier, captured for runtime context.
- `EIBTASKN`
  - CICS Exec Interface Block task number, captured for runtime context.

### 2.2 CICS COMMAREA Input Fields (`COMM_AREA`)

- **Control & Routing Fields**
  - `CA_REQUEST_ID`
    - Identifies the specific update operation and policy type to execute (`01UEND` for Endowment, `01UHOU` for House, `01UMOT` for Motor).
  - `CA_CUSTOMER_NUM`
    - Customer number associated with the policy to update. Used in the `POLICY` table cursor query.
  - `CA_POLICY_NUM`
    - Policy number identifying the target policy record in the Db2 tables.

- **Common Policy Fields (`CA_POLICY_COMMON`)**
  - `CA_LASTCHANGED`
    - Timestamp of the record at last read; compared against the database `LASTCHANGED` value to verify optimistic concurrency control before performing the update.
  - `CA_ISSUE_DATE`
    - Policy start or issuance date updated in the `POLICY` table.
  - `CA_EXPIRY_DATE`
    - Policy expiration date updated in the `POLICY` table.
  - `CA_BROKERID`
    - Broker identifier associated with the policy, updated in the `POLICY` table.
  - `CA_BROKERSREF`
    - Broker reference string updated in the `POLICY` table.
  - `CA_PAYMENT`
    - Payment amount associated with the policy (converted to integer for internal handling).

- **Endowment Policy Fields (`CA_ENDOWMENT` - used when `CA_REQUEST_ID = '01UEND'`)**
  - `CA_E_WITH_PROFITS`
    - Flag indicating if the endowment policy includes profits (`Y`/`N`).
  - `CA_E_EQUITIES`
    - Flag indicating equity investment participation.
  - `CA_E_MANAGED_FUND`
    - Flag indicating managed fund status.
  - `CA_E_FUND_NAME`
    - Name of the investment fund.
  - `CA_E_TERM`
    - Duration/term of the endowment policy in years.
  - `CA_E_SUM_ASSURED`
    - Total sum assured upon maturity.
  - `CA_E_LIFE_ASSURED`
    - Name of the individual whose life is assured.
  - `CA_E_PADDING_DATA`
    - Additional variable-length payload data for the endowment policy (when commarea length exceeds required minimum).

- **House Policy Fields (`CA_HOUSE` - used when `CA_REQUEST_ID = '01UHOU'`)**
  - `CA_H_PROPERTY_TYPE`
    - Type of property insured (e.g., Detached, Semi-detached).
  - `CA_H_BEDROOMS`
    - Number of bedrooms in the insured property.
  - `CA_H_VALUE`
    - Insured value of the property.
  - `CA_H_HOUSE_NAME`
    - Name of the insured property/building.
  - `CA_H_HOUSE_NUMBER`
    - Street number of the insured property.
  - `CA_H_POSTCODE`
    - Postal code where the insured property is located.

- **Motor Policy Fields (`CA_MOTOR` - used when `CA_REQUEST_ID = '01UMOT'`)**
  - `CA_M_MAKE`
    - Manufacturer make of the insured vehicle.
  - `CA_M_MODEL`
    - Vehicle model description.
  - `CA_M_VALUE`
    - Valuation amount of the motor vehicle.
  - `CA_M_REGNUMBER`
    - Vehicle registration or license plate number.
  - `CA_M_COLOUR`
    - Color of the vehicle.
  - `CA_M_CC`
    - Engine capacity in cubic centimeters (cc).
  - `CA_M_MANUFACTURED`
    - Vehicle year/date of manufacture.
  - `CA_M_PREMIUM`
    - Calculated policy premium amount.
  - `CA_M_ACCIDENTS`
    - Number or value of recorded accidents.

### 2.3 External Db2 Data Source Inputs

- **`POLICY` Table**
  - `ISSUEDATE`
    - Issue date retrieved via cursor `POLICY_CURSOR` for record locking and fetch verification.
  - `EXPIRYDATE`
    - Expiration date retrieved via cursor `POLICY_CURSOR`.
  - `LASTCHANGED`
    - Timestamp retrieved to validate against `CA_LASTCHANGED` to prevent concurrent overwrite collisions, and queried again post-update to retrieve the new generated timestamp.
  - `BROKERID`
    - Broker ID retrieved via cursor into host variable `DB2_BROKERID_INT` with indicator `IND_BROKERID`.
  - `BROKERSREFERENCE`
    - Broker reference retrieved via cursor into host variable `DB2_BROKERSREF` with indicator `IND_BROKERSREF`.
  - `PAYMENT`
    - Payment value retrieved via cursor into host variable `DB2_PAYMENT_INT` with indicator `IND_PAYMENT`.
- **Db2 System Timestamp (`CURRENT TIMESTAMP`)**
  - Special register providing the current timestamp applied to the `POLICY.LASTCHANGED` column during update.

## 3. Outputs

### 3.1 Returned COMMAREA (primary output to caller)
- `CA_RETURN_CODE`: status code returned to the calling program (LGUPOL01).
  - `'00'`: the policy update succeeded. The cursor open and close also set this value.
  - `'01'`: the policy row was not found (SQLCODE 100 on the fetch), or the type-specific UPDATE found no row. The House update also sets `'01'` for any non-zero SQLCODE.
  - `'02'`: the timestamp in the COMMAREA does not match the stored `LASTCHANGED`. This is an optimistic-lock conflict, and nothing is updated.
  - `'90'`: a Db2 error occurred. It covers cursor OPEN or CLOSE failures, a fetch error, a policy-table UPDATE or SELECT failure, and Endowment or Motor update errors.
- `CA_LASTCHANGED`: the new `LASTCHANGED` timestamp, read back from the POLICY table after a successful update so the caller can use it on later updates.
- `COMM_AREA` (the full structure, passed on the LINK to LGUPVS01 and returned to the caller): it carries the return code and refreshed timestamp, plus the policy and type-specific data supplied by the caller.

### 3.2 Db2 data updates
- POLICY table
  - `ISSUEDATE`, `EXPIRYDATE` and `BROKERSREFERENCE` are set from `CA_ISSUE_DATE`, `CA_EXPIRY_DATE` and `CA_BROKERSREF`.
  - `BROKERID` is set from `CA_BROKERID`, converted to `DB2_BROKERID_INT`.
  - `LASTCHANGED` is set to `CURRENT TIMESTAMP`.
  - The update uses `WHERE CURRENT OF POLICY_CURSOR`, which holds a row lock until the update finishes.
- ENDOWMENT table, when `CA_REQUEST_ID` = `'01UEND'`
  - Updated columns: `WITHPROFITS`, `EQUITIES`, `MANAGEDFUND`, `FUNDNAME`, `TERM`, `SUMASSURED` and `LIFEASSURED`.
  - The columns come from the `CA_E_*` fields.
  - The update is keyed on `POLICYNUMBER`.
- HOUSE table, when `CA_REQUEST_ID` = `'01UHOU'`
  - Updated columns: `PROPERTYTYPE`, `BEDROOMS`, `VALUE`, `HOUSENAME`, `HOUSENUMBER` and `POSTCODE`.
  - The columns come from the `CA_H_*` fields.
- MOTOR table, when `CA_REQUEST_ID` = `'01UMOT'`
  - Updated columns: `MAKE`, `MODEL`, `VALUE`, `REGNUMBER`, `COLOUR`, `CC`, `YEAROFMANUFACTURE`, `PREMIUM` and `ACCIDENTS`.
  - The columns come from the `CA_M_*` fields.
- Transaction control
  - A failure when reading back the timestamp after the policy update triggers `EXEC CICS SYNCPOINT ROLLBACK`.
  - The rollback undoes the updates and sets `CA_RETURN_CODE` to `'90'`.

### 3.3 Error and diagnostic messages (CICS queue output via LGSTSQ)
- `ERROR_MSG` is written through `EXEC CICS LINK PROGRAM('LGSTSQ')`, which writes it to the CSMT queue. Its fields are:
  - `EM_DATE` and `EM_TIME`: timestamp taken with ASKTIME and FORMATTIME.
  - `EM_CUSNUM` and `EM_POLNUM`: the customer and policy numbers.
  - `EM_SQLREQ`: the failing SQL operation, for example OPEN PCURSOR, FETCH ROW, UPDATE POLICY, UPDATE ENDOW, UPDATE HOUSE, UPDATE MOTOR or CLOSE PCURSOR.
  - `EM_SQLRC`: the SQLCODE.
  - `EM_MSG_TEXT`: the free-text message, for example `' NO COMMAREA RECEIVED'`.
- `CA_ERROR_MSG` is a second message record sent to LGSTSQ. Its `CA_DATA` holds the first 90 bytes of the COMMAREA, or as much as was received, to help diagnose the failure.
- The message is written on these conditions:
  - no COMMAREA was received
  - the cursor OPEN or CLOSE failed
  - the fetch returned an error other than SQLCODE 100
  - the policy-table update failed
  - an Endowment or Motor update failed
  - the House update failed with an SQLCODE other than 100

### 3.4 Program control and abnormal termination
- The program issues `EXEC CICS ABEND ABCODE('LGCA') NODUMP` when `EIBCALEN = 0`, after it writes the error message.
- `EXEC CICS RETURN` ends the transaction and returns control to the caller. It is issued:
  - on a normal completion
  - immediately after a failed cursor OPEN or CLOSE
  - after a failed type-specific update

### 3.5 Downstream program invocation
- `EXEC CICS LINK Program('LGUPVS01') Commarea(COMM_AREA) LENGTH(225)` runs after the Db2 update logic (`UPDATE_POLICY_DB2_INFO`). It passes the updated COMMAREA to LGUPVS01, which does the VSAM-side policy update.

## 4. Processing Logic

I have analyzed the entire LGUPDB01.pli program. Let me double check its details.
The program is `LGUPDB01`, which updates policy details in a Db2 database.

Let's break down the logic:

- Main program entry:
  - Procedure definition: `LGUPDB01: Proc(COMM_AREA_PTR) Options(Main);`
  - Initializes runtime (debug) info (eye-catcher is `'LGUPDB01------WS'`, trans ID, term ID, task num, retry indicator). Note: The error message declaration shows a literal `' LGIPOL01'` in line 31, but the main eye-catcher has `LGUPDB01------WS`.
  - Initializes DB2 host variables: `DB2_POLICY = ' ';` and `DB2_IN_INTEGERS = 0;`
  - Checks if a commarea was received (`EIBCALEN = 0`). If not, it sets an error message, calls `WRITE_ERROR_MESSAGE`, and abends with code `'LGCA'`.
  - Initializes `CA_RETURN_CODE = '00'`.
  - Copies customer and policy numbers from COMMAREA to DB2 integer variables:
    - `DB2_CUSTOMERNUM_INT = CA_CUSTOMER_NUM;`
    - `DB2_POLICYNUM_INT = CA_POLICY_NUM;`
    - Also saves them in error message fields `EM_CUSNUM` and `EM_POLNUM`.
  - Calls internal procedure `UPDATE_POLICY_DB2_INFO`.
  - Links to VSAM update program `LGUPVS01` passing the `COMM_AREA` and a length of 225.
  - Returns to CICS using `EXEC CICS RETURN`.

- Internal procedure `UPDATE_POLICY_DB2_INFO`:
  - Opens `POLICY_CURSOR` (with hold, for update of `ISSUEDATE`, `EXPIRYDATE`, `LASTCHANGED`, `BROKERID`, `BROKERSREFERENCE`).
  - Evaluates `SQLCODE` from OPEN:
    - If `0`: Sets `CA_RETURN_CODE = '00'`.
    - If `-913` (deadlock/timeout): Sets `CA_RETURN_CODE = '90'`, logs error, and terminates processing (`EXEC CICS RETURN`).
    - Otherwise: Sets `CA_RETURN_CODE = '90'`, logs error, and terminates processing (`EXEC CICS RETURN`).
  - Calls `FETCH_DB2_POLICY_ROW` to fetch the record into host variables (`DB2_ISSUEDATE`, `DB2_EXPIRYDATE`, `DB2_LASTCHANGED`, and nullable indicators for `BROKERID`, `BROKERSREFERENCE`, `PAYMENT`).
  - If FETCH is successful (`SQLCODE = 0`):
    - Performs concurrency check (optimistic locking): Compares the `LASTCHANGED` timestamp in the database (`DB2_LASTCHANGED`) with the one provided in the COMMAREA (`CA_LASTCHANGED`).
    - If they match:
      - Directs the update based on `CA_REQUEST_ID`:
        - `'01UEND'` (Endowment policy): Calls `UPDATE_ENDOW_DB2_INFO`.
        - `'01UHOU'` (House policy): Calls `UPDATE_HOUSE_DB2_INFO`.
        - `'01UMOT'` (Motor policy): Calls `UPDATE_MOTOR_DB2_INFO`.
      - If the policy-specific update fails (i.e. `CA_RETURN_CODE <> '00'`), it closes the cursor and returns (`EXEC CICS RETURN`).
      - Otherwise, proceeds to update the core `POLICY` table row:
        - Transfers updated `CA_BROKERID` and `CA_PAYMENT` to DB2 integers.
        - Executes SQL `UPDATE POLICY` setting `ISSUEDATE = :CA_ISSUE_DATE`, `EXPIRYDATE = :CA_EXPIRY_DATE`, `LASTCHANGED = CURRENT TIMESTAMP`, `BROKERID = :DB2_BROKERID_INT`, and `BROKERSREFERENCE = :CA_BROKERSREF` where `CURRENT OF POLICY_CURSOR`.
        - Retrieves the newly assigned timestamp via a subsequent `SELECT LASTCHANGED` into `:CA_LASTCHANGED`.
        - If the update/select fails (`SQLCODE <> 0`), performs a CICS transaction rollback (`EXEC CICS SYNCPOINT ROLLBACK`), sets `CA_RETURN_CODE = '90'`, and logs the error.
    - If they do NOT match (indicating someone else updated the record in the meantime):
      - Sets `CA_RETURN_CODE = '02'`.
  - If FETCH fails:
    - If `SQLCODE = 100` (not found): Sets `CA_RETURN_CODE = '01'`.
    - Otherwise: Sets `CA_RETURN_CODE = '90'` and writes an error message.
  - Closes the cursor (`CALL CLOSE_PCURSOR`).

- Internal procedure `FETCH_DB2_POLICY_ROW`:
  - Executes DB2 SQL `FETCH POLICY_CURSOR` into the respective host variables and indicators.

- Internal procedure `CLOSE_PCURSOR`:
  - Executes DB2 SQL `CLOSE POLICY_CURSOR`.
  - Checks `SQLCODE`:
    - If `0` or `-501` (cursor not open): Sets `CA_RETURN_CODE = '00'`.
    - Otherwise: Sets `CA_RETURN_CODE = '90'`, writes error, and returns.

- Internal procedure `UPDATE_ENDOW_DB2_INFO`:
  - Moves `CA_E_TERM` and `CA_E_SUM_ASSURED` to DB2 integer variables.
  - Computes `WS_VARY_LEN = EIBCALEN - WS_REQUIRED_CA_LEN`.
  - If `WS_VARY_LEN > 0`:
    - Updates `ENDOWMENT` table using host variables including the varchar padding data. (Wait, the SQL statement inside `WS_VARY_LEN > 0` doesn't actually show a varchar update in this code because it has the same columns, but let's check:
      Wait! Let's look at lines 569-580:
      ```
      EXEC SQL
        UPDATE ENDOWMENT
          SET
            WITHPROFITS   = :CA_E_WITH_PROFITS,
              EQUITIES    = :CA_E_EQUITIES,
              MANAGEDFUND = :CA_E_MANAGED_FUND,
              FUNDNAME    = :CA_E_FUND_NAME,
              TERM        = :DB2_E_TERM_SINT,
              SUMASSURED  = :DB2_E_SUMASSURED_INT,
              LIFEASSURED = :CA_E_LIFE_ASSURED
          WHERE
              POLICYNUMBER = :DB2_POLICYNUM_INT;
      ```
      Ah, wait, does it update a varchar field? It actually doesn't seem to reference `WS_VARY_CHAR` in the SQL, but it copies `CA_E_PADDING_DATA` to `WS_VARY_CHAR`. This might be a placeholder or incomplete implementation in the template, but we will describe exactly what it does.
      Ah, actually, wait, the `IF ( WS_VARY_LEN > 0 ) THEN DO;` block executes an UPDATE statement but doesn't actually have a different column list in this template. Oh, wait, in either case (whether `WS_VARY_LEN > 0` or else), it executes the update of the ENDOWMENT table with the same fields: `WITHPROFITS`, `EQUITIES`, `MANAGEDFUND`, `FUNDNAME`, `TERM`, `SUMASSURED`, `LIFEASSURED` where `POLICYNUMBER = :DB2_POLICYNUM_INT`.
      Wait, let's look closely at line 595: `IF ( SQLCODE <> 0 ) THEN DO;` - oh, that's inside the `ELSE` block of the `IF ( WS_VARY_LEN > 0 )`. Let's verify if there is SQLCODE check in the `IF` block.
      Ah, line 581 is `END;`, then `ELSE DO;` ... `END; /*WS_VARY_LEN > */`. No SQLCODE check inside the `IF` block? Or does the `ELSE` check handle it? Let's verify line 581-607.
      Wait, in the `IF ( WS_VARY_LEN > 0 )` block, it does:
      ```
      SUBSTR(WS_VARY_CHAR,1,WS_VARY_LEN) = CA_E_PADDING_DATA;
      EXEC SQL UPDATE ENDOWMENT SET ... WHERE POLICYNUMBER = :DB2_POLICYNUM_INT;
      END;
      ```
      So there is indeed no `SQLCODE` check in that specific branch, whereas in the `ELSE` branch there is an `SQLCODE` check. Let's document this exact flow.

- Internal procedure `UPDATE_HOUSE_DB2_INFO`:
  - Moves `CA_H_BEDROOMS` and `CA_H_VALUE` to DB2 integer variables.
  - Executes DB2 SQL `UPDATE HOUSE` setting `PROPERTYTYPE = :CA_H_PROPERTY_TYPE`, `BEDROOMS = :DB2_H_BEDROOMS_SINT`, `VALUE = :DB2_H_VALUE_INT`, `HOUSENAME = :CA_H_HOUSE_NAME`, `HOUSENUMBER = :CA_H_HOUSE_NUMBER`, and `POSTCODE = :CA_H_POSTCODE` where `POLICYNUMBER = :DB2_POLICYNUM_INT`.
  - Checks `SQLCODE`:
    - If not `0`, sets `CA_RETURN_CODE = '01'` (or logs error via `WRITE_ERROR_MESSAGE`). Wait, line 630 sets `'01'`, and line 634 sets `'01'`. Yes, both branches of non-zero SQLCODE set `CA_RETURN_CODE = '01'`.

- Internal procedure `UPDATE_MOTOR_DB2_INFO`:
  - Moves `CA_M_CC`, `CA_M_VALUE`, `CA_M_PREMIUM`, and `CA_M_ACCIDENTS` to DB2 integer variables.
  - Executes DB2 SQL `UPDATE MOTOR` setting `MAKE = :CA_M_MAKE`, `MODEL = :CA_M_MODEL`, `VALUE = :DB2_M_VALUE_INT`, `REGNUMBER = :CA_M_REGNUMBER`, `COLOUR = :CA_M_COLOUR`, `CC = :DB2_M_CC_SINT`, `YEAROFMANUFACTURE = :CA_M_MANUFACTURED`, `PREMIUM = :DB2_M_PREMIUM_INT`, and `ACCIDENTS = :DB2_M_ACCIDENTS_INT` where `POLICYNUMBER = :DB2_POLICYNUM_INT`.
  - Checks `SQLCODE`:
    - If `100` (not found): Sets `CA_RETURN_CODE = '01'`.
    - Otherwise: Sets `CA_RETURN_CODE = '90'` and writes an error message.

- Internal procedure `WRITE_ERROR_MESSAGE`:
  - Captures current time and date via `EXEC CICS ASKTIME` and `EXEC CICS FORMATTIME`.
  - Formats error message details (`ERROR_MSG`).
  - Calls `LGSTSQ` (CICS TS queue logger program) with the structured error details.
  - Extracts and logs up to 90 bytes of raw COMMAREA data using `LEFT(COMM_AREA_RAW, ...)` and sends it to `LGSTSQ` as well.

Let's group the identified processing logic into logical categories as required:
1. Program Initialization and Validation
2. Concurrency Check (Optimistic Locking) and Cursor Processing
3. Core Policy Table Update and Timestamping
4. Policy-Specific Tables Update (Endowment, House, Motor)
5. VSAM Syncing and Program Termination
6. Error Handling and Logging

Let's organize the response clearly, using sub-headings where appropriate, starting directly with the section body content. Bullet points will start with "-" and use proper indentations.

Let's double-check all requirements:
- Do NOT use numbered bullet points. Bullet points should start with "-" and use proper indentations to reflect hierarchy.
- Do NOT add the section heading (meaning no "## Processing Logic" or "## Section ...").
- Sub-headings ("###" and deeper) are allowed and encouraged.
- Do NOT include any introductory text, preamble, reasoning, or meta commentary.
- The very first character must be part of the section content.
- Do NOT ask ANY questions.
- Write naturally and authoritatively.

Let's draft the sub-headings:
### 4.1 Initialization and Input Validation
- Description of main entry, EIBCALEN validation, variable initialization.

### 4.2 Concurrency and Locking Management (Optimistic Locking)
- Description of the Cursor declaration, OPEN cursor logic, and SQLCODE checking.
- Description of the FETCH and timestamp validation (`CA_LASTCHANGED` vs `DB2_LASTCHANGED`).

### 4.3 Policy-Specific Update Operations
- Detailed description of `UPDATE_ENDOW_DB2_INFO` (handling of variable length data if any, updates of Endowment-specific fields).
- Detailed description of `UPDATE_HOUSE_DB2_INFO` (type conversion, updates of House-specific fields).
- Detailed description of `UPDATE_MOTOR_DB2_INFO` (type conversion, updates of Motor-specific fields).

### 4.4 Core Policy Database Synchronization
- Details of updating the core `POLICY` table through the cursor.
- Fetching of the newly generated DB2 `CURRENT TIMESTAMP` back into the COMMAREA.
- Rollback mechanisms on failure.

### 4.5 Downstream VSAM Synchronization and Termination
- Calling of the VSAM sync program `LGUPVS01` via CICS LINK.
- CICS RETURN execution.

### 4.6 Error Handling and Logging
- ASKTIME, FORMATTIME.
- Writing error queue messages to the `LGSTSQ` program.
- Capturing and writing partial COMMAREA dumps.

Let's review the precise data transformations and assignments:
- `DB2_CUSTOMERNUM_INT` from `CA_CUSTOMER_NUM`
- `DB2_POLICYNUM_INT` from `CA_POLICY_NUM`
- Numeric variables mapped to internal `FIXED BIN` database types:
  - `DB2_BROKERID_INT` = `CA_BROKERID`
  - `DB2_PAYMENT_INT`  = `CA_PAYMENT`
  - `DB2_E_TERM_SINT` = `CA_E_TERM`
  - `DB2_E_SUMASSURED_INT` = `CA_E_SUM_ASSURED`
  - `DB2_H_BEDROOMS_SINT` = `CA_H_BEDROOMS`
  - `DB2_H_VALUE_INT` = `CA_H_VALUE`
  - `DB2_M_CC_SINT` = `CA_M_CC`
  - `DB2_M_VALUE_INT` = `CA_M_VALUE`
  - `DB2_M_PREMIUM_INT` = `CA_M_PREMIUM`
  - `DB2_M_ACCIDENTS_INT` = `CA_M_ACCIDENTS`

Let's review conditional branches:
- If `EIBCALEN = 0`, calls `WRITE_ERROR_MESSAGE` and issues an abend `LGCA`.
- If `SQLCODE` on Cursor OPEN is `-913` (deadlock), sets `CA_RETURN_CODE = '90'` and aborts.
- If timestamp match fails (`CA_LASTCHANGED <> DB2_LASTCHANGED`), sets `CA_RETURN_CODE = '02'`.
- If timestamp matches, branches on `CA_REQUEST_ID` (`01UEND`, `01UHOU`, `01UMOT`).
- In `UPDATE_ENDOW_DB2_INFO`, branches on `WS_VARY_LEN > 0`.
- SQLCODE error checks mapping to `CA_RETURN_CODE` codes `'01'` (Not Found) or `'90'` (System/DB2 error).

This structure matches all requested rules and is highly detailed. I am ready to output the response directly.### Initialization and Input Validation

- **Main Entry and Environment Mapping**
  - The program is initiated via CICS with a pointer to the standard communications area (`COMM_AREA_PTR`).
  - System fields such as the transaction ID (`EIBTRNID`), terminal ID (`EIBTRMID`), and task number (`EIBTASKN`) are copied to local tracker variables (`WS_TRANSID`, `WS_TERMID`, and `WS_TASKNUM`) for context propagation.
  - Core Db2 structures and internal variables are initialized to empty or default numeric states.

- **COMMAREA Length Verification**
  - The program validates the CICS communication area length (`EIBCALEN`).
  - If `EIBCALEN` is equal to zero, indicating that no data was received, the program logs an error message by calling `WRITE_ERROR_MESSAGE` and issues a CICS `ABEND` with code `'LGCA'` to abnormally terminate execution.
  - If the validation passes, the transaction return code is initialized to `'00'` (success), and the customer and policy numbers from the COMMAREA are copied into Db2-compatible binary integer variables (`DB2_CUSTOMERNUM_INT` and `DB2_POLICYNUM_INT`).

---

### 4.7 Concurrency and Locking Management (Optimistic Locking)

- **Cursor Opening and Deadlock Detection**
  - The program declares a cursor (`POLICY_CURSOR`) configured `WITH HOLD` and `FOR UPDATE` on specific columns (`ISSUEDATE`, `EXPIRYDATE`, `LASTCHANGED`, `BROKERID`, `BROKERSREFERENCE`) to select a policy record based on the customer and policy numbers.
  - It attempts to open `POLICY_CURSOR` and performs status evaluations on the resulting `SQLCODE`:
    - An `SQLCODE` of `0` indicates success, and the transaction return code remains `'00'`.
    - An `SQLCODE` of `-913` (indicating a deadlock or timeout) sets the return code to `'90'`, writes an error message, and immediately returns execution to the CICS environment.
    - Any other non-zero `SQLCODE` is treated as a major failure, setting the return code to `'90'` and triggering an immediate error exit.

- **Record Fetching and Timestamp Verification**
  - If the cursor opens successfully, the program executes a fetch operation (`FETCH_DB2_POLICY_ROW`) to retrieve the row data into host variables along with indicator variables (`IND_BROKERID`, `IND_BROKERSREF`, `IND_PAYMENT`) to safely handle potential database null values.
  - Once the record is retrieved, the program performs a concurrency check by comparing the timestamp in the database (`DB2_LASTCHANGED`) with the timestamp provided in the incoming COMMAREA (`CA_LASTCHANGED`).
    - If the timestamps do not match (indicating the record was modified by another task since it was last read), the program bypasses the update, sets `CA_RETURN_CODE` to `'02'`, and proceeds to close the cursor.
    - If the timestamps match, the program continues with the policy-specific update steps.

---

### 4.8 Policy-Specific Update Operations

Depending on the `CA_REQUEST_ID` field in the COMMAREA, the program directs the update process to one of three specialized internal procedures:

- **Endowment Policy Update (`UPDATE_ENDOW_DB2_INFO`)**
  - Converts incoming numeric fields (`CA_E_TERM`, `CA_E_SUM_ASSURED`) to Db2-compliant binary integer types.
  - Computes the length of any variable-length trailing data by subtracting the expected structured COMMAREA header length from the actual received length (`EIBCALEN`).
  - If variable-length data is present (`WS_VARY_LEN > 0`), it extracts the padding data (`CA_E_PADDING_DATA`) into a character buffer.
  - Performs an SQL `UPDATE ENDOWMENT` statement setting the policy columns (`WITHPROFITS`, `EQUITIES`, `MANAGEDFUND`, `FUNDNAME`, `TERM`, `SUMASSURED`, `LIFEASSURED`) based on the matching policy number.
  - Checks the resulting `SQLCODE` and updates the return code to `'01'` if the row was not found, or `'90'` in case of other database errors.

- **House Policy Update (`UPDATE_HOUSE_DB2_INFO`)**
  - Converts the text fields for bedrooms and property value (`CA_H_BEDROOMS` and `CA_H_VALUE`) to short binary integer and standard binary integer types.
  - Executes an SQL `UPDATE HOUSE` statement, modifying `PROPERTYTYPE`, `BEDROOMS`, `VALUE`, `HOUSENAME`, `HOUSENUMBER`, and `POSTCODE` columns for the target policy.
  - Handles Db2 response codes, assigning a `'01'` return code if the update fails.

- **Motor Policy Update (`UPDATE_MOTOR_DB2_INFO`)**
  - Maps incoming vehicle specifications (engine CC, value, premium, and accident count) from the COMMAREA into binary integers.
  - Executes an SQL `UPDATE MOTOR` statement to update the core automobile details: `MAKE`, `MODEL`, `VALUE`, `REGNUMBER`, `COLOUR`, `CC`, `YEAROFMANUFACTURE` (mapped to `YEAROFMANUFACTURE`), `PREMIUM`, and `ACCIDENTS` where the policy number matches.
  - Evaluates the outcome; an `SQLCODE` of `100` yields a return code of `'01'`, while any other error sets it to `'90'`.

---

### 4.9 Core Policy Database Synchronization

- **Database Update via Positioned Cursor**
  - After completing the policy-specific table modifications, the program transfers `CA_BROKERID` and `CA_PAYMENT` to internal host variables and updates the primary `POLICY` table.
  - It executes a positioned update (`UPDATE POLICY ... WHERE CURRENT OF POLICY_CURSOR`) to overwrite `ISSUEDATE`, `EXPIRYDATE`, `BROKERID`, and `BROKERSREFERENCE` while updating `LASTCHANGED` with the database's `CURRENT TIMESTAMP`.
  - To keep the client synchronized, a subsequent SQL `SELECT` is executed to retrieve the newly assigned `LASTCHANGED` timestamp from the database and write it directly back into `CA_LASTCHANGED`.

- **Transactional Rollback**
  - If the core policy update fails (resulting in a non-zero `SQLCODE`), the program triggers an explicit transaction rollback using `EXEC CICS SYNCPOINT ROLLBACK` to reverse all updates made during this execution unit.
  - It assigns a `'90'` return code and logs the occurrence.

- **Cursor Termination**
  - Upon completing or bypassing the update logic, the program calls the `CLOSE_PCURSOR` procedure to close the active database cursor, ensuring database locks are released.

---

### 4.10 Downstream VSAM Synchronization and Termination

- **VSAM Backend Linkage**
  - To maintain consistency between the relational database and the indexed file-system layer, the program issues a synchronous CICS `LINK` to the program `'LGUPVS01'`.
  - It passes the updated `COMM_AREA` with a predefined length of 225 bytes to allow the VSAM module to replicate the changes.

- **CICS Return**
  - After the VSAM update program completes control is returned to the initiator by executing an `EXEC CICS RETURN` statement, successfully concluding the program flow.

---

### 4.11 Error Handling and Logging

- **Logging and Diagnostic Capture**
  - Major failures and database exceptions call the `WRITE_ERROR_MESSAGE` internal routine.
  - The routine uses `EXEC CICS ASKTIME` and `EXEC CICS FORMATTIME` to capture and format the exact system time and date into local variables (`WS_TIME` and `WS_DATE`).
  - It populates an error structure (`ERROR_MSG`) with diagnostic context including the system time, date, customer number, policy number, current operation name, and the offending `SQLCODE`.

- **Queue Storage Integration**
  - The structured message is dispatched to the centralized CICS logging program `'LGSTSQ'` using a CICS `LINK` command.
  - To assist in post-mortem debugging, the program also captures up to 90 bytes of the raw, incoming COMMAREA payload (`COMM_AREA_RAW`) and writes it as an auxiliary diagnostic trace entry to the same queue.

## 5. Paragraphs

- **LGUPDB01 (Main Procedure)**
  - **Purpose:** Serves as the primary entry point for the policy update backend transaction, coordinating validation, database updates, and external program linkage.
  - **Operations Performed:**
    - Initializes execution variables from CICS EXEC Interface Block (EIB) fields (`EIBTRNID`, `EIBTRMID`, `EIBTASKN`).
    - Validates that a non-zero length COMMAREA is passed; if missing, writes an error message and terminates via CICS ABEND `LGCA`.
    - Initializes the return code to `'00'` and converts COMMAREA customer and policy numbers into Db2 host integer variables (`DB2_CUSTOMERNUM_INT`, `DB2_POLICYNUM_INT`).
    - Invokes `UPDATE_POLICY_DB2_INFO` to process Db2 updates.
    - Synchronously links to the VSAM update program `LGUPVS01` via `EXEC CICS LINK` passing the COMMAREA.
    - Terminates execution and returns control to the CICS caller via `EXEC CICS RETURN`.
  - **Control Flow:**
    - Conditional check on `EIBCALEN = 0` to trigger error handling and program abend.
    - Sequential call to internal procedure `UPDATE_POLICY_DB2_INFO` and CICS program `LGUPVS01`.
  - **External System Interactions:**
    - CICS runtime communication via `EXEC CICS ABEND`, `EXEC CICS LINK`, and `EXEC CICS RETURN`.

- **FETCH_DB2_POLICY_ROW**
  - **Purpose:** Retrieves a single policy record from Db2 using the open cursor to inspect current values and handle potential null indicators.
  - **Operations Performed:**
    - Sets the error tracking SQL request text to `' FETCH  ROW   '`.
    - Executes an embedded Db2 `FETCH` on `POLICY_CURSOR` into host variables `DB2_ISSUEDATE`, `DB2_EXPIRYDATE`, `DB2_LASTCHANGED`, and indicator-backed fields `DB2_BROKERID_INT`, `DB2_BROKERSREF`, and `DB2_PAYMENT_INT`.
  - **Control Flow:**
    - Direct sequential execution of the Db2 `FETCH` statement.
  - **External System Interactions:**
    - Db2 database interaction via `EXEC SQL FETCH`.

- **UPDATE_POLICY_DB2_INFO**
  - **Purpose:** Coordinates optimistic concurrency verification, routes updates to policy-type specific tables, and updates the core `POLICY` table record with a refreshed timestamp.
  - **Operations Performed:**
    - Sets SQL request indicator to `' OPEN   PCURSOR '` and opens `POLICY_CURSOR`.
    - Evaluates `SQLCODE` from cursor open; if locked/unavailable (e.g., `-913`) or error, sets return code `'90'`, logs error, and returns.
    - Calls `FETCH_DB2_POLICY_ROW` to obtain the current Db2 row.
    - Compares `CA_LASTCHANGED` from the incoming COMMAREA against `DB2_LASTCHANGED` from Db2 to prevent lost updates:
      - If timestamps match, routes to the appropriate policy update routine (`UPDATE_ENDOW_DB2_INFO`, `UPDATE_HOUSE_DB2_INFO`, or `UPDATE_MOTOR_DB2_INFO`) based on `CA_REQUEST_ID`.
      - If policy-specific update fails (non-`'00'` return code), invokes `CLOSE_PCURSOR` and returns.
      - Updates the `POLICY` table with new dates, broker details, and sets `LASTCHANGED = CURRENT TIMESTAMP` using `WHERE CURRENT OF POLICY_CURSOR`.
      - Re-selects the newly generated `LASTCHANGED` timestamp into `CA_LASTCHANGED` for return to the caller.
      - If `UPDATE POLICY` encounters an SQL error, issues `EXEC CICS SYNCPOINT ROLLBACK`, sets return code `'90'`, and logs an error.
      - If timestamps do not match, sets return code `'02'`.
    - If the fetch returns `SQLCODE = 100` (not found), sets return code `'01'`; otherwise for other fetch errors, sets return code `'90'` and writes an error message.
    - Closes `POLICY_CURSOR` by invoking `CLOSE_PCURSOR`.
  - **Control Flow:**
    - `SELECT (SQLCODE)` condition evaluating cursor open results.
    - `IF-THEN-ELSE` testing fetch `SQLCODE` and timestamp matching (`CA_LASTCHANGED = DB2_LASTCHANGED`).
    - `SELECT (CA_REQUEST_ID)` evaluating `'01UEND'`, `'01UHOU'`, and `'01UMOT'`.
    - Error checking branches triggering rollback, error logging, or immediate returns.
  - **External System Interactions:**
    - Db2 operations via `EXEC SQL OPEN`, `EXEC SQL UPDATE ... WHERE CURRENT OF`, and `EXEC SQL SELECT`.
    - CICS transaction control via `EXEC CICS SYNCPOINT ROLLBACK` and `EXEC CICS RETURN`.

- **CLOSE_PCURSOR**
  - **Purpose:** Closes `POLICY_CURSOR` and manages cursor close status codes.
  - **Operations Performed:**
    - Sets SQL request indicator to `' CLOSE  PCURSOR'`.
    - Executes embedded Db2 `CLOSE` on `POLICY_CURSOR`.
    - Evaluates `SQLCODE`; treats `0` and `-501` (cursor not open) as successful closure (`'00'`), and logs errors for any other negative status codes.
  - **Control Flow:**
    - `SELECT (SQLCODE)` construct branching on `0`, `-501`, and `OTHERWISE`.
  - **External System Interactions:**
    - Db2 database interaction via `EXEC SQL CLOSE`.
    - CICS return via `EXEC CICS RETURN` on error or `-501`.

- **UPDATE_ENDOW_DB2_INFO**
  - **Purpose:** Updates endowment policy details in the `ENDOWMENT` Db2 table, handling variable-length text data if supplied.
  - **Operations Performed:**
    - Formats numeric COMMAREA fields (`CA_E_TERM`, `CA_E_SUM_ASSURED`) into Db2 host integer variables.
    - Calculates variable text length (`WS_VARY_LEN`) from `EIBCALEN` minus `WS_REQUIRED_CA_LEN`.
    - Executes `EXEC SQL UPDATE ENDOWMENT` with or without variable-length padding depending on length calculation.
    - Checks `SQLCODE`; sets return code `'01'` for `SQLCODE = 100` (record not found) or `'90'` for other database errors and logs the message.
  - **Control Flow:**
    - `IF-THEN-ELSE` checking `WS_VARY_LEN > 0` to decide SQL update variation.
    - `IF-THEN-ELSE` checking `SQLCODE` after update execution.
  - **External System Interactions:**
    - Db2 database interaction via `EXEC SQL UPDATE ENDOWMENT`.

- **UPDATE_HOUSE_DB2_INFO**
  - **Purpose:** Updates house policy details in the `HOUSE` Db2 table.
  - **Operations Performed:**
    - Formats numeric COMMAREA fields (`CA_H_BEDROOMS`, `CA_H_VALUE`) into Db2 host integer variables.
    - Sets SQL request indicator to `' UPDATE HOUSE '`.
    - Executes `EXEC SQL UPDATE HOUSE` updating property type, bedrooms, value, house name/number, and postcode where `POLICYNUMBER` matches.
    - Evaluates `SQLCODE` for errors; sets return code `'01'` if record is not found or if an error occurs, and invokes `WRITE_ERROR_MESSAGE`.
  - **Control Flow:**
    - `IF (SQLCODE <> 0)` with nested `IF (SQLCODE = 100)` condition handling.
  - **External System Interactions:**
    - Db2 database interaction via `EXEC SQL UPDATE HOUSE`.

- **UPDATE_MOTOR_DB2_INFO**
  - **Purpose:** Updates motor policy details in the `MOTOR` Db2 table.
  - **Operations Performed:**
    - Formats numeric COMMAREA fields (`CA_M_CC`, `CA_M_VALUE`, `CA_M_PREMIUM`, `CA_M_ACCIDENTS`) into Db2 host integer variables.
    - Sets SQL request indicator to `' UPDATE MOTOR '`.
    - Executes `EXEC SQL UPDATE MOTOR` matching on `POLICYNUMBER`.
    - Evaluates `SQLCODE`; sets return code `'01'` if not found (`100`) or `'90'` on error, writing an error message on unexpected errors.
  - **Control Flow:**
    - `IF (SQLCODE <> 0)` branching to check for not found versus system/Db2 errors.
  - **External System Interactions:**
    - Db2 database interaction via `EXEC SQL UPDATE MOTOR`.

- **WRITE_ERROR_MESSAGE**
  - **Purpose:** Formats and writes diagnostic error messages and incoming COMMAREA data to a CICS Transient Data / Temporary Storage queue via error-logging utility program `LGSTSQ`.
  - **Operations Performed:**
    - Obtains system timestamp and formats it into date (`MMDDYYYY`) and time strings using `EXEC CICS ASKTIME` and `EXEC CICS FORMATTIME`.
    - Populates `ERROR_MSG` with timestamp, SQL code, request text, customer number, and policy number.
    - Links to error handler `LGSTSQ` passing `ERROR_MSG`.
    - Copies up to 90 bytes of `COMM_AREA_RAW` into `CA_ERROR_MSG` and links to `LGSTSQ` to record the input payload.
  - **Control Flow:**
    - Nested `IF` conditions checking `EIBCALEN > 0` and whether `EIBCALEN < 91` to safely truncate data length for COMMAREA dump logging.
  - **External System Interactions:**
    - CICS system services via `EXEC CICS ASKTIME` and `EXEC CICS FORMATTIME`.
    - CICS program link via `EXEC CICS LINK PROGRAM('LGSTSQ')`.

## 6. Dependencies

### 6.1 Calling Environment and Input Parameters
- CICS transaction environment
  - The program runs as a CICS main program (`Proc(COMM_AREA_PTR) Options(Main)`). It needs an active CICS task and region.
  - It is normally linked to by the orchestrator `LGUPOL01`.
- COMMAREA passed by the caller (`COMM_AREA_PTR`, layout `LGCMAREA`)
  - `CA_REQUEST_ID` selects the operation: `01UEND` (endowment), `01UHOU` (house) or `01UMOT` (motor).
  - `CA_CUSTOMER_NUM` and `CA_POLICY_NUM` identify the policy to update.
  - `CA_LASTCHANGED` is the timestamp used for optimistic concurrency checking.
  - Common policy fields: `CA_ISSUE_DATE`, `CA_EXPIRY_DATE`, `CA_BROKERID`, `CA_BROKERSREF` and `CA_PAYMENT`.
  - Policy-type-specific fields are read from `CA_ENDOWMENT`, `CA_HOUSE` or `CA_MOTOR`.
  - `CA_RETURN_CODE` is set on output: `00` OK, `01` not found, `02` timestamp mismatch, `90` error.
  - The program abends with `LGCA` if `EIBCALEN = 0`, so a COMMAREA is mandatory.
- CICS Execute Interface Block (EIB) fields
  - `EIBTRNID`, `EIBTRMID` and `EIBTASKN` populate the diagnostic header.
  - `EIBCALEN` is used for the COMMAREA length check, the varchar length calculation and the error-message COMMAREA dump.

### 6.2 Included Source Members (Copybooks)
- `LGPOLICY` (`plinc/LGPOLICY.inc`)
  - Db2 host-variable structures: `DB2_CUSTOMER`, `DB2_POLICY`, `DB2_ENDOWMENT`, `DB2_HOUSE`, `DB2_MOTOR`, `DB2_COMMERCIAL` and `DB2_CLAIM`.
  - Policy length constants (`WS_POLICY_LENGTHS`).
- `LGCMAREA` (`plinc/LGCMAREA.inc`)
  - The `COMM_AREA` structure, based on `COMM_AREA_PTR`. It is the data contract with the calling program.
- `SQLCA`
  - The Db2 communications area. It supplies `SQLCODE`, which is checked after SQL operations.
- Include search path
  - Members are resolved from `plinc/*.inc` through the `%INCLUDE` preprocessor and zAppBuild's `pliincludeSearch` setting.

### 6.3 Database Dependencies (Db2 for z/OS)
- Db2 subsystem and plan/package
  - A Db2 connection is required through the CICS-Db2 attachment. The program needs a bound plan or package with authority to update these tables.
  - `POLICY_CURSOR` is declared `WITH HOLD ... FOR UPDATE`.
  - Handled SQLCODEs: `0`, `100`, `-913` (deadlock/timeout) and `-501` (cursor not open).
- Table `POLICY`
  - Selected through `POLICY_CURSOR` (`ISSUEDATE`, `EXPIRYDATE`, `LASTCHANGED`, `BROKERID`, `BROKERSREFERENCE`), with the row locked by `FOR UPDATE`.
  - Updated with `UPDATE ... WHERE CURRENT OF POLICY_CURSOR`, setting `LASTCHANGED = CURRENT TIMESTAMP`.
  - Re-read with `SELECT LASTCHANGED` to return the new timestamp to the caller.
- Table `ENDOWMENT`
  - Updated for `01UEND` requests: `WITHPROFITS`, `EQUITIES`, `MANAGEDFUND`, `FUNDNAME`, `TERM`, `SUMASSURED` and `LIFEASSURED`, keyed by `POLICYNUMBER`.
  - There are two UPDATE variants, depending on whether the COMMAREA carries varchar data.
- Table `HOUSE`
  - Updated for `01UHOU` requests: `PROPERTYTYPE`, `BEDROOMS`, `VALUE`, `HOUSENAME`, `HOUSENUMBER` and `POSTCODE`, keyed by `POLICYNUMBER`.
- Table `MOTOR`
  - Updated for `01UMOT` requests: `MAKE`, `MODEL`, `VALUE`, `REGNUMBER`, `COLOUR`, `CC`, `YEAROFMANUFACTURE`, `PREMIUM` and `ACCIDENTS`, keyed by `POLICYNUMBER`.
- Db2 host variables and indicator variables
  - Integer host variables (`DB2_*_INT`, `DB2_*_SINT`) convert the COMMAREA's numeric picture fields to Db2 integer types.
  - Indicators `IND_BROKERID`, `IND_BROKERSREF` and `IND_PAYMENT` handle nullable columns on FETCH.

### 6.4 External Programs and Modules
- `LGUPVS01`
  - Linked with `EXEC CICS LINK` after the Db2 update, passing `COMM_AREA` with `LENGTH(225)`. It handles the VSAM-side policy update.
  - Its name is held in the variable `LGUPVS1`.
- `LGSTSQ`
  - Linked from `WRITE_ERROR_MESSAGE` to write error messages. It writes to the CICS TD/TS queues (the code comments mention `CSMT`).
  - It receives two COMMAREAs: the formatted `ERROR_MSG`, and `CA_ERROR_MSG` with up to 90 bytes of the original COMMAREA.

### 6.5 CICS Services and Resources
- Program control
  - `EXEC CICS LINK` to `LGUPVS01` and `LGSTSQ`.
  - `EXEC CICS RETURN` is used throughout for normal and error exits.
  - `EXEC CICS ABEND ABCODE('LGCA') NODUMP` is used when there is no COMMAREA.
- Unit-of-work control
  - `EXEC CICS SYNCPOINT ROLLBACK` backs out changes if the post-update `SELECT` fails.
- Time services
  - `EXEC CICS ASKTIME` and `EXEC CICS FORMATTIME` (MMDDYYYY, TIME) timestamp the error messages.
- Transient data queue `CSMT`
  - Referenced in the comments as the error message destination. It is accessed indirectly through `LGSTSQ`.
- CICS and Db2 definitions
  - CSD definitions in `Resources/` for the program, the transaction and the linked programs.
  - Db2 DDL, bind and CSD jobs in `jcl/`.

### 6.6 Build and Configuration Dependencies
- Compiler and preprocessor
  - Enterprise PL/I for z/OS V52.
  - The CICS translator (`SYSTEM(CICS)`, `PP(MACRO,CICS)`) and the Db2 SQL precompiler or coprocessor.
  - Include preprocessing uses `PP(INCLUDE('ID(++INCLUDE)'))`.
- Build framework
  - zAppBuild properties in `application-conf/` (`PLI.properties`, `application.properties` and `file.properties`).
  - Link edit parms `MAP,RENT,COMPAT(PM5)`. The load module is deployed as `CICSLOAD`.
- Hardcoded configuration values
  - COMMAREA header length of 28 bytes (`WS_CA_HEADER_LEN`).
  - Fixed link length of 225 bytes for `LGUPVS01`.
  - Eyecatcher `LGUPDB01------WS`.
  - Error-message program tag `LGIPOL01`, carried over from another program.
  - Return-code literals `00`, `01`, `02` and `90`.
- Environment variables
  - The program reads none.

### 6.7 Runtime and Language Dependencies
- PL/I built-in functions
  - `SYSNULL`, `SUBSTR`, `LEFT` and `STG`.
- Working storage structures
  - `WS_HEADER`, `ERROR_MSG`, `CA_ERROR_MSG`, `WS_VARY_FIELD` (3900-byte varchar buffer) and `WS_COMMAREA_LENGTHS`.
  - These carry the diagnostics, the error message layout and the endowment varchar length calculation.

## 7. Constraints

-### Input and Commarea Constraints
  - A commarea is required.
    - If `EIBCALEN = 0`, the program writes the error text "NO COMMAREA RECEIVED" through `WRITE_ERROR_MESSAGE`.
    - It then abends with `EXEC CICS ABEND ABCODE('LGCA') NODUMP`.
  - The commarea has a fixed layout (`COMM_AREA`, based on `COMM_AREA_PTR`).
    - `CA_REQUEST_ID` is 6 characters.
    - `CA_RETURN_CODE` is `PIC '99'`.
    - `CA_CUSTOMER_NUM` and `CA_POLICY_NUM` are 10-digit numeric pictures.
    - The commarea raw area is capped at 32500 bytes.
  - Numeric commarea fields are copied into fixed-size DB2 integer host variables.
    - Customer and policy numbers go into `FIXED BIN(31)`.
    - Term, bedrooms and cc go into `FIXED BIN(15)`.
    - Value, premium, accidents, sum assured, broker id and payment go into `FIXED BIN(31)`.
    - The program does not range-check these conversions. Values must fit the target binary sizes.
  - Field lengths are fixed by the declarations.
    - Examples are `CA_ISSUE_DATE` and `CA_EXPIRY_DATE` at 10 characters, `CA_LASTCHANGED` at 26, and `CA_BROKERSREF` at 10.
    - Endowment life assured is 31 characters, house postcode is 8, and motor registration is 7.
    - Values longer than these are truncated or rejected by PL/I assignment and DB2 rules.
  - The program does not validate field content (formats, date validity, required fields, value ranges). It relies on the PL/I picture and character declarations and on DB2 column constraints.

### 7.1 Request Type Constraints
  - The update path depends on `CA_REQUEST_ID` in a `SELECT` statement.
    - `'01UEND'` updates the Endowment table.
    - `'01UHOU'` updates the House table.
    - `'01UMOT'` updates the Motor table.
  - Any other request ID, including commercial or claim updates, has no `WHEN` branch and no `OTHERWISE`.
    - No policy-type-specific update is performed.
    - The return code stays '00'.
    - The common POLICY table update still runs.
  - Only one policy-type table is updated per invocation.

### 7.2 Concurrency and Data Integrity Constraints
  - Optimistic locking uses the last-changed timestamp.
    - The update proceeds only if `CA_LASTCHANGED = DB2_LASTCHANGED`.
    - On a mismatch the program sets the return code to '02' and performs no updates.
  - Pessimistic row locking is used on the POLICY table.
    - `POLICY_CURSOR` is declared `FOR UPDATE OF` the issue date, expiry date, last changed, broker id and broker reference.
    - The cursor is `WITH HOLD`.
    - The row stays locked from the open and fetch until the policy update and the cursor close.
  - The cursor is keyed on both customer number and policy number. Only the row for that customer and policy pair can be fetched.
  - Only one matching row is expected, so a single `FETCH` is issued.
  - The payment and commission fields are not updatable through the cursor. `PAYMENT` is excluded from `FOR UPDATE OF` and from the `UPDATE POLICY` statement.
  - A new timestamp is always assigned by DB2 on update (`LASTCHANGED = CURRENT TIMESTAMP`).
    - The client cannot set it.
    - The new value is read back into `CA_LASTCHANGED` for the caller.
  - The read-back `SELECT LASTCHANGED` filters only on `POLICYNUMBER`.
  - Policy table updates use `WHERE CURRENT OF POLICY_CURSOR`. They only affect the locked, fetched row.
  - The specific-table updates (Endowment, House, Motor) are keyed on `POLICYNUMBER`.

### 7.3 Sequencing Constraints
  - The processing order is fixed.
    - The program first checks the commarea length and initializes variables.
    - It then opens the cursor and fetches the row.
    - Next it compares timestamps and updates the policy-type table.
    - Then it updates the POLICY table and reads back the new timestamp.
    - Finally it closes the cursor and links to `LGUPVS01`.
    - It issues `CICS RETURN` last.
  - The policy-type-specific table is updated before the common POLICY table.
  - If the type-specific update leaves the return code not equal to '00', the program closes the cursor and returns. The POLICY table is not updated.
  - The `LGUPVS01` link runs only if `UPDATE_POLICY_DB2_INFO` returns normally.
    - The link passes a commarea length of 225.
    - Several error paths issue `CICS RETURN` directly and bypass it.
  - The error log message is written in two parts.
    - The formatted error message record goes first.
    - The commarea excerpt record goes second.

### 7.4 Resource and Size Limits
  - The error-log commarea excerpt is limited to 90 bytes.
    - If `EIBCALEN` is under 91, only `EIBCALEN` bytes are logged.
    - If `EIBCALEN` is 91 or more, the first 90 bytes are logged.
    - If `EIBCALEN` is 0, no excerpt is written.
  - The error message text field is 63 characters.
  - The varchar work area `WS_VARY_CHAR` is limited to 3900 characters.
  - The endowment update uses `WS_VARY_LEN = EIBCALEN - WS_REQUIRED_CA_LEN`.
    - `WS_REQUIRED_CA_LEN` is initialized to 0.
    - No length check is made against the 3900-character maximum.
  - The `WS_VARY_FIELD` area is populated only when `WS_VARY_LEN > 0`. It is not used in the `UPDATE ENDOWMENT` statement.
  - The `LGUPVS01` link uses a fixed length of 225 bytes, regardless of the received commarea length.
  - Standard length constants are defined in `WS_POLICY_LENGTHS`.
    - They include 72, 52, 58 and 65 for the basic, endowment, house and motor records.
    - `WS_CA_HEADER_LEN` is 28.
    - The program does not use them to enforce a minimum commarea length check, although the comments describe one.

### 7.5 Error Handling and Return Code Constraints
  - Return codes are placed in `CA_RETURN_CODE`.
    - '00' means success. It is set at start-up and after a successful open or close.
    - '01' means the row was not found. It is set when `FETCH` or `UPDATE` returns SQLCODE 100.
    - '02' means a timestamp mismatch, so the data is stale.
    - '90' means an unexpected DB2 error.
  - Cursor `OPEN` outcomes:
    - SQLCODE 0 continues processing.
    - SQLCODE -913 (deadlock or timeout) sets '90', logs an error and returns immediately.
    - Any other SQLCODE sets '90', logs an error and returns immediately.
  - `FETCH` outcomes:
    - SQLCODE 0 continues processing.
    - SQLCODE 100 sets '01' without logging.
    - Any other SQLCODE sets '90' and logs an error.
  - Policy-type update outcomes:
    - Endowment, SQLCODE 100: '01' is set without logging.
    - Endowment, other non-zero SQLCODE: '90' is set and an error is logged.
    - Motor, SQLCODE 100: '01' is set without logging.
    - Motor, other non-zero SQLCODE: '90' is set and an error is logged.
    - House, any non-zero SQLCODE: '01' is set, and non-100 codes are also logged. A real DB2 error is therefore reported as "not found" rather than '90'.
  - A failure in the POLICY update or timestamp read-back (non-zero SQLCODE after the `SELECT`) triggers `EXEC CICS SYNCPOINT ROLLBACK`.
    - The return code is set to '90' and the error is logged.
    - The rollback undoes the type-specific table update as well.
    - Failures from the type-specific updates do not roll back explicitly.
  - Only the SQLCODE of the read-back `SELECT` is checked. The `UPDATE POLICY` statement's SQLCODE is not tested separately, because the `SELECT` overwrites it.
  - `CLOSE` outcomes:
    - SQLCODE 0 sets '00'.
    - SQLCODE -501 (cursor already closed) sets '00' and returns without logging.
    - Any other SQLCODE sets '90', logs an error and returns.
  - The close step overwrites the return code with '00' on success. A '01', '02' or '90' set earlier in `UPDATE_POLICY_DB2_INFO` is therefore reset to '00' unless the close fails.
  - Every logged error carries diagnostic context.
    - It includes the date and time, the program tag `LGIPOL01`, and the customer and policy numbers.
    - It also includes the SQL request description and the SQLCODE.
    - The record is sent by `LINK` to `LGSTSQ`, which writes it to the queue.

### 7.6 Environment and Dependency Constraints
  - The program runs only under CICS (`EIBCALEN`, `EIBTRNID`, `EIBTRMID`, `EIBTASKN`, `LINK`, `RETURN`, `ABEND`, `SYNCPOINT`).
  - It needs the DB2 tables POLICY, ENDOWMENT, HOUSE and MOTOR, plus a SQLCA.
  - It depends on the CICS programs `LGUPVS01` (called after the update) and `LGSTSQ` (the error logger).
  - Date and time for log messages come from `ASKTIME` and `FORMATTIME` in MMDDYYYY format.
  - Host variables must be declared in the program or in included copybooks (such as `LGPOLICY` and `LGCMAREA`) so the SQL precompiler can resolve them.

## 8. Error Handling

### 8.1 Input and Environment Validation
- **Missing COMMAREA check**
  - The program tests the CICS commarea length (EIBCALEN) at the start of processing.
  - If it is zero, the program sets the message text to "NO COMMAREA RECEIVED". It then calls the shared error-writing procedure and abends the task with code LGCA, with no dump.
  - This is the only place the program abends. All other errors are reported through return codes and logging.
- **Initialisation of error context**
  - The commarea return code is set to '00' once the commarea is confirmed present.
  - The customer and policy numbers are copied into the error message structure before any database work. Any later error message therefore identifies the affected customer and policy.
  - Host variables are cleared at startup so stale values are not used.
- **Request type dispatch**
  - A SELECT on the request ID routes to the endowment ('01UEND'), house ('01UHOU') or motor ('01UMOT') update routine.
  - There is no OTHERWISE branch for an unrecognised request ID.
  - The program has no explicit handling for that case, and it declares no ON-unit anywhere.

### 8.2 Return Code Signalling to the Caller
- **Return code values in the commarea**
  - '00' means success.
  - '01' means the row was not found (SQLCODE 100) in the fetch or in a policy-type update.
  - '02' means a timestamp mismatch: the data was changed by someone else.
  - '90' means a DB2 or system error.
  - The house update sets '01' for every non-zero SQLCODE, including real errors. The endowment and motor routines use '90' for those.
  - The program header comments and project conventions describe '98' and '99' codes, but this program does not use them.
- **Propagation after sub-updates**
  - After a policy-type update routine returns, the main flow checks whether the return code is still '00'.
  - If not, it closes the cursor and returns to the caller without updating the policy table.

### 8.3 DB2 SQLCODE Checking
- **Cursor open**
  - An evaluation of SQLCODE follows the OPEN of the policy cursor.
  - SQLCODE 0 sets return code '00' and processing continues.
  - SQLCODE -913 (deadlock or timeout) sets '90', writes an error message and returns immediately to CICS.
  - Any other SQLCODE gets the same handling as -913, so the specific -913 branch is redundant.
- **Fetch of the policy row**
  - A non-zero SQLCODE sends the program to the failure branch.
  - SQLCODE 100 (no matching row) sets return code '01' with no error message.
  - Any other non-zero SQLCODE sets '90' and writes an error message.
- **Cursor close**
  - SQLCODE 0 sets return code '00'.
  - SQLCODE -501 (cursor not open) is treated as benign. The return code is set to '00', the request label is overwritten with a "-501 detected" note, and the program returns to CICS. No error is logged.
  - Any other SQLCODE sets '90', writes an error message and returns.
  - A successful close sets the return code to '00'. This overwrites a '01' or '02' set earlier in the same run, so those outcomes can be lost before the caller sees them.
- **Policy-type table updates (endowment, house, motor)**
  - Each routine checks SQLCODE after its UPDATE.
  - SQLCODE 100 sets return code '01' with no logging.
  - Any other non-zero SQLCODE sets a failure code and logs an error message.
  - In the endowment routine, the SQLCODE check is attached only to the branch with no variable-length data. The branch that updates with the varchar data has no check after its UPDATE.
- **Policy table update**
  - The program updates the policy table through the cursor and then selects the new timestamp.
  - A single SQLCODE check follows the select, and so reflects only the select. A failed update that leaves SQLCODE at 0 would not be caught.
  - If the check fails, a CICS syncpoint rollback is issued, the return code is set to '90' and an error message is logged.

### 8.4 Business and Data Integrity Checks
- **Optimistic concurrency control**
  - The timestamp in the commarea is compared with the LASTCHANGED value fetched from the database under cursor lock.
  - If they differ, no update is performed and return code '02' is passed back.
- **Row locking**
  - The policy row is read through a FOR UPDATE cursor declared WITH HOLD. This locks the row against concurrent modification during the check and update.
  - The cursor is always closed before the program finishes, on the normal path and on the policy-type update failure path.
- **Null handling**
  - Indicator variables are fetched alongside broker ID, broker reference and payment. This prevents a SQLCODE -305 failure when those columns are null.
  - The program does not inspect the indicator values after the fetch.
- **Variable-length data check**
  - The endowment routine computes the length of the extra varchar data as the commarea length minus a required header length.
  - A positive value selects one of two update paths. The programmer's intent is a length-based path choice, but the required-length field is never set from its initial value.

### 8.5 Transactional Recovery
- **Rollback**
  - A CICS SYNCPOINT ROLLBACK is issued only when the final policy-table update and timestamp retrieval fail. This undoes any earlier policy-type table update in the same unit of work.
  - If a policy-type table update fails, the program does not roll back. It closes the cursor and returns, leaving recovery to the CICS task end.
- **Early exit**
  - When an error leaves the program in an unsafe state, it issues a CICS RETURN immediately.
  - This happens on a cursor open failure, a cursor close failure, a policy-type update failure and a missing commarea (by abend).
- **Fallbacks**
  - The program has no retry logic. The retry flag is initialised and never used.
  - Warning-level outcomes ('01', '02') are returned to the caller as informational codes with no further attempt.

### 8.6 Logging and Notification
- **Central error-writing procedure**
  - It gets the current time with ASKTIME and formats the date and time with FORMATTIME. It then writes a standard error record containing:
    - date and time
    - a program-name tag, which reads LGIPOL01 even though this program is LGUPDB01
    - customer number and policy number
    - the last SQL request label
    - the SQLCODE
  - The record is sent by a CICS LINK to the logging program LGSTSQ, which writes it to the message queues.
- **Context labelling**
  - Before each SQL statement, a request label is placed in the error message (fetch, open, close, update policy, update endowment, update house, update motor). The log therefore shows which statement failed.
- **Commarea snapshot**
  - After the main error record, the procedure writes a second record holding up to 90 bytes of the commarea.
  - If the commarea is shorter than 91 bytes, it writes only what exists. If it is longer, it truncates to 90 bytes.
  - Both records are sent with the length of the error message structure. The second record is therefore sent with a length that does not match its own structure.
- **No-logging cases**
  - SQLCODE 100, the timestamp mismatch ('02') and the -501 close case return a code to the caller but write no error message.

### 8.7 Downstream Call Handling
- **Link to LGUPVS01**
  - After the database update routine finishes, the program links to the VSAM update program with a fixed commarea length of 225.
  - It does not check the return code first. The link is made even if the DB2 update failed and returned '01' or '02'.
  - It has no check of the outcome after the link, and no handling for a link failure.

## 9. Examples

### 9.1 Example 1: Motor policy updated successfully

**Input (COMMAREA, EIBCALEN > 0)**

| Field | Value |
|---|---|
| CA_REQUEST_ID | `01UMOT` |
| CA_CUSTOMER_NUM | `0000000001` |
| CA_POLICY_NUM | `0000000123` |
| CA_ISSUE_DATE / CA_EXPIRY_DATE | `2024-01-01` / `2025-01-01` |
| CA_LASTCHANGED | `2024-03-10-10.15.00.123456` (same as the value in the POLICY row) |
| CA_BROKERID / CA_BROKERSREF | `0000000005` / `BREF000123` |
| CA_M_MAKE / CA_M_MODEL | `Toyota` / `Corolla` |
| CA_M_VALUE / CA_M_CC | `015000` / `1600` |
| CA_M_PREMIUM / CA_M_ACCIDENTS | `000450` / `000001` |

**Expected output**
- The MOTOR row for policy 123 holds the new make, model, value, cc, premium and accidents.
- The POLICY row holds the new dates and broker data, and its `LASTCHANGED` is set to `CURRENT TIMESTAMP`.
- The COMMAREA comes back with `CA_RETURN_CODE = '00'` and `CA_LASTCHANGED` set to the new timestamp.
- The program links to `LGUPVS01` with a 225-byte COMMAREA, then returns.

**Explanation**
1. `POLICY_CURSOR` opens with SQLCODE 0 and fetches the one matching row (SQLCODE 0).
2. `CA_LASTCHANGED` equals `DB2_LASTCHANGED`, so the update is allowed to continue.
3. `SELECT(CA_REQUEST_ID)` matches `'01UMOT'` and calls `UPDATE_MOTOR_DB2_INFO`. That procedure moves the numeric fields into integer host variables and runs `UPDATE MOTOR`. The SQLCODE is 0, so the return code stays `'00'`.
4. `UPDATE POLICY ... WHERE CURRENT OF POLICY_CURSOR` runs, and a `SELECT LASTCHANGED` copies the new timestamp into the COMMAREA.
5. `CLOSE_PCURSOR` closes the cursor with SQLCODE 0, and the return code stays `'00'`.

### 9.2 Example 2: Stale timestamp (optimistic lock failure)

**Input**
- `CA_REQUEST_ID = '01UHOU'`, customer `0000000001`, policy `0000000123`.
- `CA_LASTCHANGED = '2024-03-01-09.00.00.000000'`. Another user has since changed the row, so DB2 holds `2024-03-10-10.15.00.123456`.

**Expected output**
- `CA_RETURN_CODE = '02'`.
- Neither the HOUSE row nor the POLICY row is changed.
- No error message is written to the queue.

**Explanation**
- The fetch succeeds, but `CA_LASTCHANGED <> DB2_LASTCHANGED`.
- The program takes the `ELSE` branch and sets the return code to `'02'`. It skips the policy-type update and the policy update.
- It then closes the cursor, which releases the lock, and returns.

### 9.3 Example 3: Policy not found

**Input**
- `CA_REQUEST_ID = '01UEND'`, customer `0000000001`, policy `0000009999`. No POLICY row matches this pair.

**Expected output**
- `CA_RETURN_CODE = '01'`.
- No tables are modified.

**Explanation**
- `OPEN` succeeds, but the first `FETCH` returns SQLCODE 100 (no row).
- Because the SQLCODE is 100, the return code is set to `'01'`. `WRITE_ERROR_MESSAGE` is not called.
- The cursor is closed and the program returns.

### 9.4 Example 4: Missing COMMAREA

**Input**
- The program is invoked with `EIBCALEN = 0`.

**Expected output**
- The message ` NO COMMAREA RECEIVED` is written to the error queue through `LGSTSQ`. This message carries the date and time, the program name, and the customer and policy numbers.
- The task abends with code `LGCA` and no dump.

**Explanation**
- The `EIBCALEN = 0` check runs before any DB2 access. It calls `WRITE_ERROR_MESSAGE` and then issues `EXEC CICS ABEND ABCODE('LGCA') NODUMP`.

### 9.5 Example 5: DB2 resource failure on cursor open

**Input**
- A valid `01UMOT` request, but `OPEN POLICY_CURSOR` returns SQLCODE -913 (deadlock or timeout).

**Expected output**
- `CA_RETURN_CODE = '90'`.
- An error record is written with `SQLREQ = ' OPEN   PCURSOR '` and the SQLCODE. The first 90 bytes of the COMMAREA are also logged as `COMMAREA=...`.
- The program returns immediately.

**Explanation**
- The `SELECT(SQLCODE)` after the open matches the `-913` case, and so would any other non-zero SQLCODE. Both set the return code to `'90'`, call `WRITE_ERROR_MESSAGE`, and issue `EXEC CICS RETURN`.

---

Generated by IBM Bob Premium Package for Z
