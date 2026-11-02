## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
  - [2.1 Procedure parameter](#21-procedure-parameter)
  - [2.2 CICS execution environment (EIB)](#22-cics-execution-environment-eib)
  - [2.3 Commarea: request control fields](#23-commarea-request-control-fields)
  - [2.4 Commarea: common policy details](#24-commarea-common-policy-details)
  - [2.5 Commarea: policy-type-specific details](#25-commarea-policy-type-specific-details)
  - [2.6 Configuration switch](#26-configuration-switch)
  - [2.7 External services and system data](#27-external-services-and-system-data)
- [3. Outputs](#3-outputs)
- [4. Processing Logic](#4-processing-logic)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
  - [6.1 CICS Middleware and System Services](#61-cics-middleware-and-system-services)
  - [6.2 External Programs and Subroutines](#62-external-programs-and-subroutines)
  - [6.3 Include Files and Data Structures](#63-include-files-and-data-structures)
  - [6.4 Input Parameters and Memory Pointers](#64-input-parameters-and-memory-pointers)
- [7. Constraints](#7-constraints)
  - [7.1 Communication Area and Interface Constraints](#71-communication-area-and-interface-constraints)
  - [7.2 Data Validation and Formatting Constraints](#72-data-validation-and-formatting-constraints)
  - [7.3 Execution Sequencing and Flow Constraints](#73-execution-sequencing-and-flow-constraints)
  - [7.4 Diagnostic Logging and Queue Constraints](#74-diagnostic-logging-and-queue-constraints)
- [8. Error Handling](#8-error-handling)
- [9. Examples](#9-examples)
  - [9.1 Example 1: Normal add request (commarea long enough, business rules off)](#91-example-1-normal-add-request-commarea-long-enough-business-rules-off)
  - [9.2 Example 2: Commarea too short](#92-example-2-commarea-too-short)
  - [9.3 Example 3: No commarea received](#93-example-3-no-commarea-received)
  - [9.4 Example 4: Endowment add with business rules switched on](#94-example-4-endowment-add-with-business-rules-switched-on)

## 1. Purpose

`LGAPOL01` is a CICS-hosted PL/I orchestrator program that handles the addition of new insurance policies — covering Endowment, House, Motor, Commercial, and Claim policy types — on behalf of the IBM General Insurance (GenApp) application. Invoked via a CICS LINK with a 32,500-byte communication area, it validates that a sufficient commarea has been provided, then optionally invokes the IBM Operational Decision Manager (ODM) business rules program `LGAPBR01` when the request is an endowment-add (`CA_REQUEST_ID = '01AEND'`) and the `BUSINESS_RULES` flag is set to `'Y'`. Regardless of whether business rule processing applies, it unconditionally delegates the actual Db2 data insertion to `LGAPDB01`, propagating the full commarea and surfacing any non-zero return code back to the caller. Error conditions — including a missing commarea — are logged to a CICS transient data queue via the shared `LGSTSQ` utility program before the transaction is terminated.

## 2. Inputs

### 2.1 Procedure parameter
- COMM_AREA_PTR
  - Pointer to the CICS communication area (DFHCOMMAREA) passed in by the calling program or transaction.
  - It addresses the `COMM_AREA` structure, which carries every policy field the program uses.

### 2.2 CICS execution environment (EIB)
- EIBCALEN
  - Length of the commarea received.
  - Zero causes an error message and an ABEND with code 'LGCA'.
  - A value below the required length sets the return code to '98' and returns to the caller.
  - Also decides how much of the commarea is written to the error queue.
- EIBTRNID
  - Transaction ID, saved for run-time and debug information.
- EIBTRMID
  - Terminal ID, saved for run-time and debug information.
- EIBTASKN
  - Task number, saved for run-time and debug information.

### 2.3 Commarea: request control fields
- CA_REQUEST_ID
  - Request identifier supplied by the caller.
  - The value '01AEND' (add endowment policy) triggers the business rule program LGAPBR01 when business rules are active.
- CA_RETURN_CODE
  - Status code in the commarea.
  - The program resets it to '00' on entry.
  - The called insert program LGAPDB01 sets it on return, and a value above zero ends the transaction.
- CA_CUSTOMER_NUM
  - Customer number, copied into the error message for traceability.

### 2.4 Commarea: common policy details
- CA_POLICY_NUM
  - Policy number.
- CA_BROKERID
  - Broker identifier.
- CA_PAYMENT
  - Payment or premium amount.
- CA_ISSUE_DATE
  - Policy issue date.
- CA_EXPIRY_DATE
  - Policy expiry date.

### 2.5 Commarea: policy-type-specific details
- Endowment
  - CA_E_TERM is the term in years.
  - CA_E_SUM_ASSURED is the sum assured.
- House
  - CA_H_VALUE is the house value.
- Motor
  - CA_M_VALUE is the vehicle value.
- Commercial
  - CA_B_Status is the commercial policy underwriting status.
- Claim
  - CA_C_Value is the claim value.

### 2.6 Configuration switch
- BUSINESS_RULES
  - Switch with default 'N'. It is set in the source rather than read from outside the program.
  - Changing it to 'Y' enables business rule processing for endowment adds.

### 2.7 External services and system data
- System date and time from CICS (ASKTIME/FORMATTIME)
  - Used to timestamp error messages.
- Linked programs
  - LGAPBR01 (business rules) and LGAPDB01 (data insert) receive the commarea and update it, including the return code, which is read back after the link.

## 3. Outputs

**COMMAREA Return Fields**
- `CA_RETURN_CODE` — the primary status code written back to the caller in the COMMAREA; set to `'00'` on entry, `'98'` if the COMMAREA is too short, or propagated as-is from `LGAPDB01` (and optionally `LGAPBR01`) if those linked programs set a non-zero value
- `CA_POLICY_NUM` — populated by the downstream `LGAPDB01` call; returns the newly assigned policy number to the caller after a successful insert
- Policy-type-specific fields updated by `LGAPDB01` and optionally adjusted by `LGAPBR01` before the insert (all within the COMMAREA union `CA_POLICY_REQUEST`):
  - `CA_ISSUE_DATE`, `CA_EXPIRY_DATE`, `CA_BROKERID`, `CA_PAYMENT` — common policy details echoed back or confirmed
  - `CA_E_SUM_ASSURED`, `CA_E_TERM` — endowment-specific result fields, subject to business-rule modification by `LGAPBR01` before the Db2 insert
  - `CA_H_VALUE` — house policy result field
  - `CA_M_VALUE` — motor policy result field
  - `CA_B_STATUS`, `CA_B_RejectReason` — commercial policy underwriting status and reject reason, settable by `LGAPBR01`
  - `CA_C_Value` — claim value field

**CICS Links (downstream program invocations)**
- `LGAPBR01` (conditional) — linked via `EXEC CICS LINK` only when `BUSINESS_RULES = 'Y'` and `CA_REQUEST_ID = '01AEND'`; receives and may modify the full 32500-byte COMMAREA before the Db2 insert; the updated COMMAREA fields (especially endowment and commercial fields) are its observable output
- `LGAPDB01` (unconditional) — linked via `EXEC CICS LINK` to perform the actual Db2 inserts for the policy; all COMMAREA policy fields and `CA_RETURN_CODE` are outputs returned through this call

**CICS Transient Data Queue (TDQ) error messages — written via `LGSTSQ`**
- Error message record written when no COMMAREA is received (`EIBCALEN = 0`); content: date, time, program name `LGAPOL01`, and the literal `' NO COMMAREA RECEIVED'`
  - `EM_DATE` / `EM_TIME` — current date and time formatted by `EXEC CICS FORMATTIME`
  - `EM_CUSNUM` — customer number from `CA_CUSTOMER_NUM`, embedded in the error record for traceability
  - `EM_POLNUM` — policy number slot in the error record
  - `EM_SQLREQ` / `EM_SQLRC` — SQL request identifier and SQLCODE slots in the error record (not populated by this program, but part of the fixed message structure written to the queue)
- COMMAREA dump record — up to 90 bytes of the raw COMMAREA (`CA_DATA`) are appended to the TDQ as a second message record when `EIBCALEN > 0` at the time the error is written

**CICS ABEND**
- `ABCODE('LGCA')` — issued via `EXEC CICS ABEND` with `NODUMP` when no COMMAREA is present; this is a visible, terminal side effect that terminates the CICS task

## 4. Processing Logic

- **Initialization and Context Capture**
  - The program is the CICS entry point (`Options(Main)`) for adding a policy. It receives the commarea through the pointer `COMM_AREA_PTR`, which overlays the `COMM_AREA` structure.
  - The commarea holds the request ID, return code, customer number, and a policy area. The policy area can be redefined as endowment, house, motor, commercial, or claim details.
  - The program copies the CICS environment values into the working-storage header `WS_HEADER`, for runtime diagnostics:
    - `EIBTRNID` goes to `WS_TRANSID`.
    - `EIBTRMID` goes to `WS_TERMID`.
    - `EIBTASKN` goes to `WS_TASKNUM`.
    - `EIBCALEN` goes to `WS_CALEN`.
  - `BUSINESS_RULES` defaults to 'N'. The line that would set it to 'Y' is commented out, so business rule processing is off unless someone edits the source.

- **Commarea Validation**
  - **No commarea received**
    - If `EIBCALEN = 0`, the program sets `EM_MSG_TEXT` to ' NO COMMAREA RECEIVED'.
    - It calls the internal procedure `WRITE_ERROR_MESSAGE` to log the error.
    - It then abends the task with `EXEC CICS ABEND ABCODE('LGCA') NODUMP`, so no dump is produced and control does not continue.
  - **Commarea present**
    - `CA_RETURN_CODE` is set to '00'.
    - The commarea address is saved in `WS_ADDR_DFHCOMMAREA`.
    - `CA_CUSTOMER_NUM` is copied to `EM_CUSNUM`, so any later error message carries the customer number.
  - **Length check**
    - `WS_REQUIRED_CA_LEN` is computed as `WS_CA_HEADER_LEN` (28) plus its own initial value of 0. No per-policy-type length is added, so the required length is 28.
    - If `EIBCALEN` is less than `WS_REQUIRED_CA_LEN`, the program sets `CA_RETURN_CODE` to '98' (commarea too short). It returns to the caller immediately with `EXEC CICS RETURN`, and no insert is attempted.

- **Optional Business Rule Processing**
  - This step runs only when `BUSINESS_RULES = 'Y'` and `CA_REQUEST_ID = '01AEND'` (add endowment policy).
  - In that case the program links to `LGAPBR01` with `COMM_AREA` and a length of 32500. That program applies the business rules, using the ODM-based rules processing named in the program comments.
  - `LGAPBR01` receives the commarea and may modify it before the insert. This includes the return code and the endowment sum assured.
  - With the default 'N', or for any other request ID, this step is skipped.

- **Policy Data Insert**
  - The program always links to `LGAPDB01` with `COMM_AREA` and a length of 32500. `LGAPDB01` performs the actual policy insert.
  - The policy data is passed as one block. It includes the common fields (issue and expiry dates, broker ID, payment) and the type-specific fields, such as endowment term and sum assured, house value, motor value, commercial status, and claim value.
  - The insert program writes its result back into the commarea through `CA_RETURN_CODE`.

- **Result Handling and Return**
  - After the link, if `CA_RETURN_CODE > 0`, the program issues `EXEC CICS RETURN`.
  - If the return code is 0, it also issues `EXEC CICS RETURN`.
  - Both paths therefore end the same way, and the caller sees the `CA_RETURN_CODE` set by the insert program.
  - The `RETURN;` statement after the final CICS RETURN is never reached.

- **Error Message Logging (`WRITE_ERROR_MESSAGE` Procedure)**
  - **Timestamp creation**
    - `EXEC CICS ASKTIME` obtains the current absolute time into `ABS_TIME`.
    - `EXEC CICS FORMATTIME` converts it to a date (`DATE1`, MMDDYYYY format) and a time (`TIME1`).
    - These values are placed in `EM_DATE` and `EM_TIME` of the `ERROR_MSG` structure.
  - **Message write**
    - `EXEC CICS LINK PROGRAM('LGSTSQ')` passes `ERROR_MSG` as the commarea, with a length of `STG(ERROR_MSG)`. `LGSTSQ` writes the message to the queues.
    - The message holds the date, time, program name 'LGAPOL01', and a variable part. The variable part is either free text (`EM_MSG_TEXT`) or the customer number, policy number, SQL request, and SQLCODE fields.
  - **Commarea snapshot**
    - If `EIBCALEN > 0`, the procedure also writes up to 90 bytes of the commarea.
    - If `EIBCALEN < 91`, `CA_DATA` is set to the left-justified `EIBCALEN` bytes of `COMM_AREA_RAW`.
    - Otherwise, `CA_DATA` takes the first 90 bytes.
    - In both cases `CA_ERROR_MSG` (prefixed 'COMMAREA=') is sent to `LGSTSQ` through another `EXEC CICS LINK`. The length given is `STG(ERROR_MSG)`, not the size of `CA_ERROR_MSG`.
    - When the commarea is empty (`EIBCALEN = 0`, as in the no-commarea case), this snapshot is skipped.

- **External Interactions**
  - **Programs called through CICS LINK:**
    - `LGAPBR01` for business rules, conditional.
    - `LGAPDB01` for the policy insert, always called after validation.
    - `LGSTSQ` for error logging.
  - **CICS services:**
    - `ASKTIME` and `FORMATTIME` for date and time.
    - `ABEND` for a missing commarea.
    - `RETURN` for ending the task.
  - The program does no direct file or database I/O. All persistence happens in the linked programs.
  - The program has no loops. Control flow is a linear sequence with guard conditions that exit early.
  - The constants `LGAPDB1`, `LGAPBR1` and `LGAPVS1` are declared but not used in the executable logic. The `LINK` statements use literal program names.

## 5. Paragraphs

- **LGAPOL01 (main procedure, `Proc(COMM_AREA_PTR) Options(Main)`)**
  - Purpose: CICS entry point for adding a policy (Endowment, House, Motor or Commercial). It validates the commarea, optionally applies business rules, then delegates the database insert to LGAPDB01.
  - Declarations:
    - It declares the working storage: the debug header `WS_HEADER`, the time and date fields, the `BUSINESS_RULES` switch (default 'N'), and the `ERROR_MSG` and `CA_ERROR_MSG` structures.
    - It declares the program-name constants LGAPDB01, LGAPBR01 and LGAPVS01, the commarea length fields, and the based `COMM_AREA` layout (the LGCMAREA include).
    - `COMM_AREA` is based on `COMM_AREA_PTR` and holds the request ID, return code, customer number and policy fields, including `CA_POLICY_NUM`, `CA_BROKERID`, `CA_PAYMENT`, the issue and expiry dates, and the endowment, house, motor, commercial and claim sub-structures.
  - Operations:
    - It copies EIBTRNID, EIBTRMID, EIBTASKN and EIBCALEN into `WS_TRANSID`, `WS_TERMID`, `WS_TASKNUM` and `WS_CALEN`.
    - If EIBCALEN is 0, it sets `EM_MSG_TEXT` to ' NO COMMAREA RECEIVED', calls `WRITE_ERROR_MESSAGE`, and abends with `EXEC CICS ABEND ABCODE('LGCA') NODUMP`.
    - It sets `CA_RETURN_CODE` to '00', saves the commarea pointer in `WS_ADDR_DFHCOMMAREA`, and copies `CA_CUSTOMER_NUM` into `EM_CUSNUM` for error traceability.
    - It adds the header length (28) to `WS_REQUIRED_CA_LEN` and compares the total with EIBCALEN.
    - It links to LGAPBR01 for business rules when required, then links to LGAPDB01 for the insert.
  - Control flow:
    - Conditional 1: if EIBCALEN is 0, it logs the error and abends.
    - Conditional 2: if EIBCALEN is less than the required length, it sets `CA_RETURN_CODE` to '98' and issues `EXEC CICS RETURN`.
    - Conditional 3: if `BUSINESS_RULES` is 'Y' and `CA_REQUEST_ID` is '01AEND' (endowment add), it performs `EXEC CICS LINK PROGRAM('LGAPBR01')`. `BUSINESS_RULES` is initialised to 'N' and never set to 'Y' in the code, so this link is skipped by default.
    - After the LGAPDB01 link, if `CA_RETURN_CODE` is greater than 0 it returns to the caller. Otherwise it returns to the caller by the final `EXEC CICS RETURN`. The trailing PL/I `RETURN` statement is not reached.
  - External interactions:
    - CICS LINK to LGAPBR01 (business rules) with the commarea `COMM_AREA`, length 32500. It is conditional.
    - CICS LINK to LGAPDB01 (data insert) with the commarea `COMM_AREA`, length 32500.
    - CICS ABEND and CICS RETURN.
    - The commarea is passed in and out, and the return code reports the result to the caller.

- **WRITE_ERROR_MESSAGE (internal procedure)**
  - Purpose: Builds a timestamped error message and writes it to the error queue through the LGSTSQ program. It is called when no commarea is received.
  - Operations:
    - It issues `EXEC CICS ASKTIME ABSTIME(ABS_TIME)` to get the current time.
    - It issues `EXEC CICS FORMATTIME` to produce `DATE1` (MMDDYYYY) and `TIME1`, then moves them into `EM_DATE` and `EM_TIME`.
    - It links to LGSTSQ with `ERROR_MSG` as the commarea, using `STG(ERROR_MSG)` as the length. The message holds the date, time, program name 'LGAPOL01', the customer and policy numbers, and the message text.
    - It then writes up to 90 bytes of the received commarea as a second message: `CA_DATA` is set from `LEFT(COMM_AREA_RAW, ...)`, and `CA_ERROR_MSG` is sent to LGSTSQ.
  - Control flow:
    - If EIBCALEN is greater than 0, it writes the commarea data.
    - If EIBCALEN is less than 91, `CA_DATA` takes the first EIBCALEN bytes of the commarea.
    - Otherwise, `CA_DATA` takes the first 90 bytes.
    - Both branches issue the same LINK to LGSTSQ with `CA_ERROR_MSG`. The length passed is `STG(ERROR_MSG)`, not the size of `CA_ERROR_MSG`.
    - The procedure ends with `RETURN`.
  - External interactions:
    - CICS ASKTIME and FORMATTIME for the current date and time.
    - CICS LINK to LGSTSQ, which writes to the error queues (TDQ/TSQ).

## 6. Dependencies

### 6.1 CICS Middleware and System Services
- CICS System Interface
  - EXEC CICS LINK: Used to transfer control synchronously to external modules (`LGAPDB01`, `LGAPBR01`, and `LGSTSQ`), passing communication structures.
  - EXEC CICS RETURN: Returns control and execution status back to the calling CICS program or terminal transaction.
  - EXEC CICS ABEND: Terminates the task abnormally with abend code `'LGCA'` (`NODUMP`) when no COMMAREA is passed (`EIBCALEN = 0`).
  - EXEC CICS ASKTIME and FORMATTIME: Retrieves absolute system time (`ABSTIME`) and formats the current date (`MMDDYYYY`) and time (`TIME`) for error tracking.
- CICS EXEC Interface Block (EIB)
  - `EIBCALEN`: Contains the length of the passed communication area used to validate incoming request payloads.
  - `EIBTRNID`: Identifies the executing CICS transaction identifier.
  - `EIBTRMID`: Identifies the executing CICS terminal identifier.
  - `EIBTASKN`: Contains the unique CICS task number assigned to the transaction execution.

### 6.2 External Programs and Subroutines
- `LGAPDB01`
  - Policy database insert program invoked via `EXEC CICS LINK` to perform backend persistence for new policy records across different policy types (Endowment, House, Motor, Commercial).
- `LGAPBR01`
  - External Business Rules (ODM) validation program conditionally invoked via `EXEC CICS LINK` when business rules processing is enabled (`BUSINESS_RULES = 'Y'`) and an endowment policy is being added (`CA_REQUEST_ID = '01AEND'`).
- `LGSTSQ`
  - Standard error logging routine invoked via `EXEC CICS LINK` to write formatted error headers (`ERROR_MSG`) and raw COMMAREA payload fragments (`CA_ERROR_MSG`) to temporary storage queues or transient data destinations.

### 6.3 Include Files and Data Structures
- `LGCMAREA` (`plinc/LGCMAREA.inc`)
  - Provides the central 32,500-byte communication area definition (`COMM_AREA`) mapped over `COMM_AREA_PTR`, establishing request/response data contracts between orchestrators, UI menus, and backend database programs. Key mapped fields include:
    - `CA_REQUEST_ID`: Six-character request function code (e.g., `'01AEND'`).
    - `CA_RETURN_CODE`: Two-digit return code indicating processing status (`'00'` for success, `'98'` for invalid COMMAREA length, or downstream error codes).
    - `CA_CUSTOMER_NUM`: Ten-digit customer identifier.
    - `CA_POLICY_REQUEST`: Sub-structure containing shared policy headers and specific policy unions (`CA_ENDOWMENT`, `CA_HOUSE`, `CA_MOTOR`, `CA_COMMERCIAL`, `CA_CLAIM`).

### 6.4 Input Parameters and Memory Pointers
- `COMM_AREA_PTR`
  - Main procedure entry pointer parameter (`Proc(COMM_AREA_PTR) Options(Main)`) pointing to the CICS `DFHCOMMAREA` data buffer passed by the invoking program.

## 7. Constraints

### 7.1 Communication Area and Interface Constraints
- Communication area presence constraint:
  - The program requires an active CICS communication area (`DFHCOMMAREA`).
  - If `EIBCALEN` equals `0`, the program writes an error message (`NO COMMAREA RECEIVED`) to the error queue via `WRITE_ERROR_MESSAGE` and terminates immediately by issuing `EXEC CICS ABEND ABCODE('LGCA') NODUMP`.
- Communication area minimum length constraint:
  - The program enforces a minimum incoming commarea length equal to the common policy header length (`WS_CA_HEADER_LEN = 28` bytes).
  - If `EIBCALEN` is less than `WS_REQUIRED_CA_LEN` (28 bytes), the program sets `CA_RETURN_CODE = '98'` and returns immediately via `EXEC CICS RETURN` without executing downstream processing.
- Communication area fixed payload buffer constraint:
  - Downstream program links (`LGAPBR01` and `LGAPDB01`) pass the communication area structure `COMM_AREA` with an exact fixed length constraint of `LENGTH(32500)`.
  - The total declared size of the based `COMM_AREA` union is 32,500 bytes (composed of 6 bytes `CA_REQUEST_ID`, 2 bytes `CA_RETURN_CODE`, 10 bytes `CA_CUSTOMER_NUM`, and 32,482 bytes `CA_REQUEST_SPECIFIC`).

### 7.2 Data Validation and Formatting Constraints
- Request identification formatting:
  - The request type identifier `CA_REQUEST_ID` is defined as a fixed 6-character field (`CHAR(6)`), which dictates the conditional routing rules.
- Return status code values and initialization:
  - `CA_RETURN_CODE` is defined as a 2-digit numeric picture (`PIC '99'`).
  - The program enforces initialization of `CA_RETURN_CODE` to `'00'` on entry before performing length validation and dispatching calls.
  - Return code `'98'` is specifically assigned when the supplied commarea is shorter than 28 bytes.
- Numeric identifier definitions:
  - Customer number `CA_CUSTOMER_NUM` and policy number `CA_POLICY_NUM` are constrained to 10-digit numeric values (`PIC '9999999999'`).
  - Broker identifier `CA_BROKERID` is constrained to 10 digits (`PIC '9999999999'`).
  - Payment amount `CA_PAYMENT` is constrained to a 6-digit numeric field (`PIC '999999'`).
- Policy-specific data field formatting:
  - Endowment policy: `CA_E_TERM` is constrained to a 2-digit numeric value (`PIC '99'`), and `CA_E_SUM_ASSURED` is constrained to 6 numeric digits (`PIC '999999'`).
  - House policy: `CA_H_BEDROOMS` is constrained to 3 digits (`PIC '999'`), and `CA_H_VALUE` is constrained to 8 digits (`PIC '99999999'`).
  - Motor policy: `CA_M_VALUE`, `CA_M_PREMIUM`, and `CA_M_ACCIDENTS` are constrained to 6 numeric digits (`PIC '999999'`), and `CA_M_CC` is constrained to 4 digits (`PIC '9999'`).
  - Commercial policy: Peril and status codes (`CA_B_FirePeril`, `CA_B_CrimePeril`, `CA_B_FloodPeril`, `CA_B_WeatherPeril`, `CA_B_Status`) are constrained to 4 numeric digits (`PIC '9999'`), while associated premiums are constrained to 8 numeric digits (`PIC '99999999'`).
  - Claim policy: Claim identifier `CA_C_Num` is constrained to 10 digits (`PIC '9999999999'`), and amounts `CA_C_Paid` and `CA_C_Value` are constrained to 8 digits (`PIC '99999999'`).

### 7.3 Execution Sequencing and Flow Constraints
- Conditional business rules invocation:
  - Business rules processing is conditionally guarded by two sequential checks: `BUSINESS_RULES = 'Y'` and `CA_REQUEST_ID = '01AEND'`.
  - The `BUSINESS_RULES` flag is statically initialized to `'N'`, requiring explicit modification to enable business rules invocation.
  - When enabled, `LGAPBR01` is strictly executed only for endowment policy additions (`'01AEND'`) prior to calling the database insert program.
- Backend persistence invocation and termination:
  - The database insert program `LGAPDB01` is invoked unconditionally via `EXEC CICS LINK` if the commarea validation checks pass.
  - If `CA_RETURN_CODE` returned by `LGAPDB01` is greater than `0`, the program immediately exits via `EXEC CICS RETURN`, preventing further processing.

### 7.4 Diagnostic Logging and Queue Constraints
- Error queue messaging payload limit:
  - In the `WRITE_ERROR_MESSAGE` internal routine, commarea data logged to the transient data queue via `LGSTSQ` is capped at a maximum length of 90 bytes.
  - If `0 < EIBCALEN < 91`, only the available `EIBCALEN` bytes are copied using `LEFT(COMM_AREA_RAW, EIBCALEN)`; otherwise, exactly 90 bytes are truncated and copied using `LEFT(COMM_AREA_RAW, 90)`.

## 8. Error Handling

- **Commarea presence validation**
  - The program tests whether the CICS-supplied commarea length (EIBCALEN) is zero at the start of the common code.
  - If no commarea was received, it sets the error message text to " NO COMMAREA RECEIVED" and calls the local WRITE_ERROR_MESSAGE procedure to log the problem.
  - It then abnormally terminates the transaction with a CICS ABEND using abend code 'LGCA' and the NODUMP option, so no dump is produced.
  - This is a fatal path. Control does not continue to the policy processing.

- **Commarea length validation**
  - The program computes a required commarea length: the fixed header length (28 bytes) plus the required-length working field, which is initialised to 0.
  - If EIBCALEN is smaller than this required length, the commarea is too short.
  - Recovery is graceful. CA_RETURN_CODE is set to '98' and the program issues a CICS RETURN to the caller. It does not abend and does not write an error message.
  - The caller detects the problem through the '98' return code.
  - The required length contains only the header length. It is not extended for the different policy types, so the check covers the header only and not the policy-specific data.

- **Return code initialisation and propagation**
  - After the commarea presence check passes, CA_RETURN_CODE is set to '00'. This gives the caller a known success state before any processing.
  - After the data insert program LGAPDB01 is linked, the program tests CA_RETURN_CODE.
    - If it is greater than zero, the insert program has reported a failure. The program issues a CICS RETURN so the non-zero code goes back to the caller.
    - The success path also ends in a CICS RETURN, so both outcomes return to the caller. The return code is the only difference between them.
  - The program does not handle the error itself. It passes the failure status upstream, and the caller must interpret the code.

- **Error context capture for traceability**
  - The customer number from the commarea (CA_CUSTOMER_NUM) is copied into the error message structure (EM_CUSNUM) before validation continues. Any later error message can then identify the customer.
  - The task's identifying details are saved at entry: transaction ID, terminal ID, task number and commarea length. These serve as diagnostic information about the invocation.
  - The error message structure also has fields for policy number, SQL request and SQLCODE. This program does not populate them, because it makes no SQL calls. The fields appear to be shared with other programs.

- **Error logging and notification (WRITE_ERROR_MESSAGE procedure)**
  - The procedure gets the current time with CICS ASKTIME and formats it into date and time strings with CICS FORMATTIME. These are placed in the message so each entry is timestamped.
  - It writes the formatted error message (date, time, program name, customer number and message text) by linking to the queue-writing program LGSTSQ, which writes to the queues.
  - It then writes a second record containing part of the commarea contents, for diagnosis.
    - If EIBCALEN is greater than 0 and less than 91, it copies the commarea contents (up to EIBCALEN bytes) into the data portion of the commarea error message.
    - If EIBCALEN is 91 or more, it copies only the first 90 bytes.
    - The data is sent to LGSTSQ in a second link. This limits the size of the logged data, so the logging cannot overrun its buffer.
  - The procedure is only invoked from the missing-commarea path. The short-commarea path and the insert-failure path do not log through it.
  - The procedure does not check the outcome of its own CICS commands. A failure in the logging links is not detected.

- **Conditional business rule processing**
  - The business rule program LGAPBR01 is linked only if the BUSINESS_RULES switch is 'Y' and the request ID is '01AEND' (endowment add). The switch defaults to 'N', so the rule check is bypassed unless it is deliberately enabled in code.
  - This is a control mechanism that selectively skips the business rules path. It is not a failure handler.
  - No check follows the link to LGAPBR01. If rule processing fails, nothing in this program responds to it directly. Any failure would have to be reflected in the commarea return code, and this program looks at that only after the LGAPDB01 link.

- **Absence of explicit CICS response and condition handling**
  - The EXEC CICS LINK commands to LGAPBR01, LGAPDB01 and LGSTSQ do not use RESP or RESP2 options, and the program defines no ON-unit or condition handlers.
  - A failure of a link, such as a program not found, would therefore fall to default CICS handling, which normally abends the task. The program has no recovery path for these cases.
  - The only recovery logic is the early return with code '98', and the passing back of a non-zero return code from the insert program.

## 9. Examples

### 9.1 Example 1: Normal add request (commarea long enough, business rules off)

**Input**
- EIBCALEN = 200
- CA_REQUEST_ID = `01AEND`
- CA_CUSTOMER_NUM = `0000000123`
- CA_POLICY_NUM = `0000000456`
- CA_PAYMENT = `000150`
- CA_E_TERM = `20`
- CA_E_SUM_ASSURED = `050000`
- BUSINESS_RULES = `N` (the default)

**Expected output**
- The program sets CA_RETURN_CODE to `00` and links to `LGAPDB01` with the commarea.
- If `LGAPDB01` leaves CA_RETURN_CODE at `00`, the program returns to the caller with CA_RETURN_CODE = `00`.
- `LGAPBR01` is not called, and no error message is written.

**Explanation**
- EIBCALEN is 200, which is at least the required 28 bytes (`WS_CA_HEADER_LEN` of 28 plus `WS_REQUIRED_CA_LEN` of 0), so the length check passes.
- BUSINESS_RULES stays `N`, so the business rule link is skipped even though CA_REQUEST_ID is `01AEND`.
- The data insert program `LGAPDB01` is always linked next.
- A return code of 0 is not greater than 0, so the program takes the normal `EXEC CICS RETURN`.

### 9.2 Example 2: Commarea too short

**Input**
- EIBCALEN = 20
- CA_REQUEST_ID = `01ACUS`
- CA_CUSTOMER_NUM = `0000000123`

**Expected output**
- CA_RETURN_CODE = `98`.
- The program returns immediately to the caller.
- `LGAPDB01` is not linked.

**Explanation**
- EIBCALEN is not 0, so there is no abend.
- The program first sets CA_RETURN_CODE to `00` and copies the customer number into EM_CUSNUM.
- The required length is 28, and 20 is less than 28.
- The program therefore sets CA_RETURN_CODE to `98` and issues `EXEC CICS RETURN` before any insert is attempted.

### 9.3 Example 3: No commarea received

**Input**
- EIBCALEN = 0

**Expected output**
- The program writes an error record to the queue through `LGSTSQ`. The record has this form: `MM/DD/YYYY HH:MM:SS LGAPOL01 NO COMMAREA RECEIVED`.
- The task then abends with ABCODE `LGCA` and no dump.

**Explanation**
- The zero-length test is the first check in the program.
- `WRITE_ERROR_MESSAGE` sets EM_MSG_TEXT to `NO COMMAREA RECEIVED`.
- It gets the current time with `ASKTIME` and `FORMATTIME`, and links to `LGSTSQ` to write the message.
- The second `LGSTSQ` link, which writes the commarea data, is skipped because EIBCALEN is 0.
- The program then issues `EXEC CICS ABEND ABCODE('LGCA') NODUMP`.

### 9.4 Example 4: Endowment add with business rules switched on

**Input**
- EIBCALEN = 200
- CA_REQUEST_ID = `01AEND`
- CA_E_SUM_ASSURED = `050000`
- BUSINESS_RULES = `Y`. This needs a code change, because the `MOVE 'Y'` line is commented out in the source.

**Expected output**
- The program links to `LGAPBR01` first, with the 32500-byte commarea.
- It then links to `LGAPDB01`.
- On return, CA_RETURN_CODE holds the value set by those programs. A value above 0 ends the transaction with `EXEC CICS RETURN`. If it is `00`, the normal return is taken.

**Explanation**
- Both conditions for the rules call are met: BUSINESS_RULES is `Y` and CA_REQUEST_ID is `01AEND`.
- `LGAPBR01` can therefore review or adjust the policy data before `LGAPDB01` inserts it.
- For any other request ID, such as a house add, only the `LGAPDB01` link is done, even with the switch on.

---

Generated by IBM Bob Premium Package for Z
