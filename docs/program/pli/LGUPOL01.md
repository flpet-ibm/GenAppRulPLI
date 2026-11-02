## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
  - [2.1 CICS Communication Area (DFHCOMMAREA / `COMM_AREA`)](#21-cics-communication-area-dfhcommarea--comm_area)
  - [2.2 CICS System / Execution Interface Block (EIB)](#22-cics-system--execution-interface-block-eib)
- [3. Outputs](#3-outputs)
- [4. Processing Logic](#4-processing-logic)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
  - [6.1 Logical Categories of Dependencies for LGUPOL01.pli](#61-logical-categories-of-dependencies-for-lgupol01pli)
  - [6.2 Included Source Files](#62-included-source-files)
  - [6.3 External Program and Module Invocations](#63-external-program-and-module-invocations)
  - [6.4 CICS System APIs and Runtime Environment](#64-cics-system-apis-and-runtime-environment)
- [7. Constraints](#7-constraints)
  - [7.1 Commarea presence and availability constraints](#71-commarea-presence-and-availability-constraints)
  - [7.2 Request type validation constraints](#72-request-type-validation-constraints)
  - [7.3 Commarea length constraints](#73-commarea-length-constraints)
  - [7.4 Return code constraints](#74-return-code-constraints)
  - [7.5 Processing sequence and flow constraints](#75-processing-sequence-and-flow-constraints)
  - [7.6 Data format constraints (implied by field definitions)](#76-data-format-constraints-implied-by-field-definitions)
  - [7.7 Error message and logging constraints](#77-error-message-and-logging-constraints)
  - [7.8 Resource and size limits](#78-resource-and-size-limits)
- [8. Error Handling](#8-error-handling)
  - [8.1 Input and Communication Area Validation](#81-input-and-communication-area-validation)
  - [8.2 Logging and Notification Handling](#82-logging-and-notification-handling)
  - [8.3 Downstream Error Delegation](#83-downstream-error-delegation)
- [9. Examples](#9-examples)
  - [9.1 Example 1 — Update a Motor Policy (Happy Path)](#91-example-1--update-a-motor-policy-happy-path)
  - [9.2 Example 2 — Update a House Policy with a Short COMMAREA](#92-example-2--update-a-house-policy-with-a-short-commarea)
  - [9.3 Example 3 — Unrecognised Request ID](#93-example-3--unrecognised-request-id)

## 1. Purpose

LGUPOL01 is an online CICS orchestrator program in the GenApp insurance application responsible for validating and coordinating policy update requests across different insurance types, including endowment, house, and motor policies. It serves as the business layer interface between calling programs or presentation menus and backend database update programs, verifying incoming communication area parameters and lengths before delegating the persistence of updated customer and policy data to the Db2 update backend program.

## 2. Inputs

### 2.1 CICS Communication Area (DFHCOMMAREA / `COMM_AREA`)
Passed into the program via the main procedure parameter pointer (`COMM_AREA_PTR`), providing the input request details for policy updates:
- **Header and Request Identifiers**
  - `CA_REQUEST_ID`: A 6-character code specifying the policy update action (`'01UEND'` for endowment, `'01UHOU'` for house, or `'01UMOT'` for motor). It drives program routing and determines the required minimum commarea length.
  - `CA_CUSTOMER_NUM`: A 10-digit customer identifier associated with the policy to be updated. Used in error message formatting and passed along to the database update program.
  - `CA_POLICY_NUM`: A 10-digit unique policy identifier specifying the policy record to be modified.
- **Common Policy Data**
  - `CA_ISSUE_DATE`: A 10-character field holding the start date of the insurance policy.
  - `CA_EXPIRY_DATE`: A 10-character field holding the termination/expiration date of the insurance policy.
  - `CA_BROKERID`: A 10-digit identifier specifying the broker who manages or sold the policy.
  - `CA_BROKERSREF`: A 10-character reference string representing the policy identifier used in the broker's system.
  - `CA_PAYMENT`: A 6-digit numeric field representing the premium or payment amount due on the policy.
- **Endowment Policy Details (`CA_ENDOWMENT`)**
  - `CA_E_TERM`: A 2-digit numeric field indicating the term length (in years) of the endowment policy.
  - `CA_E_SUM_ASSURED`: A 6-digit numeric field indicating the guaranteed payout amount upon maturity or death.
  - `CA_E_LIFE_ASSURED`: A 31-character field containing the full name of the insured person.
- **House Policy Details (`CA_HOUSE`)**
  - `CA_H_PROPERTY_TYPE`: A 15-character description indicating the type of residential property (e.g., detached, bungalow).
  - `CA_H_VALUE`: An 8-digit numeric field containing the insured valuation of the property.
- **Motor Policy Details (`CA_MOTOR`)**
  - `CA_M_MAKE`: A 15-character string specifying the manufacturer/make of the insured vehicle.
  - `CA_M_VALUE`: A 6-digit numeric field representing the market/insured value of the vehicle.
  - `CA_M_PREMIUM`: A 6-digit numeric field containing the calculated premium for the motor policy.

### 2.2 CICS System / Execution Interface Block (EIB)
Maintained by the CICS runtime environment and queried directly by the program to govern transaction context and validation:
- `EIBCALEN`: Provides the length of the incoming communication area, used to check whether a commarea was supplied and ensure it meets the minimum byte length required for the specific policy type.
- `EIBTRNID`: The 4-character identifier of the executing CICS transaction, captured for execution tracking and diagnostics.
- `EIBTRMID`: The 4-character terminal identifier from which the transaction was initiated.
- `EIBTASKN`: The 7-digit packed-decimal task number assigned by CICS to the running transaction.

## 3. Outputs

- **Commarea return status (returned to caller)**
  - CA_RETURN_CODE
    - Set to '00' once a commarea is confirmed present.
    - Set to '98' when the supplied commarea length (EIBCALEN) is shorter than the minimum for the requested policy type. The program returns to CICS without calling the update program.
    - Set to '99' when CA_REQUEST_ID is not '01UEND', '01UHOU' or '01UMOT'.
    - Any other value comes from LGUPDB01, which is linked with the commarea and can change it before the program returns.
  - Commarea returned to the caller
    - EXEC CICS RETURN passes back the whole commarea (COMM_AREA_RAW, 32,500 bytes). It includes the request fields and any changes LGUPDB01 made.

- **Data passed to the database update program (LGUPDB01)**
  - The program does not write to the database itself. It links to LGUPDB01 with the full commarea, and LGUPDB01 performs the policy update.
  - Request and key fields
    - CA_REQUEST_ID selects the policy type to update.
    - CA_CUSTOMER_NUM identifies the customer.
    - CA_POLICY_NUM identifies the policy record.
  - Common policy details
    - CA_ISSUE_DATE and CA_EXPIRY_DATE
    - CA_BROKERID and CA_BROKERSREF
    - CA_PAYMENT
  - Endowment policy details (request '01UEND')
    - CA_E_TERM
    - CA_E_SUM_ASSURED
    - CA_E_LIFE_ASSURED
  - House policy details (request '01UHOU')
    - CA_H_PROPERTY_TYPE
    - CA_H_VALUE
  - Motor policy details (request '01UMOT')
    - CA_M_MAKE
    - CA_M_VALUE
    - CA_M_PREMIUM
  - The program calls LGUPDB01 even when CA_REQUEST_ID is unsupported and the return code is '99'. The return code is not checked before the link.

- **Error messages written to the queue (through LGSTSQ)**
  - These messages are written only when no commarea is received (EIBCALEN = 0).
  - Error message record (ERROR_MSG)
    - Contains the formatted date (EM_DATE), the time (EM_TIME), the program name and the text ' NO COMMAREA RECEIVED'.
    - The customer number (CA_CUSTOMER_NUM) and policy number (CA_POLICY_NUM) are copied into the message fields EM_CUSNUM and EM_POLNUM. In the no-commarea path they are not populated, because they are only copied after that check.
  - Commarea dump record (CA_ERROR_MSG)
    - Holds up to the first 90 bytes of the commarea, prefixed with 'COMMAREA='.
    - It is written only when EIBCALEN is greater than 0, so it is not produced in the no-commarea path.

- **Abnormal termination**
  - When no commarea is received, the program issues a CICS ABEND with code 'LGCA' and no dump, after writing the error message.

- **Control flow outcomes**
  - Early return to the caller with return code '98' when the commarea is too short.
  - Normal return to the caller after LGUPDB01 completes.

## 4. Processing Logic

-	**Initialization and Runtime Context Capture**
	-	The program runs as a CICS main PL/I procedure. It receives its communication area through `COMM_AREA_PTR`.
	-	It copies the transaction ID (`EIBTRNID`), terminal ID (`EIBTRMID`) and task number (`EIBTASKN`) into the `WS_HEADER` debug fields (`WS_TRANSID`, `WS_TERMID`, `WS_TASKNUM`).
	-	The working storage holds fixed length constants for the commarea header (28 bytes) and for the full endowment (124), house (130) and motor (137) request layouts. These are used for length validation.

- **Commarea Presence Validation**
	-	If `EIBCALEN` is 0, no commarea was received.
		-	The program sets `EM_MSG_TEXT` to ' NO COMMAREA RECEIVED'.
		-	It calls `WRITE_ERROR_MESSAGE` to log the problem.
		-	It then issues `EXEC CICS ABEND ABCODE('LGCA') NODUMP`, which ends the task abnormally without a dump.
	-	If a commarea is present, the program does the following:
		-	It sets `CA_RETURN_CODE` to '00'.
		-	It saves `EIBCALEN` in `WS_CALEN`.
		-	It saves the commarea address in `WS_ADDR_DFHCOMMAREA`.
		-	It copies `CA_CUSTOMER_NUM` and `CA_POLICY_NUM` into `EM_CUSNUM` and `EM_POLNUM`, so later error messages can include them.

- **Request Type Determination and Length Validation**
	-	A `SELECT` on `CA_REQUEST_ID` chooses the policy type and the minimum commarea length.
		-	For '01UEND' (endowment), the required length is `WS_CA_HEADER_LEN` plus `WS_FULL_ENDOW_LEN`.
		-	For '01UHOU' (house), the required length is `WS_CA_HEADER_LEN` plus `WS_FULL_HOUSE_LEN`.
		-	For '01UMOT' (motor), the required length is `WS_CA_HEADER_LEN` plus `WS_FULL_MOTOR_LEN`.
		-	For each of these three types, `WS_REQUIRED_CA_LEN` is built up in two additive steps.
			-	If `EIBCALEN` is less than the required length, the program sets `CA_RETURN_CODE` to '98'.
			-	It then issues `EXEC CICS RETURN` immediately, so no update is attempted.
		-	For any other request ID (`OTHERWISE`), the program sets `CA_RETURN_CODE` to '99' to mark an unsupported request.
			-	The program does not return at this point. Control falls through to the update call.

- **Database Update Delegation**
	-	After the `SELECT`, the program calls the internal procedure `UPDATE_POLICY_DB2_INFO`.
		-	The procedure issues `EXEC CICS LINK PROGRAM('LGUPDB01')` with the full commarea (`COMM_AREA_RAW`, length 32500).
		-	`LGUPDB01` receives the customer number, policy number, common policy details and the type-specific data.
			-	The common policy details are issue date, expiry date, broker ID, broker's reference and payment.
			-	The type-specific data is the endowment, house or motor fields.
		-	`LGUPDB01` performs the actual DB2 update. This program does no SQL itself.
		-	Any return code set by `LGUPDB01` stays in `CA_RETURN_CODE` of the shared commarea.
	-	The program then issues `EXEC CICS RETURN` to give control back to the caller.

- **Error Message Logging (WRITE_ERROR_MESSAGE)**
	-	Timestamp preparation
		-	`EXEC CICS ASKTIME` gets the current absolute time into `WS_ABSTIME`.
		-	`EXEC CICS FORMATTIME` converts it to an MMDDYYYY date (`WS_DATE`) and a time (`WS_TIME`).
		-	These values are moved into `EM_DATE` and `EM_TIME` in the error message structure.
	-	Error message output
		-	`EXEC CICS LINK PROGRAM('LGSTSQ')` passes `ERROR_MSG`. The message contains the date, time, program name 'LGUPOL01', the message text (or customer and policy numbers) and the length from `STG(CA_ERROR_MSG)`.
		-	`LGSTSQ` writes the message to the queues.
	-	Commarea dump for diagnostics
		-	This step runs only if `EIBCALEN` is greater than 0.
			-	If `EIBCALEN` is less than 91, `CA_DATA` is set to the left-justified commarea content of length `EIBCALEN`.
			-	Otherwise, `CA_DATA` is set to the first 90 bytes of `COMM_AREA_RAW`.
			-	In both cases the program issues another `EXEC CICS LINK PROGRAM('LGSTSQ')` with `CA_ERROR_MSG`, so the commarea content is logged.
		-	This step is skipped in the no-commarea path, where `EIBCALEN` is 0, so only the first message is written.

- **Control Flow and External Interactions Summary**
	-	There are no loops. Control flow is sequential with conditional branching (`IF` and `SELECT`).
	-	There is no direct file I/O or SQL. All external interaction goes through CICS services.
		-	`LINK` to `LGUPDB01` performs the DB2 policy update.
		-	`LINK` to `LGSTSQ` writes error and diagnostic messages to queues.
		-	`ASKTIME` and `FORMATTIME` supply timestamps.
		-	`ABEND` and `RETURN` end the task.
	-	The return codes written to `CA_RETURN_CODE` are '00' for normal start, '98' for a commarea that is too short, and '99' for an unsupported request.

## 5. Paragraphs

- **Main Procedure Body (LGUPOL01)**
  - Entry point of the program, invoked as a CICS main procedure via `Proc(COMM_AREA_PTR) Options(Main)`.
  - Initialises working-storage header fields (`WS_TRANSID`, `WS_TERMID`, `WS_TASKNUM`) from the EIB (CICS Executive Interface Block) fields `EIBTRNID`, `EIBTRMID`, and `EIBTASKN`.
  - **COMMAREA validation — zero-length guard:** Checks `EIBCALEN = 0`. If no COMMAREA was passed, it calls `WRITE_ERROR_MESSAGE` to log the error, then issues `EXEC CICS ABEND ABCODE('LGCA') NODUMP` to terminate abnormally.
  - Initialises `CA_RETURN_CODE` to `'00'` (success), saves the COMMAREA pointer length into `WS_CALEN`, and copies `CA_CUSTOMER_NUM` and `CA_POLICY_NUM` into error-message fields (`EM_CUSNUM`, `EM_POLNUM`) for diagnostic use.
  - **Policy-type dispatch and COMMAREA length check (`SELECT` on `CA_REQUEST_ID`):**
    - `'01UEND'` (endowment update): Computes the minimum required COMMAREA length as `WS_CA_HEADER_LEN + WS_FULL_ENDOW_LEN`. If `EIBCALEN` is less than this, sets `CA_RETURN_CODE = '98'` and returns immediately to the caller.
    - `'01UHOU'` (house update): Same pattern using `WS_FULL_HOUSE_LEN`.
    - `'01UMOT'` (motor update): Same pattern using `WS_FULL_MOTOR_LEN`.
    - `OTHERWISE`: Sets `CA_RETURN_CODE = '99'` to signal an unsupported request type; does not return early, allowing the call to `UPDATE_POLICY_DB2_INFO` to still proceed.
  - Calls `UPDATE_POLICY_DB2_INFO` to delegate the actual database update.
  - Issues `EXEC CICS RETURN` to hand control back to the caller.

- **UPDATE_POLICY_DB2_INFO**
  - A short internal procedure that delegates the Db2 update to the backend program `LGUPDB01`.
  - Issues `EXEC CICS LINK Program('LGUPDB01') Commarea(COMM_AREA_RAW) LENGTH(32500)`, passing the full 32,500-byte raw COMMAREA to the linked program. No conditional logic is present; the procedure always performs the link unconditionally.
  - On return from `LGUPDB01`, control flows back to the main procedure body, which then issues `EXEC CICS RETURN`.

- **WRITE_ERROR_MESSAGE**
  - A diagnostic utility procedure that formats and writes error information to a CICS Transient Data Queue (TDQ) via the `LGSTSQ` logging program.
  - Calls `EXEC CICS ASKTIME ABSTIME(WS_ABSTIME)` to capture the current absolute time, then `EXEC CICS FORMATTIME` to format it into `WS_DATE` (MM/DD/YYYY) and `WS_TIME` (HH:MM:SS), which are copied into `EM_DATE` and `EM_TIME` in the `ERROR_MSG` structure.
  - Issues `EXEC CICS LINK PROGRAM('LGSTSQ') COMMAREA(ERROR_MSG)` to write the formatted error message (containing date, time, program name `LGUPOL01`, customer number, policy number, SQL request name, and SQLCODE) to the TDQ.
  - **Conditional COMMAREA dump:** If `EIBCALEN > 0`, writes up to 90 bytes of the raw COMMAREA content to the TDQ as a second message via another `EXEC CICS LINK PROGRAM('LGSTSQ') COMMAREA(CA_ERROR_MSG)` call:
    - If `EIBCALEN < 91`: copies only the actual COMMAREA bytes (`LEFT(COMM_AREA_RAW, EIBCALEN)`) into `CA_DATA` before linking.
    - Otherwise: copies the first 90 bytes (`LEFT(COMM_AREA_RAW, 90)`) into `CA_DATA` before linking.

## 6. Dependencies

The source code of the program is completely read and analyzed!

Let's summarize what we have found from the source code and the context in AGENTS.md:

### 6.1 Logical Categories of Dependencies for LGUPOL01.pli

1. **Input Parameters and Data Structures (CICS COMMAREA)**
   - The program is a CICS main program that expects an input parameter `COMM_AREA_PTR` passed as a pointer (`Proc(COMM_AREA_PTR) Options(Main)`).
   - The program maps this pointer to the logical data structure `COMM_AREA` defined in the include file `LGCMAREA` (`plinc/LGCMAREA.inc`).
   - The COMMAREA contains:
     - `CA_REQUEST_ID`: Used to determine the policy type to update (`01UEND` for Endowment, `01UHOU` for House, and `01UMOT` for Motor).
     - `CA_CUSTOMER_NUM` and `CA_POLICY_NUM`: Saved for tracking and error reporting.
     - Specific policy-type structures used to transport update details:
       - `CA_ENDOWMENT` for endowment policies.
       - `CA_HOUSE` for house policies.
       - `CA_MOTOR` for motor policies.
     - `CA_RETURN_CODE`: Used to indicate the execution result back to the caller (e.g., `'00'` for success, `'98'` for COMMAREA too short, `'99'` for unsupported requests).

2. **External Program and Module Invocations**
   - **`LGUPDB01`**: A Db2 database update module that is invoked via `EXEC CICS LINK PROGRAM('LGUPDB01')` passing the `COMM_AREA_RAW` of size 32,500 bytes. This program performs the actual database update operations.
   - **`LGSTSQ`**: An error queue writing module, invoked via `EXEC CICS LINK PROGRAM('LGSTSQ')` to log error details stored in `ERROR_MSG` and `CA_ERROR_MSG`.

3. **CICS System APIs and Environmental Dependencies**
   - **`EIB` (Execution Interface Block) fields**:
     - `EIBCALEN`: Used to validate that the communication area has been received and is of sufficient length to support the requested update type.
     - `EIBTRNID`: Used to capture the transaction ID (`WS_TRANSID`).
     - `EIBTRMID`: Used to capture the terminal ID (`WS_TERMID`).
     - `EIBTASKN`: Used to capture the task number (`WS_TASKNUM`).
   - **CICS Commands**:
     - `EXEC CICS ABEND`: Executed with abend code `'LGCA'` if no COMMAREA is received (`EIBCALEN = 0`).
     - `EXEC CICS RETURN`: Executed to return control back to the invoking program or transaction.
     - `EXEC CICS ASKTIME`: Executed to obtain the system absolute time (`WS_ABSTIME`) for error logging.
     - `EXEC CICS FORMATTIME`: Executed to format the absolute time into readable date (`WS_DATE`) and time (`WS_TIME`) variables.

4. **Included Source Files (Copybooks / Header Files)**
   - **`LGCMAREA` (`plinc/LGCMAREA.inc`)**: Contains the definition of the global communication area structure (`COMM_AREA`) used for inter-program communication.

5. **Configuration and Environment Values**
   - **CICS Transaction Environment**: Must run under a CICS region with the target transactions and programs defined.
   - **Expected Program Constants**: Uses local variables / policy length definitions (`WS_CUSTOMER_LEN`, `WS_POLICY_LEN`, `WS_ENDOW_LEN`, `WS_HOUSE_LEN`, `WS_MOTOR_LEN`, `WS_FULL_ENDOW_LEN`, `WS_FULL_HOUSE_LEN`, `WS_FULL_MOTOR_LEN`, `WS_SUMRY_ENDOW_LEN`) to validate that incoming COMMAREA lengths match the policy layouts.

Let's double-check all the details.
Is there any database dependency directly inside `LGUPOL01.pli`? No. The AGENTS.md document says: "Orchestrators contain no SQL. LGUPOL01 -> LGUPDB01 (Db2). It also links LGUPVS01." Wait, does it link `LGUPVS01`? Looking closely at the code of `LGUPOL01.pli`, it does NOT call `LGUPVS01`. It only links `LGUPDB01` (under `UPDATE_POLICY_DB2_INFO: PROCEDURE`). Oh, the AGENTS.md says "Update policy flow: LGUPOL01 -> LGUPDB01 (Db2). It also links LGUPVS01." But that might be a general description of the update policy flow or maybe `LGUPDB01` links `LGUPVS01`, or maybe the PL/I version only links `LGUPDB01`. Looking at the actual source code of `LGUPOL01.pli`, there is only a call to `LGUPDB01` and `LGSTSQ`. So we will accurately document only what is in the actual source code of `LGUPOL01.pli` and note the context.

Let's write down the response in bulleted list format as required:
- Grouped into logical categories.
- Starting with "-" and proper indentation.
- No numbered bullet points.
- No section heading (the user will provide it).
- Directly outputting the section content (no introduction, no conversational preamble).

Let's format the output beautifully.### Input Parameters and Data Structures
- **CICS COMMAREA (`COMM_AREA_PTR`)**
  - The program is invoked as a CICS main program with a pointer to the Communication Area (`COMM_AREA_PTR`). It is mapped to the `COMM_AREA` structure defined in the `LGCMAREA` include.
- **`LGCMAREA` structure fields**
  - `CA_REQUEST_ID`: An input field used to identify the requested update type (`01UEND` for Endowment, `01UHOU` for House, or `01UMOT` for Motor) and validate that the passed COMMAREA is long enough.
  - `CA_CUSTOMER_NUM`: An input identifier representing the customer, used for validation and passed to downstream database update processes.
  - `CA_POLICY_NUM`: An input identifier representing the policy, used for validation and passed to downstream database update processes.
  - `CA_RETURN_CODE`: An output code returned to the caller (`'00'` for success, `'98'` if the COMMAREA is too short, `'99'` for unsupported requests).
  - Specific Policy Substructures:
    - `CA_ENDOWMENT`: Holds the endowment-specific update fields (e.g., `CA_E_TERM`, `CA_E_SUM_ASSURED`, `CA_E_LIFE_ASSURED`).
    - `CA_HOUSE`: Holds the house-specific update fields (e.g., `CA_H_PROPERTY_TYPE`, `CA_H_VALUE`).
    - `CA_MOTOR`: Holds the motor-specific update fields (e.g., `CA_M_MAKE`, `CA_M_VALUE`, `CA_M_PREMIUM`).

### 6.2 Included Source Files
- **`LGCMAREA` (`plinc/LGCMAREA.inc`)**
  - A PL/I include file that declares the structure of the communication area. It defines the layout for incoming requests, including common policy headers, customer data, and policy-type-specific sub-structures (Endowment, House, Motor, Commercial, and Claim).

### 6.3 External Program and Module Invocations
- **`LGUPDB01`**
  - An external DB2 update program invoked via `EXEC CICS LINK` to perform the actual database updates for the requested policy. The program receives the 32,500-byte communication area as its payload.
- **`LGSTSQ`**
  - An error logging utility program invoked via `EXEC CICS LINK` to write diagnostic messages and a portion of the communication area to a CICS Transient Data Queue (TDQ) or Temporary Storage Queue (TSQ) when failures occur.

### 6.4 CICS System APIs and Runtime Environment
- **CICS Execution Interface Block (EIB)**
  - `EIBCALEN`: Used to verify that a non-empty COMMAREA of sufficient length was received.
  - `EIBTRNID`, `EIBTERMID`, and `EIBTASKN`: Captured and stored in local variables to enrich runtime error messages with transaction ID, terminal ID, and task number information.
- **CICS Services**
  - `EXEC CICS ABEND`: Issued with the code `'LGCA'` when the program is invoked without an input COMMAREA.
  - `EXEC CICS RETURN`: Used to return execution control back to the CICS invoking transaction or program.
  - `EXEC CICS ASKTIME` and `EXEC CICS FORMATTIME`: Used to retrieve and format the current system date and time for diagnostic logging within the `WRITE_ERROR_MESSAGE` procedure.

## 7. Constraints

### 7.1 Commarea presence and availability constraints
- A commarea must be supplied by the caller.
  - The program tests `EIBCALEN = 0`.
  - If no commarea is present, it sets `EM_MSG_TEXT` to ' NO COMMAREA RECEIVED' and calls `WRITE_ERROR_MESSAGE`.
  - It then abends the task with `EXEC CICS ABEND ABCODE('LGCA') NODUMP`, so no dump is produced and no further processing occurs.
- The return code is reset only after the commarea is confirmed to exist.
  - `CA_RETURN_CODE` is set to '00' after the `EIBCALEN` check.
  - The commarea length and address are saved in `WS_CALEN` and `WS_ADDR_DFHCOMMAREA`.
  - The customer and policy numbers are saved in `EM_CUSNUM` and `EM_POLNUM` for any later error message.
- The commarea is addressed through the based structure `COMM_AREA` (pointer `COMM_AREA_PTR`).
  - The raw area is 32500 bytes.
  - The request-specific area is defined as a union, so only one policy layout (endowment, house, motor, commercial or claim) is valid per request.

### 7.2 Request type validation constraints
- `CA_REQUEST_ID` must be one of the supported 6-character codes.
  - '01UEND' is an endowment policy update.
  - '01UHOU' is a house policy update.
  - '01UMOT' is a motor policy update.
  - The check is a `SELECT (CA_REQUEST_ID)` with three `WHEN` branches.
  - Any other value goes to `OTHERWISE`, which sets `CA_RETURN_CODE = '99'` (unsupported request).
- Only these three policy types can be updated by this program.
  - The commercial and claim layouts exist in the commarea definition but have no branch.
  - Requests for them fall into the unsupported (`'99'`) path.

### 7.3 Commarea length constraints
- The commarea must be at least as long as the minimum for the requested policy type.
  - The minimum is the header length (`WS_CA_HEADER_LEN` = 28) plus a type-specific full length.
  - The check is `EIBCALEN < WS_REQUIRED_CA_LEN`.
  - Endowment: `WS_FULL_ENDOW_LEN` = 124, so the minimum is 152 bytes.
  - House: `WS_FULL_HOUSE_LEN` = 130, so the minimum is 158 bytes.
  - Motor: `WS_FULL_MOTOR_LEN` = 137, so the minimum is 165 bytes.
- A commarea that is too short is rejected.
  - The program sets `CA_RETURN_CODE = '98'`.
  - It issues `EXEC CICS RETURN` immediately.
  - The database update program is not called.
- The length calculation excludes variable-length data.
  - For endowment, the minimum does not allow for the VARCHAR field.
  - `WS_VARY_FIELD` is defined with a 3900-character maximum but is not used in the validation.
- There is no upper-bound check on `EIBCALEN`.
  - The link to the update program always passes a fixed length of 32500 (`LENGTH(32500)`).

### 7.4 Return code constraints
- `CA_RETURN_CODE` is a 2-digit numeric (`PIC '99'`) with these values set by this program.
  - '00' is the initial value on entry.
  - '98' means the commarea is too short.
  - '99' means the request ID is unsupported.
- Other return codes can only come from the called program, `LGUPDB01`.

### 7.5 Processing sequence and flow constraints
- The order of operations is fixed.
  - Capture the EIB values (transaction ID, terminal ID, task number).
  - Check that a commarea exists.
  - Initialize the return code and save the keys.
  - Validate the request ID and commarea length.
  - Link to `LGUPDB01`.
  - Return.
- A length failure short-circuits the flow.
  - `EXEC CICS RETURN` is issued before the database update is reached.
- An unsupported request ID does not stop processing.
  - The `OTHERWISE` branch sets `'99'` but does not return.
  - Control still reaches `CALL UPDATE_POLICY_DB2_INFO`.
  - `LGUPDB01` is therefore still linked to, and it may overwrite the return code.
- The database update is delegated to `LGUPDB01`.
  - The link is `EXEC CICS LINK Program('LGUPDB01')`.
  - It passes `COMM_AREA_RAW` with length 32500.
  - Business validation of the field values (dates, amounts, broker data) is not done here. Those values are passed through unchanged.
- The program does not check the outcome of the LINK.
  - There is no `RESP` or `RESP2` handling.
  - It returns to the caller regardless.

### 7.6 Data format constraints (implied by field definitions)
- Numeric fields are picture-defined and must hold digits only.
  - `CA_CUSTOMER_NUM`: 10 digits.
  - `CA_POLICY_NUM`: 10 digits.
  - `CA_BROKERID`: 10 digits.
  - `CA_PAYMENT`: 6 digits.
  - `CA_E_TERM`: 2 digits.
  - `CA_E_SUM_ASSURED`: 6 digits.
  - `CA_H_VALUE`: 8 digits.
  - `CA_M_VALUE`: 6 digits.
  - `CA_M_PREMIUM`: 6 digits.
- Character fields have fixed maximum lengths.
  - `CA_REQUEST_ID`: 6.
  - `CA_ISSUE_DATE` and `CA_EXPIRY_DATE`: 10 each.
  - `CA_BROKERSREF`: 10.
  - `CA_E_LIFE_ASSURED`: 31.
  - `CA_H_PROPERTY_TYPE`: 15.
  - `CA_M_MAKE`: 15.
- Broker ID, broker's reference and payment may be null in the database.
  - The indicator variables `IND_BROKERID`, `IND_BROKERSREF` and `IND_PAYMENT` are declared for them.
  - They are not used in this program.
- The program does not check date formats, date ordering (issue before expiry), numeric ranges or required values for these fields.

### 7.7 Error message and logging constraints
- Error messages are written through `WRITE_ERROR_MESSAGE`.
  - They are sent by `EXEC CICS LINK PROGRAM('LGSTSQ')`.
  - The message carries the date and time (from `ASKTIME`/`FORMATTIME`, MMDDYYYY format), the program name, the customer number and the policy number.
  - The message text is limited to 63 characters (`EM_MSG_TEXT`).
- The commarea dump written with the error is capped at 90 bytes.
  - It is written only if `EIBCALEN > 0`.
  - If `EIBCALEN < 91`, only the available `EIBCALEN` bytes are written.
  - Otherwise the first 90 bytes are written.
  - `CA_DATA` is a 90-character field, which enforces this cap.
- Error logging is invoked only for the no-commarea condition.
  - The length failure ('98') and unsupported request ('99') cases do not log a message. They only set the return code.
- The `LENGTH(STG(CA_ERROR_MSG))` argument is used for both LINK calls to `LGSTSQ`.
  - The first call passes `ERROR_MSG`, which has a different size from `CA_ERROR_MSG`.
  - The length passed to `LGSTSQ` is therefore tied to the size of `CA_ERROR_MSG`, not `ERROR_MSG`.

### 7.8 Resource and size limits
- The commarea is capped at 32500 bytes (`COMM_AREA_RAW CHAR(32500)`, and the `LINK` length of 32500).
- The `LGUPDB01` target program name is fixed in `LGUPDB1` (CHAR(8) 'LGUPDB01').
- The maximum data written to the error queue per commarea dump is 90 bytes.

## 8. Error Handling

### 8.1 Input and Communication Area Validation
- Missing Communication Area Check:
  - The program validates whether a communication area was supplied by checking if the communication area length in the EXEC Interface Block equals zero.
  - If no communication area is passed, an error message is formatted indicating that no commarea was received, the error logging routine is invoked, and the program terminates abnormally with an explicit transaction ABEND code.
- Policy Request Type Validation:
  - The program evaluates the request identifier field in the communication area using a selection construct to ensure it matches one of the supported policy update operation codes (endowment, house, or motor policy update).
  - If an unrecognized or unsupported request identifier is encountered, the selection defaults to the alternate branch, sets the commarea return code to indicate an error condition, and bypasses further update processing before returning to the caller.
- Communication Area Length Validation:
  - For each valid request type, the program computes the minimum required communication area length by adding the fixed communication area header length to the full length specific to that policy type.
  - It compares the received communication area length against the calculated required length.
  - If the received communication area is shorter than the required minimum, the program sets the commarea return code to signify an insufficient communication area length and immediately terminates execution via a CICS return command.

### 8.2 Logging and Notification Handling
- Error Message Formatting and Transient Logging:
  - An internal error logging procedure captures system execution details, including the transaction ID, terminal ID, task number, and current timestamp using CICS time and date services.
  - The formatted error message incorporates identification data such as the program name, customer number, and policy number.
  - The message is written to a CICS temporary storage queue by linking to an external error logging utility program.
- Commarea Payload Logging:
  - When communication area data is present during an error, the procedure extracts up to the first ninety bytes (or fewer if the length is under ninety bytes) into a secondary logging structure.
  - This payload snippet is also forwarded to the logging subprogram to assist in post-failure diagnostics.

### 8.3 Downstream Error Delegation
- Delegation to Database Update Backend:
  - Validated requests are delegated via a CICS link command to the Db2 update backend program, passing the communication area buffer.
  - Status updates, database exceptions, and SQL operation outcomes are managed and populated into the communication area return code by the linked subprogram before control returns to the orchestrator.

## 9. Examples

### 9.1 Example 1 — Update a Motor Policy (Happy Path)

**Input COMMAREA fields:**

| Field | Value |
|---|---|
| `CA_REQUEST_ID` | `'01UMOT'` |
| `CA_CUSTOMER_NUM` | `0000001234` |
| `CA_POLICY_NUM` | `0000009876` |
| `CA_ISSUE_DATE` | `01/01/2020` |
| `CA_EXPIRY_DATE` | `01/01/2025` |
| `CA_BROKERID` | `0000000042` |
| `CA_BROKERSREF` | `BR-0099` |
| `CA_PAYMENT` | `001200` |
| `CA_M_MAKE` | `Ford` |
| `CA_M_VALUE` | `015000` |
| `CA_M_PREMIUM` | `000850` |
| COMMAREA length | ≥ 165 bytes (`WS_CA_HEADER_LEN(28) + WS_FULL_MOTOR_LEN(137)`) |

**Expected output:**

- `CA_RETURN_CODE` = `'00'`
- The full 32 500-byte COMMAREA is forwarded via `EXEC CICS LINK` to `LGUPDB01`, which performs the actual Db2 UPDATE for the MOTOR table.
- Control returns to the calling program with `CA_RETURN_CODE = '00'`.

**How the logic produces this result:**

1. `EIBCALEN > 0`, so no abend is issued and `CA_RETURN_CODE` is initialised to `'00'`.
2. The `SELECT (CA_REQUEST_ID)` statement matches `'01UMOT'`. The required COMMAREA length is calculated as `WS_CA_HEADER_LEN (28) + WS_FULL_MOTOR_LEN (137) = 165`.
3. Because `EIBCALEN ≥ 165`, the length check passes and no `'98'` return code is set.
4. `UPDATE_POLICY_DB2_INFO` is called, which issues `EXEC CICS LINK Program('LGUPDB01')` passing the entire COMMAREA (32 500 bytes).
5. After `LGUPDB01` completes, `EXEC CICS RETURN` hands control back to the caller.

---

### 9.2 Example 2 — Update a House Policy with a Short COMMAREA

**Input COMMAREA fields:**

| Field | Value |
|---|---|
| `CA_REQUEST_ID` | `'01UHOU'` |
| `CA_CUSTOMER_NUM` | `0000005678` |
| `CA_POLICY_NUM` | `0000001111` |
| `CA_H_PROPERTY_TYPE` | `Detached` |
| `CA_H_VALUE` | `00250000` |
| COMMAREA length | 50 bytes (deliberately short; required = `28 + 130 = 158`) |

**Expected output:**

- `CA_RETURN_CODE` = `'98'`
- `EXEC CICS RETURN` is issued immediately; `LGUPDB01` is **not** called.

**How the logic produces this result:**

1. `EIBCALEN = 50 > 0`, so no abend. `CA_RETURN_CODE` is initialised to `'00'`.
2. The `SELECT` matches `'01UHOU'`. Required length = `28 + 130 = 158`.
3. `EIBCALEN (50) < WS_REQUIRED_CA_LEN (158)`, so `CA_RETURN_CODE` is set to `'98'` and `EXEC CICS RETURN` exits the program without linking to `LGUPDB01`.

---

### 9.3 Example 3 — Unrecognised Request ID

**Input COMMAREA fields:**

| Field | Value |
|---|---|
| `CA_REQUEST_ID` | `'01UCOM'` (not a supported update type) |
| `CA_CUSTOMER_NUM` | `0000007777` |
| `CA_POLICY_NUM` | `0000002222` |
| COMMAREA length | 32 500 bytes |

**Expected output:**

- `CA_RETURN_CODE` = `'99'`
- `UPDATE_POLICY_DB2_INFO` is still called (the `OTHERWISE` branch sets the return code but does not `RETURN` early), forwarding the COMMAREA to `LGUPDB01`. The Db2 backend would then handle or reject the unsupported request.

**How the logic produces this result:**

1. `EIBCALEN > 0`; `CA_RETURN_CODE` initialised to `'00'`.
2. `SELECT (CA_REQUEST_ID)` falls into the `OTHERWISE` branch because `'01UCOM'` matches none of `'01UEND'`, `'01UHOU'`, or `'01UMOT'`. `CA_RETURN_CODE` is set to `'99'`.
3. Unlike the `'98'` length-check path, the `OTHERWISE` branch does **not** issue `EXEC CICS RETURN`. Execution falls through to `CALL UPDATE_POLICY_DB2_INFO`, so `LGUPDB01` is still linked to with the `'99'` return code already in the COMMAREA.

---

Generated by IBM Bob Premium Package for Z
