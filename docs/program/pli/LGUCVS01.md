## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
- [3. Outputs](#3-outputs)
- [4. Processing Logic](#4-processing-logic)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
- [7. Constraints](#7-constraints)
- [8. Error Handling](#8-error-handling)
- [9. Examples](#9-examples)
  - [9.1 Example 1 — Successful Customer Update](#91-example-1--successful-customer-update)
  - [9.2 Example 2 — VSAM Read-for-Update Fails (customer not found / file unavailable)](#92-example-2--vsam-read-for-update-fails-customer-not-found--file-unavailable)
  - [9.3 Example 3 — VSAM Rewrite Fails](#93-example-3--vsam-rewrite-fails)

## 1. Purpose

This CICS PL/I program updates a customer record in the VSAM KSDS customer file (KSDSCUST). A calling program passes it a communication area that holds the customer number and the new customer details, such as name, date of birth, address, contact information and number of policies. The program uses the customer number as the key to lock the matching customer record for update. It then replaces that record with the data supplied by the caller. If the read or the rewrite fails, the program sets a return code in the communication area (81 for the read, 82 for the rewrite). It then records a time-stamped error message and a sample of the communication area data through the shared logging program (LGSTSQ). It finishes by abending the transaction so the failure is visible to the caller and the operations team.

## 2. Inputs

I will now analyze the extracted source code and any other details of LGUCVS01.pli.

The program `LGUCVS01` takes one parameter via its entry point: `COMM_AREA_PTR`. This is a pointer to `COMM_AREA` which is a `BASED` union mapping the CICS COMMAREA structure (`LGCMAREA`).

Let's look at what data from the COMMAREA and outside is used.
Inputs to the program include:
- `COMM_AREA_PTR` / `EIBCALEN` passed from CICS representing the communication area.
- Inside the COMMAREA, specific fields are read:
  - `CA_CUSTOMER_NUM`: Used as the key field (`RIDFLD`) in the CICS `READ` command to look up the customer record from VSAM KSDS. It is also used in `REWRITE` as the start of the buffer `FROM(CA_Customer_Num)` containing the updated customer record.
- VSAM KSDS File 'KSDSCUST' as an external data source:
  - The program performs `EXEC CICS READ FILE('KSDSCUST') UPDATE` to read the customer record, using `CA_Customer_Num` as the key.
  - The program then performs `EXEC CICS REWRITE FILE('KSDSCUST')` using `CA_Customer_Num` as the source of the rewrite buffer. (Notice that the rewrite buffer source is `FROM(CA_Customer_Num)`, which writes the updated customer record. Since `CA_Customer_Num` is in the `COMM_AREA` structure, it means the caller populated the rest of the customer fields directly within the commarea following `CA_Customer_Num`, or rather the layout of `COMM_AREA` starting at `CA_Customer_Num` matches the customer record structure being rewritten).

Let's organize these logical categories:
- Program parameters and CICS interface
- Communication Area (`COMM_AREA`) fields (such as `CA_CUSTOMER_NUM` and the fields following it representing the customer data to be rewritten)
- External data sources (such as VSAM dataset `KSDSCUST`)

Let's formulate the bulleted list accurately based on the program logic and inputs. I will write the final response directly without any introductory text, as requested.

## 3. Outputs

- **Data written to files or databases**
  - VSAM KSDS file KSDSCUST: the program rewrites the customer record that it locked with the preceding READ UPDATE.
    - Data written: the CICS REWRITE takes its data from the COMMAREA, starting at `CA_CUSTOMER_NUM`. It uses the length `WS_Commarea_LenF`, which is set from `EIBCALEN`.
    - Fields in that data:
      - Key and identifier: `CA_CUSTOMER_NUM`.
      - Customer personal data: `CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_DOB`, `CA_POSTCODE`.
      - Customer contact data: `CA_EMAIL_ADDRESS`.
      - Policy count: `CA_NUM_POLICIES`.
    - Role: this is the main output of the program. It persists the updated customer information in the VSAM file.
    - The record is read first into `WS_CUSTOMER_AREA`, using `CA_CUSTOMER_NUM` as the key (RIDFLD, 10 bytes). It is read only to hold it for update. The rewrite does not use this buffer.
    - The policy-related COMMAREA fields (`CA_POLICY_NUM`, `CA_PAYMENT`, `CA_BROKERID`, `CA_ISSUE_DATE`, `CA_EXPIRY_DATE`) are in the same COMMAREA. They overlay the same storage as the customer data. They go to the file only as part of the raw COMMAREA bytes that the rewrite writes.

- **Return values to the caller (COMMAREA)**
  - `CA_RETURN_CODE`: tells the calling program the outcome of the update.
    - '81': the READ FILE UPDATE on KSDSCUST failed (RESP is not NORMAL).
    - '82': the REWRITE on KSDSCUST failed (RESP is not NORMAL).
    - When both operations succeed, the program does not set it. It returns normally with the COMMAREA content as passed in.
  - `CA_CUSTOMER_NUM` and `CA_REQUEST_ID` stay in the COMMAREA, so the caller can match the result to the request.

- **Messages written to queues or other programs**
  - Error log message: on a failed READ or REWRITE, the program sends a formatted message to the logging program LGSTSQ with a CICS LINK. The routine is `WRITE_ERROR_MESSAGE`.
    - Message contents:
      - Timestamp: date (`EM_DATE`, from `WS_DATE`) and time (`EM_TIME`, from `WS_TIME`), taken from the CICS ASKTIME and FORMATTIME commands.
      - Program identifier: the literal 'LGUCVS01'.
      - Customer number: `EM_CUSNUM`, taken from `CA_CUSTOMER_NUM`.
      - Fixed text: ' Re-write  KSDSCUST '.
      - CICS response codes: `EM_RESPRC` (RESP) and `EM_RESP2RC` (RESP2).
    - Role: this gives operations and support staff a diagnostic record of the failure.
  - COMMAREA dump message: when `EIBCALEN` is greater than 0, a second message is linked to LGSTSQ. It has the prefix 'COMMAREA=' followed by `CA_Data`.
    - `CA_Data` holds the first `EIBCALEN` bytes of the raw COMMAREA, or the first 90 bytes if `EIBCALEN` is 91 or more.
    - Role: this records the input data that was being processed when the error occurred.

- **Program termination and abend side effects**
  - Read failure: the program abends with code 'LGV1' (NODUMP), after the return code is set to '81' and the error messages are logged.
  - Rewrite failure: the program abends with code 'LGV2' (NODUMP), after the return code is set to '82' and the error messages are logged.
    - Role of the abends: they end the transaction with a distinct code for each failure point. The CICS ABEND ends the task, so the `EXEC CICS RETURN` after it is not reached.
  - Successful path: the program ends with a normal RETURN, and the file update is complete.

## 4. Processing Logic

-  **Initialization and Setup**
  - The program is a CICS main procedure that receives a pointer to the COMMAREA (`COMM_AREA_PTR`) and overlays the `COMM_AREA` based structure on it.
  - It copies the CICS commarea length (`EIBCALEN`) into two working fields:
    - `WS_Commarea_Len` is used as the length of the read.
    - `WS_Commarea_LenF` is used as the length of the rewrite.
  - Working storage holds the response codes (`WS_RESP`, `WS_RESP2`), a 1024-byte customer buffer (`WS_Customer_Area`), time and date fields, and the error message layouts.

- **Customer Record Retrieval (VSAM KSDS Read for Update)**
  - The program issues `EXEC CICS READ FILE('KSDSCUST')` with the `UPDATE` option, which locks the record for a later rewrite.
    - The record is read into `WS_Customer_Area`.
    - The key (`RIDFLD`) is `CA_CUSTOMER_NUM`, the 10-digit customer number from the COMMAREA, with `KEYLENGTH(10)`.
    - The CICS response goes into `WS_RESP`.
  - **Read failure handling**
    - If `WS_RESP` is not `DFHRESP(NORMAL)`, the program:
      - Saves `EIBRESP2` in `WS_RESP2`.
      - Sets `CA_RETURN_CODE` to '81' so the caller knows the read-for-update failed.
      - Calls the internal procedure `WRITE_ERROR_MESSAGE` to log the failure.
      - Abends the task with `ABCODE('LGV1')` and `NODUMP`, then issues `EXEC CICS RETURN`.
  - The data read into `WS_Customer_Area` is not used afterwards. The read only establishes the update lock and positions the record for the rewrite.

- **Customer Record Update (Rewrite)**
  - If the read succeeds, the program issues `EXEC CICS REWRITE FILE('KSDSCUST')`.
    - The source data is taken `FROM(CA_Customer_Num)`, which is the start of the customer fields in the COMMAREA.
    - The length is `WS_Commarea_LenF`, which is `EIBCALEN`.
    - The response goes into `WS_RESP`.
  - **Rewrite failure handling**
    - If `WS_RESP` is not `DFHRESP(NORMAL)`, the program:
      - Saves `EIBRESP2` in `WS_RESP2`.
      - Sets `CA_RETURN_CODE` to '82' so the caller knows the rewrite failed.
      - Calls `WRITE_ERROR_MESSAGE`.
      - Abends the task with `ABCODE('LGV2')` and `NODUMP`, then issues `EXEC CICS RETURN`.
  - If the rewrite succeeds, the program ends with a plain `RETURN`. `CA_RETURN_CODE` is not set on this path.

- **Error Logging (WRITE_ERROR_MESSAGE Procedure)**
  - **Timestamp creation**
    - `EXEC CICS ASKTIME` gets the current absolute time into `WS_ABSTIME`.
    - `EXEC CICS FORMATTIME` converts it into `WS_DATE` (MMDDYYYY format) and `WS_TIME`.
  - **Error message construction**
    - The program fills the `ERROR_MSG` structure with:
      - The date (`EM_DATE`) and time (`EM_TIME`).
      - The customer number from the COMMAREA (`EM_CUSNUM`).
      - The CICS response code (`EM_RESPRC`) and the secondary response code (`EM_RESP2RC`).
    - The structure also contains the fixed text "LGUCVS01", " CNUM=", " Re-write KSDSCUST ", " RESP=" and " RESP2=".
  - **Logging to the queue program**
    - The program calls `LGSTSQ` with `EXEC CICS LINK`, passing `ERROR_MSG` as the COMMAREA with its length from `STG(ERROR_MSG)`. `LGSTSQ` is the external logging program.
  - **Conditional logging of the input COMMAREA contents**
    - This step runs only if `EIBCALEN > 0`, meaning a COMMAREA was passed.
    - If `EIBCALEN < 91`:
      - The program copies the whole COMMAREA content (left-justified, `EIBCALEN` bytes) into `CA_Data`.
      - It links to `LGSTSQ` with `CA_ERROR_MSG`, which has the "COMMAREA=" prefix.
    - Otherwise (`EIBCALEN` is 91 or more):
      - The program copies only the first 90 bytes of the COMMAREA into `CA_Data`.
      - It links to `LGSTSQ` with `CA_ERROR_MSG`.
  - The procedure then returns to the caller.

- **Control Flow Summary**
  - The program is sequential and has no loops.
  - Decisions come from three tests:
    - The response check after the READ.
    - The response check after the REWRITE.
    - The `EIBCALEN` tests inside the error logger.
  - Any file I/O failure ends the task with an abend, after the error has been logged through `LGSTSQ`.

- **External System Interactions**
  - VSAM KSDS file `KSDSCUST`: read for update, then rewrite.
  - CICS system services: `ASKTIME` and `FORMATTIME` for timestamps, `LINK` to the `LGSTSQ` logging program, `ABEND` for failure termination, and `RETURN`.
  - The caller communicates with the program through the COMMAREA. The program returns the status in `CA_RETURN_CODE` ('81' for a read failure, '82' for a rewrite failure).

## 5. Paragraphs

- **LGUCVS01 (main procedure body)**
  - Purpose: Entry point of the CICS PL/I program, declared `Options(Main)` and taking `COMM_AREA_PTR`. It updates a customer record in the VSAM KSDS file `KSDSCUST` using data passed in the COMMAREA.
  - Operations:
    - Copies `EIBCALEN` into `WS_Commarea_Len` and `WS_Commarea_LenF`. The first is used as the read length and the second as the rewrite length.
    - Issues `EXEC CICS READ FILE('KSDSCUST')` with the `UPDATE` option. It reads into `WS_Customer_Area` (1024 bytes), uses `CA_Customer_Num` as `RIDFLD` with `KeyLength(10)`, and returns the response in `WS_RESP`.
    - Issues `EXEC CICS REWRITE FILE('KSDSCUST')` from `CA_Customer_Num`, with length `WS_Commarea_LenF`, and returns the response in `WS_RESP`.
    - Ends with `RETURN`.
  - Control flow:
    - After the READ, if `WS_RESP` is not `DFHRESP(NORMAL)`, it does the following:
      - Saves `EIBRESP2` into `WS_RESP2`.
      - Sets `CA_RETURN_CODE` to '81'.
      - Calls `WRITE_ERROR_MESSAGE`.
      - Abends with code `LGV1` and `NODUMP`.
      - Issues `EXEC CICS RETURN`.
    - After the REWRITE, if `WS_RESP` is not `DFHRESP(NORMAL)`, it does the following:
      - Saves `EIBRESP2` into `WS_RESP2`.
      - Sets `CA_RETURN_CODE` to '82'.
      - Calls `WRITE_ERROR_MESSAGE`.
      - Abends with code `LGV2` and `NODUMP`.
      - Issues `EXEC CICS RETURN`.
    - If both operations succeed, the program falls through to the final `RETURN`.
  - External interactions:
    - VSAM KSDS file `KSDSCUST`, through CICS file control (READ for update, then REWRITE).
    - The COMMAREA, which is addressed through the based structure `COMM_AREA`. It carries the request fields and returns the status in `CA_RETURN_CODE`.
    - CICS ABEND processing on failure.

- **WRITE_ERROR_MESSAGE**
  - Purpose: Internal procedure, called when a VSAM READ or REWRITE fails. It builds a timestamped error message and sends it to the logging program `LGSTSQ`. The program name suggests a TS/TD queue logger.
  - Operations:
    - `EXEC CICS ASKTIME` gets the current absolute time into `WS_ABSTIME`.
    - `EXEC CICS FORMATTIME` converts it to a date (`MMDDYYYY` format) in `WS_DATE` and a time in `WS_TIME`.
    - Fills the `ERROR_MSG` structure:
      - `EM_DATE` from `WS_DATE`.
      - `EM_TIME` from `WS_TIME`.
      - `EM_CUSNUM` from `CA_Customer_Num`.
      - `EM_RESPRC` from `WS_RESP`.
      - `EM_RESP2RC` from `WS_RESP2`.
    - `EXEC CICS LINK PROGRAM('LGSTSQ')` sends `ERROR_MSG`, with its length given by `STG(ERROR_MSG)`.
    - If a COMMAREA was passed, it copies the leading part of `COMM_AREA_RAW` into `CA_Data`, then makes a second `LINK` to `LGSTSQ` with `CA_ERROR_MSG`. This message has the `COMMAREA=` prefix.
  - Control flow:
    - The COMMAREA dump happens only if `EIBCALEN > 0`.
    - If `EIBCALEN < 91`, it copies `EIBCALEN` bytes, left-justified, into `CA_Data`.
    - Otherwise it copies the first 90 bytes.
    - Both branches make the same `LINK` call.
    - The procedure ends with `RETURN`, which goes back to the caller.
  - External interactions:
    - CICS time services (`ASKTIME` and `FORMATTIME`).
    - Up to two `LINK` calls to program `LGSTSQ` for error logging, which pass `ERROR_MSG` and, when a COMMAREA exists, `CA_ERROR_MSG`.

## 6. Dependencies

- **Data stores and files**
  - VSAM KSDS file `KSDSCUST`
    - It holds the customer records and is defined in CICS as a file resource.
    - The program reads a record with `EXEC CICS READ FILE('KSDSCUST') ... UPDATE`. The key is `CA_CUSTOMER_NUM` (RIDFLD, KEYLENGTH 10). The record goes into `WS_Customer_Area`, a 1024-byte buffer.
    - The program updates the record with `EXEC CICS REWRITE FILE('KSDSCUST')`. The source field is `CA_Customer_Num`, and the length is `WS_Commarea_LenF`, which is set from `EIBCALEN`.
    - The READ-for-update takes a record lock that the REWRITE then uses. The file must therefore be defined in CICS as updatable.

- **Called programs and queues**
  - CICS program `LGSTSQ`
    - It is called with `EXEC CICS LINK` from the `WRITE_ERROR_MESSAGE` procedure and handles the error logging.
    - The first call passes the `ERROR_MSG` structure as the COMMAREA. The structure holds the date, time, program name, customer number, and RESP and RESP2 codes.
    - If `EIBCALEN` is greater than 0, the program makes a second call. It passes the `CA_ERROR_MSG` structure, which contains up to 90 bytes of the caller's COMMAREA content, copied from `COMM_AREA_RAW`.
    - It is called either once or twice.
  - The caller of `LGUCVS01`
    - A calling program or transaction passes the COMMAREA through the `COMM_AREA_PTR` parameter.
    - The caller must supply a COMMAREA that contains the customer number and the customer data to rewrite.

- **Input parameters and data sources**
  - COMMAREA (`COMM_AREA`, based on `COMM_AREA_PTR`)
    - This is the main interface with the caller. It is a union with a raw 32500-byte view in `COMM_AREA_RAW`.
    - Fields the program uses:
      - `CA_CUSTOMER_NUM` is the record key for the READ and the source of the REWRITE.
      - `CA_RETURN_CODE` is the status returned to the caller. It is set to '81' if the READ fails and '82' if the REWRITE fails.
    - Fields declared in the layout that carry customer or policy data:
      - `CA_REQUEST_ID`
      - Customer request fields: `CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_DOB`, `CA_POSTCODE`, `CA_NUM_POLICIES`, `CA_EMAIL_ADDRESS`
      - Policy request fields: `CA_POLICY_NUM`, `CA_PAYMENT`, `CA_BROKERID`, `CA_ISSUE_DATE`, `CA_EXPIRY_DATE`
    - The layout has further customer, policy-specific (endowment, house, motor, commercial, claim) and customer-security fields. The program does not reference them directly.
  - CICS Execute Interface Block (EIB)
    - `EIBCALEN` gives the COMMAREA length. The program uses it for the record length and to decide how much COMMAREA data to log.
    - `EIBRESP2` supplies the secondary response code for error reporting.

- **Copybooks and includes**
  - `LGCMAREA` (`plinc/LGCMAREA.inc`)
    - It defines the COMMAREA layout (`COMM_AREA`, `COMM_AREA_PTR`, and all `CA_*` fields).
    - It is used by all functions that share this layout.
    - It is shown as a commented `%INCLUDE`, and its content is expanded in the source.

- **CICS runtime services**
  - CICS file control (`READ UPDATE`, `REWRITE`) for access to `KSDSCUST`.
  - CICS time services
    - `ASKTIME` gets the current absolute time into `WS_ABSTIME`.
    - `FORMATTIME` formats it into `WS_DATE` (MMDDYYYY) and `WS_TIME` for the error message timestamp.
  - CICS program control
    - `LINK` calls `LGSTSQ`.
    - `ABEND` ends the task with abend codes `LGV1` (READ failure) and `LGV2` (REWRITE failure), both with NODUMP.
    - `RETURN` ends the task.
  - `DFHRESP(NORMAL)` is used to test response codes from the file commands.

- **Configuration and environment**
  - CICS resource definitions
    - The `KSDSCUST` file definition and the `LGSTSQ` program definition must exist in the CICS region.
  - Compile and build environment
    - The program needs the PL/I compiler with the CICS translator and access to the `plinc` include library.
    - The generated build files under `.bobz`/`bobz`, the application configuration (`application-conf`) and `zapp.yaml` support the build and are not read at run time.

- **Local working data and constants**
  - Error message structures
    - `ERROR_MSG` and `CA_ERROR_MSG` are built at run time and sent to `LGSTSQ`.
    - `ERROR_MSG` includes the literal program identifier ` LGUCVS01`. The `Eyecatcher` field also holds a program identifier.
  - Working fields
    - Response and length fields: `WS_RESP`, `WS_RESP2`, `WS_Comm_Len`, `WS_Commarea_Len`, `WS_Commarea_LenF`
    - `WS_Customer_Area`, the 1024-byte buffer that receives the record
    - Date and time fields: `WS_ABSTIME`, `WS_DATE`, `WS_TIME`
    - Declared but not used in the logic: `WS_STARTCODE` and `WS_SYSID`.

## 7. Constraints

- **Data Format and Structure Constraints**
  - The caller's COMMAREA must match the layout of the `COMM_AREA` structure, which is based on `COMM_AREA_PTR`.
    - `CA_REQUEST_ID` is a 6-character field.
    - `CA_RETURN_CODE` is `PIC '99'`, so it holds 2 digits.
    - `CA_CUSTOMER_NUM` is `PIC '9999999999'`, so it holds 10 digits.
    - `CA_NUM_POLICIES` is `PIC '999'`, so it holds 3 digits.
    - `CA_POLICY_NUM` and `CA_BROKERID` are each `PIC '9999999999'`, so each holds 10 digits.
    - `CA_PAYMENT` is `PIC '999999'`, so it holds 6 digits.
  - The customer fields have fixed widths:
    - `CA_FIRST_NAME` is 10 characters.
    - `CA_LAST_NAME` is 20 characters.
    - `CA_DOB` is 10 characters.
    - `CA_POSTCODE` is 8 characters.
    - `CA_EMAIL_ADDRESS` is 100 characters.
  - `CA_ISSUE_DATE` and `CA_EXPIRY_DATE` are each 10 characters.
  - The request-specific area is a `UNION` of the customer, security, and policy views. All three views overlay the same 32482-byte region.
  - The whole COMMAREA is declared as 32500 bytes (`COMM_AREA_RAW`). This is the maximum size the structure can describe.
  - The program does not check that the numeric fields hold valid digits. It does not check required fields, date formats, or value ranges. It assumes the caller supplies valid content.
  - `WS_Customer_Area` is a 1024-byte buffer. It limits the size of the VSAM record that the `READ` can return.
  - `WS_Commarea_Len` and `WS_Commarea_LenF` are `FIXED BIN(15)`, with a maximum of 32767. They can hold the 32500-byte COMMAREA length.

- **Key and Record Access Constraints**
  - The record is located only through `CA_Customer_Num`, used as `RIDFLD` with `KeyLength(10)`. The key must be a 10-byte value that matches an existing record in the KSDS file `KSDSCUST`.
  - If the key does not match a record, the `READ` returns a non-NORMAL response. The program treats this as a failure.
  - The record is read with the `UPDATE` option. A successful read gives the task a lock on the record, and the rewrite must follow. The program does nothing else to the record between the two commands.
  - The file name is fixed as `KSDSCUST`. The program cannot update any other file.

- **Sequencing Constraints**
  - Processing follows a fixed order:
    - Read the record for update.
    - Rewrite the record.
    - Return.
  - The `REWRITE` is reached only if the `READ UPDATE` returned `DFHRESP(NORMAL)`. A failed read ends the task before the rewrite.
  - The `REWRITE` has no `FROM` data other than `CA_Customer_Num`, and it must follow the `READ UPDATE` on the same file. This ordering is the CICS requirement for a rewrite.
  - Error logging runs before the abend: `WRITE_ERROR_MESSAGE` is called, then `ABEND`.
  - Inside `WRITE_ERROR_MESSAGE`, the order is `ASKTIME`, then `FORMATTIME`, then building the message, then the `LINK`. Time and date must be obtained before the message is built.

- **Error Handling and Return Code Constraints**
  - Each CICS file command must complete with `DFHRESP(NORMAL)`. The program checks `RESP` after both the `READ` and the `REWRITE`.
  - If the `READ UPDATE` fails:
    - `CA_RETURN_CODE` is set to `'81'`.
    - `WS_RESP2` is taken from `EIBRESP2`.
    - An error message is written.
    - The task abends with code `LGV1` (`NODUMP`).
  - If the `REWRITE` fails:
    - `CA_RETURN_CODE` is set to `'82'`.
    - `WS_RESP2` is taken from `EIBRESP2`.
    - An error message is written.
    - The task abends with code `LGV2` (`NODUMP`).
  - Every error is fatal. There is no retry and no recovery path.
  - Any failure ends the task with an abend, and no dump is taken because of `NODUMP`.
  - The `RETURN` after each `ABEND` is not reachable in practice.
  - On success, the program returns normally and leaves `CA_RETURN_CODE` unchanged. It does not set a success value.

- **Error Message Logging Constraints**
  - The error message is written by linking to program `LGSTSQ`. That program must be available to the task.
  - The first message is the `ERROR_MSG` structure.
    - The date is taken from `FORMATTIME ... MMDDYYYY`. It fits `EM_DATE`, which is 10 characters.
    - The time fits `EM_TIME`, which is 8 characters.
    - `EM_CUSNUM` is 10 characters.
    - `EM_RESPRC` and `EM_RESP2RC` use the picture `+(5)9`. They show the RESP and RESP2 values with a sign and up to 6 characters.
    - The message text area is limited to 63 characters.
  - The `LINK` length is the storage size of the structure, `STG(ERROR_MSG)`.
  - A second message, `CA_ERROR_MSG`, carries a copy of the COMMAREA. It is sent only when `EIBCALEN > 0`.
    - If `EIBCALEN < 91`, the message holds the first `EIBCALEN` bytes of the COMMAREA.
    - Otherwise it holds at most the first 90 bytes. This is the size of `CA_Data`, so the logged COMMAREA content is truncated to 90 bytes.
  - If the program is called without a COMMAREA (`EIBCALEN = 0`), the COMMAREA copy is not written. The `COMM_AREA` fields are still referenced earlier in the program. The code has no check that rejects a missing COMMAREA.

- **Resource and Size Limits**
  - The record buffer is 1024 bytes.
  - The COMMAREA is up to 32500 bytes, and the logged copy is up to 90 bytes.
  - The `READ` uses `WS_Commarea_Len` as `LENGTH`, and this value is initialised from `EIBCALEN`. The length of the COMMAREA therefore affects how much of the record the `READ` can return, and a record longer than that length causes a `LENGTH` error response, which the program treats as an error.
  - The `REWRITE` uses `WS_Commarea_LenF`, which is also set from `EIBCALEN`. The rewritten record length is therefore the same as the COMMAREA length that the caller passed.
  - The key length is fixed at 10 bytes.

- **Environment and Execution Constraints**
  - The program must run in a CICS region. The file `KSDSCUST` must be defined and open, and it must allow update access.
  - The program is a main procedure (`Options(Main)`) that receives the COMMAREA pointer as a parameter.
  - The `EIB` fields `EIBCALEN` and `EIBRESP2` are used and must be available.
  - The abend codes `LGV1` and `LGV2` identify the failing operation for the caller or an operator.

## 8. Error Handling

**CICS response checking on file operations**
- Read-for-update of KSDSCUST
  - The program captures the CICS response in a RESP field and compares it to the NORMAL response value.
  - Any non-normal response sends the program into its error path. The program does not distinguish between causes such as record not found, I/O error, or a locked record. All of them are handled the same way.
  - On failure, the secondary response code (EIBRESP2) is copied to a working field so it can be logged.
  - The return code in the COMMAREA is set to '81'. This tells the caller the read-for-update failed.
- Rewrite of KSDSCUST
  - The rewrite uses the same RESP capture and NORMAL comparison.
  - On failure, EIBRESP2 is captured and the COMMAREA return code is set to '82'. This tells the caller the rewrite failed.
  - The rewrite is only reached if the read-for-update succeeded.

**Return code signalling to the caller**
- The COMMAREA return code field identifies where the failure occurred. '81' means the read failed and '82' means the rewrite failed.
- The program never sets a success value in the return code. On success it only executes a plain RETURN.
- The return code is set before the abend, so it is only useful to a caller that can see the COMMAREA after the abend, for example through transaction backout or diagnostic logging.

**Error logging and notification**
- A dedicated internal procedure, called from both failure paths, builds and writes the error messages.
  - It gets the current time with ASKTIME and formats it into date and time strings with FORMATTIME.
  - It fills a structured error message with the date, time, program identifier, customer number, and the RESP and RESP2 values. The message text includes the literal text "Re-write KSDSCUST".
  - That literal also appears in the message for a failed read, so the text can mislead.
  - It sends the message to the shared logging program LGSTSQ with a CICS LINK, passing the message as the COMMAREA.
- A second diagnostic record carries the COMMAREA contents.
  - It is only written if the incoming COMMAREA length is greater than zero.
  - If the length is under 91 bytes, it copies the whole COMMAREA, left-justified, into a 90-byte data area. If the length is 91 or more, it truncates the copy to the first 90 bytes.
  - It sends this record to LGSTSQ with a second LINK.
  - The length check protects against a missing or short COMMAREA. It prevents the program from logging data that is not there.
- The program does not check the RESP of the LINK calls to LGSTSQ. If logging fails, the program does not detect it.

**Program termination and abend handling**
- Both failure paths follow the logging step with a CICS ABEND using NODUMP.
  - Abend code 'LGV1' marks a read-for-update failure.
  - Abend code 'LGV2' marks a rewrite failure.
  - NODUMP suppresses the storage dump, which reduces overhead and noise.
- Each abend is followed by a CICS RETURN statement. It is unreachable in practice because the ABEND ends the task. It would only matter if the abend were suppressed or handled by an abend exit.
- The abend also lets CICS back out uncommitted changes. This keeps the file consistent if the read succeeded but the rewrite failed.

**Items not present**
- There are no retries or fallback logic. A failed read or rewrite does not trigger alternate paths, default values, or further attempts.
- There are no PL/I ON-unit exception handlers, such as ON ERROR or ON CONDITION. All error handling uses explicit CICS response checking.
- The program does not validate input values such as the customer number or the COMMAREA contents. It does not check the COMMAREA length for the file operations, other than in the logging routine.
- The program has no explicit HANDLE CONDITION or HANDLE ABEND logic. It never recovers from the abends it issues itself.

## 9. Examples

### 9.1 Example 1 — Successful Customer Update

**Input (COMMAREA fields):**

| Field | Value |
|---|---|
| `CA_REQUEST_ID` | `'01UCUS'` |
| `CA_RETURN_CODE` | `'00'` |
| `CA_CUSTOMER_NUM` | `0000012345` |
| `CA_FIRST_NAME` | `'Jane      '` |
| `CA_LAST_NAME` | `'Smith               '` |
| `CA_DOB` | `'1985-04-22'` |
| `CA_POSTCODE` | `'SW1A 2AA'` |
| `CA_NUM_POLICIES` | `003` |
| `CA_EMAIL_ADDRESS` | `'jane.smith@example.com'` |

**Expected output:**

- `CA_RETURN_CODE` remains `'00'`.
- The VSAM KSDS file `KSDSCUST` is updated: the existing record for customer `0000012345` is replaced with the data supplied in the COMMAREA (from `CA_CUSTOMER_NUM` onward).
- CICS RETURN is issued; control passes back to the calling orchestrator.

**How the logic works:**

1. `EXEC CICS READ … UPDATE` locks the `KSDSCUST` record keyed on `CA_CUSTOMER_NUM` (`0000012345`). The record is read into `WS_CUSTOMER_AREA` solely to acquire the CICS update lock; its contents are not used.
2. Because `WS_RESP = DFHRESP(NORMAL)`, the error branch is skipped.
3. `EXEC CICS REWRITE` writes the entire COMMAREA block starting at `CA_CUSTOMER_NUM` back to the file, effectively replacing the record with the new field values (name, DOB, postcode, policies, e-mail, etc.).
4. Again `WS_RESP = DFHRESP(NORMAL)`, so no error path is taken and the program returns normally.

---

### 9.2 Example 2 — VSAM Read-for-Update Fails (customer not found / file unavailable)

**Input (COMMAREA fields):**

| Field | Value |
|---|---|
| `CA_REQUEST_ID` | `'01UCUS'` |
| `CA_RETURN_CODE` | `'00'` |
| `CA_CUSTOMER_NUM` | `9999999999` *(does not exist in KSDSCUST)* |

**Expected output:**

- `CA_RETURN_CODE` is set to `'81'`.
- An error message is written to the CICS temporary-storage queue via `LGSTSQ`, containing the timestamp, program name `LGUCVS01`, the customer number `9999999999`, and the CICS RESP/RESP2 codes.
- CICS abend `LGV1` (NODUMP) is issued, then `EXEC CICS RETURN`.

**How the logic works:**

1. `EXEC CICS READ … UPDATE` targets key `9999999999`. Because the record does not exist (or the file is unavailable), CICS returns a non-normal RESP code.
2. The condition `WS_RESP <> DFHRESP(NORMAL)` is true; `WS_RESP2` is captured from `EIBRESP2`.
3. `CA_RETURN_CODE` is set to `'81'` and `WRITE_ERROR_MESSAGE` is called. That procedure gets the current timestamp via `ASKTIME`/`FORMATTIME`, formats an error message with the customer number and response codes, and links to `LGSTSQ` to write it to the TS queue. If the COMMAREA is 90 bytes or fewer, it is also logged in full.
4. The program abends with code `LGV1` (no dump) and returns, signalling the failure to the caller via `CA_RETURN_CODE = '81'`.

---

### 9.3 Example 3 — VSAM Rewrite Fails

**Input (COMMAREA fields):**

Same as Example 1 (`CA_CUSTOMER_NUM = 0000012345`), but the VSAM file is in read-only mode or a resource conflict prevents the rewrite.

**Expected output:**

- `CA_RETURN_CODE` is set to `'82'`.
- An error message (timestamp, program name, customer number, RESP/RESP2) is written to the TS queue via `LGSTSQ`.
- CICS abend `LGV2` (NODUMP) is issued, then `EXEC CICS RETURN`.

**How the logic works:**

1. The initial `EXEC CICS READ … UPDATE` succeeds (RESP = NORMAL), so the record lock is held.
2. `EXEC CICS REWRITE` fails (non-normal RESP), perhaps due to a file-control conflict or I/O error.
3. `CA_RETURN_CODE` is set to `'82'`, `WRITE_ERROR_MESSAGE` logs the error, and the program terminates with abend code `LGV2`. This distinct code (`'82'` vs. `'81'`) lets the calling orchestrator distinguish a rewrite failure from a read failure.

---

Generated by IBM Bob Premium Package for Z
