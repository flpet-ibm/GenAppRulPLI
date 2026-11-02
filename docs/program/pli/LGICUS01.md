## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
  - [2.1 Program Parameter and Communication Area](#21-program-parameter-and-communication-area)
  - [2.2 Caller-Supplied Request Fields](#22-caller-supplied-request-fields)
  - [2.3 CICS Runtime Context](#23-cics-runtime-context)
  - [2.4 External Data Sources](#24-external-data-sources)
  - [2.5 Fixed Configuration Values](#25-fixed-configuration-values)
- [3. Outputs](#3-outputs)
  - [3.1 Communication Area Outputs (Passed Back to Calling Program)](#31-communication-area-outputs-passed-back-to-calling-program)
  - [3.2 CICS System Outputs and Service Invocations](#32-cics-system-outputs-and-service-invocations)
- [4. Processing Logic](#4-processing-logic)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
- [7. Constraints](#7-constraints)
- [8. Error Handling](#8-error-handling)
- [9. Examples](#9-examples)
  - [9.1 Example 1 — Successful Customer Inquiry via DB2 (Standard Path)](#91-example-1--successful-customer-inquiry-via-db2-standard-path)
  - [9.2 Example 2 — Communication Area Too Short (Validation Failure)](#92-example-2--communication-area-too-short-validation-failure)
  - [9.3 Example 3 — MQ Hit Detected (Alternate Routing Path)](#93-example-3--mq-hit-detected-alternate-routing-path)

## 1. Purpose

LGICUS01 is a CICS business-logic program that handles requests to look up a customer's details in the insurance application. It checks that the caller supplied a communication area large enough to hold a customer inquiry. If the area is missing or too small, it logs a diagnostic error message that includes the customer number and a portion of the received data. It then reads a CICS temporary storage queue to see whether a message-queue "hit" record is present. If none is found, it passes the request to the DB2 inquiry program (LGICDB01) to retrieve the customer and policy information. If a hit is found, it routes the request to an alternate program instead. The result is returned to the caller in the shared communication area, along with a return code showing whether the request was accepted.

## 2. Inputs

### 2.1 Program Parameter and Communication Area
- COMM_AREA_PTR
  - Pointer parameter of the main procedure. It addresses the CICS communication area that the caller passes in, and all the communication area fields below are based on it.
- COMM_AREA_RAW
  - 32,500-character overlay of the whole incoming communication area.
  - Error handling uses up to 90 bytes of it as diagnostic content for the error log.

### 2.2 Caller-Supplied Request Fields
- CA_CUSTOMER_NUM
  - 10-digit customer number to look up.
  - It is the key for the inquiry, and the program copies it into the error message.
- CA_RETURN_CODE
  - Return code field in the communication area.
  - The program overwrites it: '00' means the request was accepted, and '98' means the communication area was too short.
- CA_NUM_POLICIES
  - Policy count field in the communication area. The program resets it to zero on entry.
  - The downstream inquiry populates it.
- CA_FIRST_NAME, CA_LAST_NAME, CA_EMAIL_ADDRESS, CA_POLICY_DATA
  - Customer detail fields inside the communication area.
  - They come in from the caller as part of the buffer and are filled in by the downstream DB2 inquiry program (LGICDB01) for the response.

### 2.3 CICS Runtime Context
- EIBCALEN
  - Length of the communication area received.
  - A value of 0 triggers the "NO COMMAREA RECEIVED" error and an abend.
  - A value below the required minimum (customer length 72 plus header/trailer 18) returns code '98'.
- EIBTRNID, EIBTRMID, EIBTASKN
  - Transaction ID, terminal ID and task number from the CICS Execute Interface Block (EIB).
  - They are copied into the runtime and debug header fields.

### 2.4 External Data Sources
- CICS temporary storage queue named by MQ_CONTROL ('GENAWMQC')
  - It is read item by item into MQ_READ_RECORD.
  - A record whose first 6 characters are 'MQHIT=' sets MQ_HIT to 1.
  - MQ_HIT controls routing: 0 sends the request to LGICDB01, and 1 sends it to the alternate program 'AAAAAAAA'.
  - The queue's contents and its existence (the RESP value from READQ TS) decide the route.
- System date and time from CICS ASKTIME and FORMATTIME
  - Used only to timestamp error messages.

### 2.5 Fixed Configuration Values
- WS_CA_HEADERTRAILER_LEN (18) and WS_CUSTOMER_LEN (72)
  - Hard-coded constants that set the minimum acceptable communication area length.
  - They are not supplied externally.
- LGICDB01
  - Name of the downstream DB2 inquiry program that the program links to. It is a fixed value, not an input.

## 3. Outputs

### 3.1 Communication Area Outputs (Passed Back to Calling Program)

- `CA_RETURN_CODE`
  - Two-digit status code returned to the calling program. It is initialized to `'00'` (success) and set to `'98'` if the incoming communication area length (`EIBCALEN`) is less than the required minimum length (`WS_REQUIRED_CA_LEN`). Downstream linked programs may also update this field.
- `CA_NUM_POLICIES`
  - Three-digit count field initialized to `0` at the start of the transaction. It is updated by downstream inquiry programs (such as `LGICDB01`) to reflect the total number of policies found for the requested customer.
- `COMM_AREA`
  - The primary data structure shared between the calling program and downstream subprograms. In addition to status codes, specific subfields are modified or populated downstream based on the inquiry execution:
    - `CA_CUSTOMER_REQUEST`: Holds the retrieved customer profile details, including `CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_DOB`, `CA_HOUSE_NAME`, `CA_HOUSE_NUM`, `CA_POSTCODE`, `CA_PHONE_MOBILE`, `CA_PHONE_HOME`, `CA_EMAIL_ADDRESS`, and `CA_POLICY_DATA`.

### 3.2 CICS System Outputs and Service Invocations

- `EXEC CICS LINK PROGRAM(LGICDB01)` / `EXEC CICS LINK PROGRAM('AAAAAAAA')`
  - Transfers control and passes `COMM_AREA` (length 32500) to the target subprogram to execute customer retrieval. If `MQ_HIT` is `0`, control is transferred to `LGICDB01`; otherwise, it links to `'AAAAAAAA'`.
- `EXEC CICS LINK PROGRAM('LGSTSQ')`
  - Invoked during error handling within `WRITE_ERROR_MESSAGE` to log failure information:
    - Passes `ERROR_MSG` containing timestamp information (`EM_DATE`, `EM_TIME`), program identifier (`LGICUS01`), customer number (`EM_CUSNUM`), and error description (`EM_MSG_TEXT` or SQL diagnostic fields).
    - Passes `CA_ERROR_MSG` containing up to 90 bytes of raw input communication area data (`CA_DATA` / `COMM_AREA_RAW`) to assist with transaction diagnostics.
- `EXEC CICS ABEND`
  - Issues an abnormal termination with abend code `'LGCA'` and suppresses dumps (`NODUMP`) when the program is invoked without a communication area (`EIBCALEN = 0`).
- `EXEC CICS RETURN`
  - Terminates execution and returns control along with the populated or updated `COMM_AREA` to the invoking CICS program.

## 4. Processing Logic

- **Initialisation and run-time identification**
  - The program is a CICS main procedure, `LGICUS01`, entered with a pointer to the communication area (`COMM_AREA_PTR`). The communication area is based on that pointer and is 32,500 bytes long.
  - It copies the transaction ID, terminal ID and task number from the CICS EIB fields (`EIBTRNID`, `EIBTRMID`, `EIBTASKN`) into the `WS_HEADER` work structure. This structure starts with the eyecatcher `LGICUS01------WS`, which helps identify the program in a dump.
  - Working fields hold the error message layout, the temporary storage queue name `GENAWMQC` (`MQ_CONTROL`), the DB2-layer program name `LGICDB01`, and the length constants.
    - `WS_CA_HEADERTRAILER_LEN` is 18.
    - `WS_CUSTOMER_LEN` is 72.
    - `MQ_HIT` starts at 0.

- **Communication area validation**
  - **No communication area supplied**
    - If `EIBCALEN = 0`, the program sets `EM_MSG_TEXT` to " NO COMMAREA RECEIVED".
    - It calls `WRITE_ERROR_MESSAGE`, then abends the task with `EXEC CICS ABEND ABCODE('LGCA') NODUMP`.
    - This is the only path that ends in an abend. The program does not return after it.
  - **Default response values**
    - `CA_RETURN_CODE` is set to '00' and `CA_NUM_POLICIES` to 0, so the response starts clean.
    - `EIBCALEN` is saved in `WS_CALEN`, and `COMM_AREA_PTR` is saved in `WS_ADDR_DFHCOMMAREA`.
  - **Minimum length check**
    - `WS_REQUIRED_CA_LEN` is computed as `WS_CUSTOMER_LEN + WS_CA_HEADERTRAILER_LEN`, which is 72 + 18 = 90.
    - If `EIBCALEN` is less than 90, `CA_RETURN_CODE` is set to '98' and the program ends at once with `EXEC CICS RETURN`. No lookup is attempted.
  - **Diagnostic preparation**
    - If the length is sufficient, `CA_CUSTOMER_NUM` is copied into `EM_CUSNUM` of the error message structure. Any later error message therefore carries the customer number.

- **Main inquiry flow**
  - The program calls the internal procedure `GET_CUSTOMER_INFO` once.
  - When that procedure returns, the program issues `EXEC CICS RETURN`. The caller gets the communication area as updated by the downstream program.
  - The program does not check the downstream result. `CA_RETURN_CODE` and the customer data returned from the linked program pass straight back to the caller.

- **Customer information retrieval (`GET_CUSTOMER_INFO`)**
  - **Queue check for an `MQHIT=` record**
    - `MQ_HIT` is reset to 0.
    - It reads item 1 of the temporary storage queue named in `MQ_CONTROL` ('GENAWMQC') into `MQ_READ_RECORD`, which is 80 characters. The response goes into `WS_RESP`.
    - If the first read returns NORMAL, a loop runs `DO UNTIL (WS_RESP > 0)`.
      - Each pass reads the next record from the queue with `READQ TS ... NEXT`.
      - If the read is NORMAL and the first 6 characters of the record equal 'MQHIT=', `MQ_HIT` is set to 1.
      - The loop does not stop when a match is found. It reads every remaining record until a non-zero response, normally end of queue (ITEMERR), so one matching record anywhere in the queue is enough.
    - If the first read is not NORMAL, for example because the queue does not exist, the loop is skipped and `MQ_HIT` stays 0.
  - **Routing decision**
    - If `MQ_HIT = 0`, the program issues `EXEC CICS LINK PROGRAM(LGICDB01)`. The DB2 inquiry layer receives the 32,500-byte communication area and fills in the customer details, which include name, address, contact data, number of policies and policy data.
    - If `MQ_HIT = 1`, the program links to the alternate program 'AAAAAAAA' with the same communication area and length.

- **Error logging (`WRITE_ERROR_MESSAGE`)**
  - This procedure is currently called only when no communication area was received.
  - **Timestamp**
    - `EXEC CICS ASKTIME` fetches the current absolute time into `WS_ABSTIME`.
    - `EXEC CICS FORMATTIME` converts it into `WS_DATE` (MMDDYYYY format) and `WS_TIME`.
    - Both values are copied into `EM_DATE` and `EM_TIME` of the `ERROR_MSG` structure.
  - **Main error record**
    - The program links to `LGSTSQ` and passes `ERROR_MSG` as the communication area. The length is `STG(ERROR_MSG)`.
    - `LGSTSQ` is the logging program that writes to the error queues.
    - `ERROR_MSG` holds the date, time, program name and message text. Its `EM_VARIABLE` area is a union, so it can hold either free text or a customer number, policy number and SQL code layout.
  - **Communication area dump record**
    - The communication area is copied to the log only when `EIBCALEN > 0`. In the `EIBCALEN = 0` case this block is skipped.
    - If `EIBCALEN < 91`, `CA_DATA` takes the left-justified first `EIBCALEN` bytes of `COMM_AREA_RAW`. The `LEFT` result is padded to 90 characters.
    - Otherwise `CA_DATA` takes the first 90 bytes.
    - In both cases the program links to `LGSTSQ` a second time and passes `CA_ERROR_MSG`, which is the 'COMMAREA=' prefix followed by the 90 data bytes.
  - The procedure then returns to its caller.

- **External interactions**
  - **CICS temporary storage queue `GENAWMQC`:** read-only, sequential reads, used to decide how to route the request.
  - **CICS program links**
    - `LGICDB01` is the DB2 inquiry layer. It is the default route and receives the communication area.
    - `AAAAAAAA` is the alternate program, used when an `MQHIT=` record is found.
    - `LGSTSQ` is the error logging program.
  - **CICS services:** `ASKTIME`, `FORMATTIME`, `ABEND` and `RETURN`.
  - The program itself contains no direct DB2, file or MQ calls. All database access is delegated to `LGICDB01`.

## 5. Paragraphs

- **LGICUS01 (main procedure body)**: This is the entry point, declared `Options(Main)`. It receives the CICS communication area through `COMM_AREA_PTR` and drives the customer inquiry.
  - Operations:
    - Copies `EIBTRNID`, `EIBTRMID` and `EIBTASKN` into `WS_TRANSID`, `WS_TERMID` and `WS_TASKNUM` for run-time diagnostics.
    - Sets `CA_RETURN_CODE` to '00' and `CA_NUM_POLICIES` to 0.
    - Saves `EIBCALEN` in `WS_CALEN` and the commarea address in `WS_ADDR_DFHCOMMAREA`.
    - Computes `WS_REQUIRED_CA_LEN` as `WS_CUSTOMER_LEN` (72) plus `WS_CA_HEADERTRAILER_LEN` (18), which is 90.
    - Moves `CA_CUSTOMER_NUM` into `EM_CUSNUM` so the customer number appears in any error message.
    - Calls `GET_CUSTOMER_INFO` to do the lookup.
    - Issues `EXEC CICS RETURN` to hand control back to the caller.
  - Control flow:
    - If `EIBCALEN = 0`, no commarea was passed. It sets `EM_MSG_TEXT` to ' NO COMMAREA RECEIVED', calls `WRITE_ERROR_MESSAGE`, and abends with code 'LGCA' (`NODUMP`).
    - If `EIBCALEN` is less than `WS_REQUIRED_CA_LEN`, it sets `CA_RETURN_CODE` to '98' and returns to CICS immediately without calling the lookup.
    - Otherwise it calls `GET_CUSTOMER_INFO` and then returns. The final `RETURN;` statement is never reached because `EXEC CICS RETURN` comes first.
  - External interactions:
    - CICS EIB fields are read.
    - `EXEC CICS ABEND` and `EXEC CICS RETURN` end the task.
    - It writes to the commarea, which is the interface to the caller.

- **GET_CUSTOMER_INFO**: This internal procedure decides where to send the inquiry. It checks the temporary storage queue named in `MQ_CONTROL` ('GENAWMQC') for an 'MQHIT=' record. It then links either to the DB2 inquiry program or to an alternate program.
  - Operations:
    - Resets `MQ_HIT` to 0.
    - Reads item 1 of the TS queue into `MQ_READ_RECORD`, with the CICS response saved in `WS_RESP`.
    - If that read succeeds, it reads the remaining items sequentially with `NEXT`, testing each one.
    - Sets `MQ_HIT` to 1 when a record read successfully has 'MQHIT=' in its first 6 characters (`SUBSTR(MQ_READ_RECORD,1,6)`).
    - After the queue scan, links to a program using the 32,500-byte `COMM_AREA`.
  - Control flow:
    - `IF (WS_RESP = DFHRESP(NORMAL))` runs the scan only if the first read succeeded. If the queue is missing or empty, `MQ_HIT` stays 0.
    - `DO UNTIL (WS_RESP > 0)` loops until a non-normal response, normally the end of the queue (ITEMERR). The test uses `WS_RESP > 0`, so it exits on any non-zero response.
    - `IF (MQ_HIT = 0)` links to `LGICDB01`, the DB2 inquiry layer. This is the normal path.
    - `ELSE` links to the placeholder program 'AAAAAAAA'. This is the alternate path when a message-queue hit was found.
  - External interactions:
    - `EXEC CICS READQ TS` on queue 'GENAWMQC', both for the first item and with `NEXT`.
    - `EXEC CICS LINK` to `LGICDB01` or 'AAAAAAAA', passing the commarea.
    - `LGICDB01` fills in the customer details, including `CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_EMAIL_ADDRESS` and `CA_POLICY_DATA`.

- **WRITE_ERROR_MESSAGE**: This internal procedure builds a time-stamped error record and sends it to the logging program. The record carries the program name, the customer number and the message text. It also logs part of the commarea for diagnosis.
  - Operations:
    - Gets the current time with `EXEC CICS ASKTIME` into `WS_ABSTIME`.
    - Formats it with `EXEC CICS FORMATTIME` into `WS_DATE` (MMDDYYYY) and `WS_TIME`.
    - Moves the formatted values into `EM_DATE` and `EM_TIME` in `ERROR_MSG`.
    - Links to 'LGSTSQ' with `ERROR_MSG` as the commarea, using `STG(ERROR_MSG)` as the length.
    - Copies up to 90 bytes of `COMM_AREA_RAW` into `CA_DATA` and links to 'LGSTSQ' again with `CA_ERROR_MSG`.
  - Control flow:
    - The commarea dump runs only when `EIBCALEN > 0`.
    - If `EIBCALEN < 91`, it copies `LEFT(COMM_AREA_RAW, EIBCALEN)`, which is the whole commarea.
    - Otherwise it copies the first 90 bytes with `LEFT(COMM_AREA_RAW, 90)`.
    - Both branches then perform the same link to 'LGSTSQ'.
    - The procedure ends with `RETURN`.
  - External interactions:
    - CICS time services (`ASKTIME`, `FORMATTIME`).
    - Up to two `EXEC CICS LINK` calls to 'LGSTSQ', which writes the error records to the queues.

## 6. Dependencies

**Included Definitions and Copybooks**
- LGCMAREA (`plinc/LGCMAREA.inc`)
  - Supplies the shared communication area layout, which is based on `COMM_AREA_PTR` and overlaid by `COMM_AREA_RAW`.
  - Defines the request and response fields the program reads and sets:
    - `CA_REQUEST_ID`
    - `CA_RETURN_CODE`
    - `CA_CUSTOMER_NUM`
    - `CA_NUM_POLICIES`
    - `CA_FIRST_NAME`
    - `CA_LAST_NAME`
    - `CA_EMAIL_ADDRESS`
    - `CA_POLICY_DATA`
  - Also defines the policy, security and claim layouts, which this program doesn't use.

**Input Parameters and Caller-Provided Data**
- `COMM_AREA_PTR` (procedure parameter)
  - It is the address of the CICS communication area (DFHCOMMAREA) passed by the calling program or transaction.
  - It is the only data source the program needs to run.
- `CA_CUSTOMER_NUM`
  - The customer number, supplied by the caller, that drives the inquiry.
  - It is copied into `EM_CUSNUM` for diagnostics.
- Required communication area size
  - `WS_CUSTOMER_LEN` (72) plus `WS_CA_HEADERTRAILER_LEN` (18) gives a minimum length of 90.
  - The caller must supply at least this many bytes.
  - If it supplies fewer, the program sets `CA_RETURN_CODE` to '98' and returns.
- `EIBCALEN`
  - The length of the communication area passed in.
  - A value of 0 triggers the error path.
  - The program also checks it against the required length.
  - The error handler uses it to decide how many bytes of the communication area to log.

**CICS Runtime Environment**
- Execute Interface Block (EIB) fields
  - `EIBTRNID`, `EIBTRMID` and `EIBTASKN` populate the runtime debug header (`WS_TRANSID`, `WS_TERMID`, `WS_TASKNUM`).
  - `EIBCALEN` is described above.
- CICS command-level API (`EXEC CICS`), used for:
  - `RETURN`, to end the transaction.
  - `ABEND ABCODE('LGCA') NODUMP`, to terminate abnormally when no communication area is received.
  - `ASKTIME` and `FORMATTIME`, to obtain and format the current date and time (MMDDYYYY and time) for error messages.
  - `READQ TS`, to read the temporary storage queue.
  - `LINK PROGRAM`, to call other programs.
- `DFHRESP(NORMAL)`
  - Used to test the response code returned by the `READQ TS` commands.

**Temporary Storage Queue**
- `GENAWMQC` (held in `MQ_CONTROL`)
  - A CICS TS queue read by `READQ TS`, first at item 1 and then with `NEXT` until the response is not normal.
  - Each record is received into `MQ_READ_RECORD`.
  - A record starting with 'MQHIT=' sets `MQ_HIT` to 1, which controls how the request is routed.
  - The queue must be defined and available in the CICS region. If it is absent or empty, `MQ_HIT` stays 0 and the request goes to the DB2 layer.

**External Programs Invoked (CICS LINK)**
- `LGICDB01` (held in the `LGICDB01` variable)
  - The DB2 inquiry layer that retrieves the customer details and policies.
  - It is linked when no MQHIT record is found.
  - It receives `COMM_AREA` with a length of 32500 and fills in the customer fields in place.
- `AAAAAAAA`
  - A placeholder alternate program, linked when an MQHIT record is found.
  - It receives the same `COMM_AREA` with a length of 32500.
  - It must exist in the CICS program definitions if that path is used.
- `LGSTSQ`
  - The error logging program.
  - It is linked from `WRITE_ERROR_MESSAGE` to write the formatted error message (`ERROR_MSG`) to the error queues.
  - It is linked a second time with `CA_ERROR_MSG`, which carries up to 90 bytes of the incoming communication area.

**Indirect Data Stores (via Linked Programs)**
- DB2 database
  - It is reached only through `LGICDB01`.
  - This program has no embedded SQL of its own.
  - The `EM_SQLREQ` and `EM_SQLRC` error message fields are defined but not populated here.
- Error logging queues (TD/TS)
  - They are written by `LGSTSQ`.
  - They record the date, time, program name, customer number and communication area content.

**Configuration Constants and Internal Settings**
- `ATRANID` = 'DSC1'
  - It is declared as a transaction ID constant but not used in the logic.
- `WS_MODULENAME`
  - It is declared but not used.
- Hard-coded program and queue names
  - `LGICDB01`
  - `AAAAAAAA`
  - `LGSTSQ`
  - `GENAWMQC`
  - They can't be changed through environment variables or external configuration. They are fixed in the source.
- Fixed lengths
  - 32500 bytes for the commarea passed on `LINK`.
  - 90 bytes for the commarea snippet logged on error.

## 7. Constraints

- Communication area presence and size
  - A communication area must be supplied.
    - The program tests `EIBCALEN = 0`.
    - If it is zero, it sets `EM_MSG_TEXT` to ' NO COMMAREA RECEIVED' and calls `WRITE_ERROR_MESSAGE`.
    - It then ends the task with `EXEC CICS ABEND ABCODE('LGCA') NODUMP`, so no dump is produced.
  - The communication area must meet a minimum length.
    - `WS_REQUIRED_CA_LEN` is computed as `WS_CUSTOMER_LEN` (72) plus `WS_CA_HEADERTRAILER_LEN` (18), which gives 90 bytes.
    - If `EIBCALEN < WS_REQUIRED_CA_LEN`, the program sets `CA_RETURN_CODE` to '98' and issues `EXEC CICS RETURN`.
    - No customer lookup is done when the area is too short.
    - A short area is reported only through the return code. No error message is written and no abend is issued.
  - The communication area has a fixed maximum layout of 32,500 bytes.
    - `COMM_AREA_RAW` is CHAR(32500).
    - The `LINK` commands pass `LENGTH(32500)` to the called programs.
    - `CA_POLICY_DATA` is CHAR(32267), which limits the policy data returned.

- Data format and field definitions
  - `CA_CUSTOMER_NUM` is PIC '9999999999', so it holds exactly 10 numeric digits.
  - `CA_RETURN_CODE` is PIC '99', so it holds a 2-digit numeric code.
    - '00' means success.
    - '98' means the communication area is too short.
  - `CA_NUM_POLICIES` is PIC '999', so it can report at most 999 policies.
  - Customer text fields have fixed maximum lengths.
    - `CA_FIRST_NAME` is CHAR(10).
    - `CA_LAST_NAME` is CHAR(20).
    - `CA_EMAIL_ADDRESS` is CHAR(100).
    - Longer values are truncated.
  - The request-specific section of the communication area is a UNION.
    - It overlays the customer, customer-security and policy layouts, so only one interpretation applies to a given request.
    - `CA_REQUEST_INITIALISER` is CHAR(32482).
  - The program does not check `CA_CUSTOMER_NUM` for numeric content or existence. It copies the value into `EM_CUSNUM` and passes it to the downstream program.

- Initialisation and state
  - `CA_RETURN_CODE` is set to '00' and `CA_NUM_POLICIES` is set to 0 at the start of every request.
    - This happens only after the zero-length check passes.
    - It happens before the length check.
  - `MQ_HIT` is reset to 0 at the start of `GET_CUSTOMER_INFO`, so routing is decided afresh on each invocation.
  - The program captures `EIBTRNID`, `EIBTRMID` and `EIBTASKN` into `WS_TRANSID`, `WS_TERMID` and `WS_TASKNUM` for diagnostics.
    - The communication area address and length are saved in `WS_ADDR_DFHCOMMAREA` and `WS_CALEN`.

- Sequencing
  - Processing follows a fixed order.
    - Capture the CICS environment identifiers.
    - Check for a zero-length communication area.
    - Initialise the return code and policy count.
    - Check the minimum length.
    - Copy the customer number into the error message structure.
    - Call `GET_CUSTOMER_INFO`.
    - Return to the caller with `EXEC CICS RETURN`.
  - Inside `GET_CUSTOMER_INFO`, the queue must be read before routing is chosen.
    - Item 1 is read first.
    - Only if that read succeeds are the remaining items read with `NEXT`.
    - The `LINK` happens only after `MQ_HIT` has been determined.
  - Inside `WRITE_ERROR_MESSAGE`, the steps are ordered.
    - `ASKTIME` and `FORMATTIME` run first, so the error message has a date and time.
    - The error message is linked to `LGSTSQ`.
    - The communication area dump is linked to `LGSTSQ` afterwards.

- Routing
  - The temporary storage queue named in `MQ_CONTROL` ('GENAWMQC') decides which program handles the inquiry.
    - Routing depends only on whether a record in the queue starts with 'MQHIT='. The check is `SUBSTR(MQ_READ_RECORD,1,6) = 'MQHIT='`, and it only applies to a read with `RESP` of `DFHRESP(NORMAL)`.
    - If `MQ_HIT = 0`, the request is linked to `LGICDB01`, the DB2 inquiry layer.
    - If `MQ_HIT = 1`, the request is linked to the alternate program 'AAAAAAAA'.
  - The queue is read as a whole.
    - If item 1 is read successfully, the `DO UNTIL (WS_RESP > 0)` loop reads each following record.
    - The loop ends on the first non-normal response, normally the end of the queue.
    - Any matching record in the queue sets the hit flag.
    - After a hit is found the loop does not stop early and keeps reading to the end of the queue.
  - If the first read of item 1 fails, for example because the queue does not exist or is empty, the program does not read further. `MQ_HIT` stays 0 and the request goes to `LGICDB01`.
  - Only the first 6 characters of each 80-character record are compared, and the comparison is case-sensitive.
  - The `LINK` commands do not use `RESP`, so a failure in the linked program is not handled in this program. CICS default condition handling applies.

- Error reporting
  - `WRITE_ERROR_MESSAGE` is called only for a missing communication area.
  - The error message carries the date, time, program name 'LGICUS01' and a text of at most 63 characters (`EM_MSG_TEXT`).
    - The message area is a UNION, so the text overlays the customer number, policy number and SQL fields.
  - Error messages are written by linking to program `LGSTSQ` with the `ERROR_MSG` structure.
  - Diagnostic communication area content is limited to 90 bytes (`CA_DATA` is CHAR(90)).
    - If `EIBCALEN` is above 0 and below 91, only `EIBCALEN` bytes are copied.
    - If `EIBCALEN` is 91 or more, exactly the first 90 bytes are copied.
    - The dump is written only if `EIBCALEN > 0`.
    - In the zero-length abend path this condition is false, so no communication area content is logged.
  - Date and time come from `ASKTIME` and `FORMATTIME`.
    - The date uses the `MMDDYYYY` format in a 10-character field.
    - The time is held in an 8-character field.

- Environment and runtime requirements
  - The program must run in CICS (`Options(Main)` with `EXEC CICS` commands).
  - The programs `LGICDB01`, 'AAAAAAAA' and `LGSTSQ` must be available to CICS.
  - The program always ends with `EXEC CICS RETURN`, which includes the early exit for a too-short communication area.

## 8. Error Handling

-   Input validation checks
    -   Missing communication area check
        -   The program tests whether the CICS-supplied communication area length (EIBCALEN) is zero at the start of processing, before it does any other work.
        -   If it is zero, the program sets the error message text to "NO COMMAREA RECEIVED". It then calls the WRITE_ERROR_MESSAGE routine to log the problem and issues a CICS ABEND with abend code LGCA, with no dump (NODUMP).
        -   This is a fail-fast check. The transaction ends abnormally because nothing useful can be done without input.
    -   Minimum communication area length check
        -   The program computes the minimum acceptable size (WS_REQUIRED_CA_LEN) as the customer data length (72) plus the header/trailer overhead (18).
        -   If EIBCALEN is smaller than that value, the communication area is too small to hold a valid customer inquiry request.
        -   In that case the program sets the return code (CA_RETURN_CODE) to '98' and issues a CICS RETURN. It does not call the downstream layers. The caller has to interpret the '98' value, because the program does not log this condition or abend.
-   Return code signaling to the caller
    -   CA_RETURN_CODE is set to '00' as the default once the communication area is known to be present. This is the success indicator that is expected to stay in place if processing completes normally.
    -   CA_RETURN_CODE is set to '98' for a communication area that is too short. This is the program's own error code, passed back through the communication area.
    -   CA_NUM_POLICIES is reset to zero at the start of each request, so the response never carries stale policy counts.
-   Abnormal termination
    -   The only abend in the program is the LGCA abend raised when no communication area is received.
    -   Because NODUMP is specified, the abend does not produce a transaction dump. The error is reported through the logged message and the abend code only.
-   Error logging and notification
    -   The WRITE_ERROR_MESSAGE procedure formats and writes diagnostics.
        -   It gets the current time with ASKTIME and formats it into date and time strings with FORMATTIME. It places these in the error message structure (EM_DATE, EM_TIME).
        -   The error message structure also carries the program name and either free-form message text or a customer number/policy number/SQL request/SQLCODE layout. The customer number field (EM_CUSNUM) is populated from CA_CUSTOMER_NUM before the customer lookup, so any later error message can include it.
        -   It sends the formatted message to the shared logging program LGSTSQ through a CICS LINK, passing the message structure as the communication area.
    -   Raw communication area capture
        -   If the communication area length is greater than zero, the routine also sends the contents of the incoming communication area (COMM_AREA_RAW) to LGSTSQ in a second message with the "COMMAREA=" prefix.
        -   If the length is under 91 bytes, it sends only the bytes actually received (EIBCALEN). Otherwise it sends the first 90 bytes. This prevents reading beyond the supplied data.
        -   In the no-communication-area case, this second step is skipped because the length is zero.
    -   The runtime context fields (transaction ID, terminal ID, task number) are captured at program start in WS_HEADER. This gives diagnostic context in dumps or traces.
-   Recovery and fallback logic
    -   Message queue routing fallback in GET_CUSTOMER_INFO
        -   The MQ_HIT flag is initialised to 0 on every call. If no hit is detected, the default path stays in effect and the request is linked to the DB2 inquiry layer program LGICDB01.
        -   The program first reads item 1 of the temporary storage queue GENAWMQC and checks the response code (WS_RESP) against NORMAL. If the read fails for any reason, such as the queue not existing or being empty, the program takes no corrective action. It does not read further, leaves MQ_HIT at 0, and continues down the default DB2 path. In effect, a missing or unreadable control queue is treated as "no hit" rather than as an error.
        -   If the first read succeeds, the program keeps reading the following records with the NEXT option. The loop stops when WS_RESP becomes greater than 0, which is the loop's exit condition. This normally happens at the end of the queue (for example, ITEMERR) and also on any other non-normal response. Both end the loop silently without an error message.
        -   The MQ_HIT flag is set only when a read is successful and the first six characters of the record equal "MQHIT=". If a hit is found, the request is redirected to an alternate program (hard-coded as 'AAAAAAAA') instead of LGICDB01.
    -   The queue-read response codes are captured with RESP rather than allowing the default CICS condition handling. A failure on the queue therefore cannot abend the transaction, and the program controls the outcome.
-   Gaps and notable absences
    -   The two CICS LINK commands in GET_CUSTOMER_INFO (to LGICDB01 or the alternate program) have no RESP or RESP2 checks. A failure in the linked program, such as PGMIDERR or an abend, is not trapped locally. It goes to CICS default handling, and the program does not write an error message.
    -   The calls to LGSTSQ in the error-logging routine also have no response checking. A failure to log is not detected.
    -   The program does not inspect the return code that the downstream program sets in the communication area after the LINK. Any error indication from the DB2 layer passes back to the caller unchanged.
    -   The error message structure has fields for customer number, policy number, SQL request and SQLCODE. In this program only the customer number is populated, and only the "no communication area" path writes a message. The SQL-related fields are unused here and appear intended for the DB2 layer or other programs.
    -   The short communication area case (return code '98') exits through a plain CICS RETURN without logging, so that rejection leaves no diagnostic record.

## 9. Examples

### 9.1 Example 1 — Successful Customer Inquiry via DB2 (Standard Path)

**Input:**
- `EIBCALEN` = 32500 (caller provides a full-sized communication area — well above the minimum of 90 bytes)
- `CA_REQUEST_ID` = `'LGICUS'`
- `CA_CUSTOMER_NUM` = `'0000010001'`
- TS queue `GENAWMQC` is **empty** (no records present, or no record beginning with `MQHIT=`)

**Processing:**
1. `EIBCALEN` is non-zero, so no abend occurs.
2. `CA_RETURN_CODE` is set to `'00'`; `CA_NUM_POLICIES` is set to `0`.
3. The minimum required length check passes (32500 ≥ 90).
4. `GET_CUSTOMER_INFO` is called. `MQ_HIT` starts at `0`.
5. `EXEC CICS READQ TS QUEUE('GENAWMQC')` returns a non-NORMAL response (queue is empty), so the `DO UNTIL` loop is never entered and `MQ_HIT` remains `0`.
6. Because `MQ_HIT = 0`, the program issues `EXEC CICS LINK PROGRAM('LGICDB01')`, passing the full 32,500-byte communication area.
7. LGICDB01 performs the DB2 lookup and populates `CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_NUM_POLICIES`, `CA_POLICY_DATA`, etc.

**Expected Output (communication area on return):**
| Field | Value (example) |
|---|---|
| `CA_RETURN_CODE` | `'00'` |
| `CA_CUSTOMER_NUM` | `'0000010001'` |
| `CA_FIRST_NAME` | `'John      '` |
| `CA_LAST_NAME` | `'Smith               '` |
| `CA_NUM_POLICIES` | `'003'` |
| `CA_POLICY_DATA` | *(serialised policy records from DB2)* |

---

### 9.2 Example 2 — Communication Area Too Short (Validation Failure)

**Input:**
- `EIBCALEN` = 50 (caller provides a communication area smaller than the required 90 bytes: `WS_CUSTOMER_LEN`(72) + `WS_CA_HEADERTRAILER_LEN`(18) = 90)
- `CA_CUSTOMER_NUM` = `'0000099999'`

**Processing:**
1. `EIBCALEN` is non-zero, so no abend occurs.
2. `CA_RETURN_CODE` is set to `'00'`; `CA_NUM_POLICIES` is set to `0`.
3. The length check fires: 50 < 90, so `CA_RETURN_CODE` is set to `'98'` and `EXEC CICS RETURN` is issued immediately.
4. `GET_CUSTOMER_INFO` is **never called**; no DB2 or MQ interaction occurs.

**Expected Output:**
| Field | Value |
|---|---|
| `CA_RETURN_CODE` | `'98'` |
| All other fields | Unchanged / uninitialised |

---

### 9.3 Example 3 — MQ Hit Detected (Alternate Routing Path)

**Input:**
- `EIBCALEN` = 32500
- `CA_CUSTOMER_NUM` = `'0000020002'`
- TS queue `GENAWMQC` contains two records; the second record begins with `'MQHIT='` (e.g., `'MQHIT=ACTIVE                                                          '`)

**Processing:**
1. Validation passes; `CA_RETURN_CODE` = `'00'`.
2. `GET_CUSTOMER_INFO` is called. `MQ_HIT` starts at `0`.
3. Item 1 of `GENAWMQC` is read successfully (NORMAL response), so the `DO UNTIL` loop begins.
4. The `NEXT` read returns the second record; its first 6 characters equal `'MQHIT='`, so `MQ_HIT` is set to `1`.
5. The loop ends when a subsequent `READQ TS NEXT` returns a non-NORMAL response (end of queue).
6. Because `MQ_HIT = 1`, the program issues `EXEC CICS LINK PROGRAM('AAAAAAAA')` instead of LGICDB01, passing the same 32,500-byte communication area to the alternate program.

**Expected Output:**
| Field | Value |
|---|---|
| `CA_RETURN_CODE` | `'00'` *(set before routing; the alternate program may modify it)* |
| `CA_CUSTOMER_NUM` | `'0000020002'` |
| Remaining fields | Populated by program `AAAAAAAA` |

---

Generated by IBM Bob Premium Package for Z
