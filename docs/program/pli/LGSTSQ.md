## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
- [3. Outputs](#3-outputs)
- [4. Processing Logic](#4-processing-logic)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
- [7. Constraints](#7-constraints)
- [8. Error Handling](#8-error-handling)
  - [8.1 Response capture without evaluation](#81-response-capture-without-evaluation)
  - [8.2 Input source validation and routing](#82-input-source-validation-and-routing)
  - [8.3 Defaulting and fallback logic](#83-defaulting-and-fallback-logic)
  - [8.4 Resource contention handling](#84-resource-contention-handling)
  - [8.5 Notification and logging](#85-notification-and-logging)
  - [8.6 Termination](#86-termination)
- [9. Examples](#9-examples)
  - [9.1 Example 1 — Program-to-Program LINK (commarea input, default queue)](#91-example-1--program-to-program-link-commarea-input-default-queue)
  - [9.2 Example 2 — Terminal-entered message with `Q=` prefix (custom queue)](#92-example-2--terminal-entered-message-with-q-prefix-custom-queue)

## 1. Purpose

LGSTSQ is a CICS error and message logging utility for the GENAPP application. Other programs call it to record diagnostic or error text, or an operator can start it from a terminal. It builds a log record that has the originating CICS system identifier followed by the message text. It then writes that record to two places: the standard CICS transient data queue CSMT and a temporary storage queue. The temporary storage queue is named GENAERRS by default. The message can start with a "Q=nnnn" prefix, which sends the record to a queue named GENAnnnn instead. When a terminal started the program, it clears the screen and releases the keyboard to confirm the message was logged. It then returns control to CICS.

## 2. Inputs

- Function Parameters
  - `COMM_AREA_PTR`: Pointer to the communication area passed by a calling CICS program. It addresses the `COMMA` structure containing `COMMA_DATA`.
  - `COMMA_DATA`: A 90-character message text passed from an invoking program via the COMMAREA. It serves as the primary log message content when the program is called programmatically.
  - `EIBCALEN`: An Execute Interface Block (EIB) system variable supplying the length of the passed communication area, used to initialize the message length calculation.

- User and Terminal Inputs
  - `WS_RECV`: A structured record received from the terminal when the program is invoked directly via a transaction rather than a program call.
    - `WS_RECV_TRANID`: The 5-character leading transaction identifier received from the terminal input, which is stripped before logging.
    - `WS_RECV_DATA`: The 74-character message text input directly by a terminal user to be logged.

- System and Environment Context (CICS ASSIGN)
  - `WRITE_MSG_SYSID`: The 4-character CICS system identifier obtained via `EXEC CICS ASSIGN SYSID`, used as a prefix on outgoing log messages to identify the source region.
  - `WS_INVOKEPROG`: The 8-character invoking program name obtained via `EXEC CICS ASSIGN INVOKINGPROG`, used to determine whether input should be read from the COMMAREA or received from the terminal.
  - `WS_STARTCODE`: The 2-character task startup code obtained via `EXEC CICS ASSIGN STARTCODE` indicating how the task was initiated.
  - `WS_PRINSYSID`: The 4-character principal system identifier obtained via `EXEC CICS ASSIGN PRINSYSID` identifying the system associated with the principal facility.

## 3. Outputs

- **Log record written to the CICS transient data queue (TDQ)**
  - Destination: TDQ `CSMT` (`STDQ_NAME`), the standard CICS message destination.
  - Written by `EXEC CICS WRITEQ TD FROM(WRITE_MSG) LENGTH(WS_RECV_LEN)`.
  - `WRITE_MSG` (the outgoing record) is made up of:
    - `WRITE_MSG_SYSID`: the 4-character CICS system ID of the originating region, set by `ASSIGN SYSID`. It prefixes each record so the source region is identifiable.
    - `FILLER_X`: a 1-character blank separator between the system ID and the message text.
    - `WRITE_MSG_MSG`: the 90-character message text.
  - `WS_RECV_LEN` sets how many bytes of the record are written.

- **Log record written to the CICS temporary storage queue (TSQ)**
  - Destination: the TSQ named in `STSQ`.
    - The default is `GENAERRS`.
    - If the message starts with `Q=nnnn`, positions 5-8 are replaced to give `GENAnnnn`.
  - Written by `EXEC CICS WRITEQ TS FROM(WRITE_MSG) LENGTH(WS_RECV_LEN) NOSUSPEND`.
  - The record is the same `WRITE_MSG` content and length as the TDQ record.
  - `NOSUSPEND` means the task does not wait for storage if none is available. The request is ignored in that case, and the response is only stored in `WS_RESP`.

- **Message content and length determination**
  - Message source:
    - `COMMA_DATA` supplies the text when the logger is called by another program (`WS_INVOKEPROG` not blank, `WS_FLAG` = 'C').
    - `WS_RECV_DATA` supplies the text when the logger is started from a terminal (`WS_FLAG` = 'R'). It comes from a CICS RECEIVE.
    - `WS_RECV_TRANID` is the leading transaction ID part of the terminal input. It is excluded from the logged text.
  - Queue-prefix handling:
    - If the text begins with `Q=`, the 4-character suffix selects the queue name in `STSQ`.
    - The prefix is stripped through `TEMPO`, which holds the remaining text. That text is copied back into `WRITE_MSG_MSG`.
  - Length calculation:
    - `WS_RECV_LEN` starts from the commarea length (`EIBCALEN`) or the received length less 5 for the transaction ID.
    - It is reduced by 7 when a `Q=nnnn` prefix is removed.
    - It is increased by 5 for the system ID header.
    - The result is the length used for both queue writes.

- **Terminal screen output (terminal-initiated invocations only)**
  - Only happens when `WS_FLAG` = 'R'.
  - `EXEC CICS SEND TEXT FROM(FILLER_X) LENGTH(1) TERMINAL WAIT ERASE FREEKB` sends a one-character blank text to the terminal.
  - This clears the screen and unlocks the keyboard, acting as a minimal confirmation. No message text is shown.
  - Nothing is sent when the caller is another program.

- **Task termination**
  - `EXEC CICS RETURN` ends the task and returns control to CICS or the caller.
  - The program has no explicit return value.
  - The commarea is not modified.

- **Variables that do not affect outputs**
  - `WS_STARTCODE` and `WS_PRINSYSID` are populated by `ASSIGN` but never used afterwards.
  - `MSGEND` is declared but not used.
  - `WS_RESP` receives the CICS response codes, but the program never checks them. Failures on the queue writes or the terminal receive are therefore silent.

## 4. Processing Logic

- **Initialization**
  - The program is the main entry point `LGSTSQ`. It receives a pointer to the communication area, `COMM_AREA_PTR`. `COMMA_DATA` is a 90-character message overlaid on that area.
  - `WRITE_MSG` (the outgoing record) and `WS_RECV` (the terminal receive area) are cleared to blanks.
  - Defaults set at declaration:
    - `WS_RECV_LEN` starts at 80.
    - `STDQ_NAME` is `CSMT`.
    - `FILLER_X`, the separator between the system ID and the message text, is a blank.
  - `MSGEND` ("Transaction ended") is declared but never used.

- **Gathering CICS task context**
  - The program issues four `EXEC CICS ASSIGN` commands. Each one stores its RESP code in `WS_RESP`, and the code never checks it.
    - `SYSID` goes into `WRITE_MSG_SYSID`. It becomes the 4-character source-region prefix on every logged record.
    - `STARTCODE` goes into `WS_STARTCODE`.
    - `PRINSYSID` goes into `WS_PRINSYSID`.
    - `INVOKINGPROG` goes into `WS_INVOKEPROG`. This value decides where the message comes from.
  - `WS_STARTCODE` and `WS_PRINSYSID` are collected but not used later.

- **Determining the message source**
  - The check is on `WS_INVOKEPROG`.
  - **Invoked by another program** (`WS_INVOKEPROG` is not blank):
    - `WS_FLAG` is set to `C`.
    - `COMMA_DATA` from the commarea is copied into `WRITE_MSG_MSG`.
    - `WS_RECV_LEN` is set to the commarea length, `EIBCALEN`.
  - **Started from a terminal** (`WS_INVOKEPROG` is blank):
    - `EXEC CICS RECEIVE INTO(WS_RECV)` reads the typed input. `WS_RECV_LEN` is the length field.
    - `WS_FLAG` is set to `R`.
    - `WS_RECV_DATA` is copied into `WRITE_MSG_MSG`. This excludes the 5-character transaction ID in `WS_RECV_TRANID`.
    - `WS_RECV_LEN` is reduced by 5 to remove the transaction ID from the length.

- **Queue name selection and message prefix handling**
  - `STSQ` defaults to `GENAERRS`.
  - If the first two characters of `WRITE_MSG_MSG` are `Q=`:
    - Positions 3–6 of the message (`nnnn`) are copied into positions 5–8 of `STSQ`, giving the queue name `GENAnnnn`.
    - The text from position 8 onward is copied into the temporary buffer `TEMPO`.
    - `TEMPO` is copied back into `WRITE_MSG_MSG`. This removes the `Q=nnnn` prefix and the character after it.
    - `WS_RECV_LEN` is reduced by 7 to match the removed prefix.
  - If the message has no `Q=` prefix, the text and the default queue name are left as they are.

- **Length adjustment**
  - `WS_RECV_LEN` is increased by 5. This accounts for the 4-byte system ID and the 1-byte filler at the start of `WRITE_MSG`.
  - The result is the length used for both queue writes.

- **Writing to external queues**
  - **Transient data queue**
    - `EXEC CICS WRITEQ TD` writes `WRITE_MSG` to queue `CSMT`, the standard CICS message destination.
    - The write uses `WS_RECV_LEN` as its length.
    - The response goes into `WS_RESP` and is not checked.
  - **Temporary storage queue**
    - `EXEC CICS WRITEQ TS` writes the same `WRITE_MSG` to the queue named in `STSQ`.
    - It uses `NOSUSPEND`. If no storage is available, the task does not wait. The request is ignored.
    - The response goes into `WS_RESP` and is not checked.

- **Terminal feedback**
  - If `WS_FLAG` is `R` (the task started from a terminal), `EXEC CICS SEND TEXT` sends the 1-character `FILLER_X` to the terminal. This clears the screen and unlocks the keyboard.
    - The options are `ERASE`, `WAIT` and `FREEKB`.
    - The text length is 1.
  - If `WS_FLAG` is `C` (the task was called by a program), nothing is sent to a terminal.

- **Termination**
  - `EXEC CICS RETURN` ends the task or returns control to the caller.

- **Control flow summary**
  - There are no loops. The program runs straight through with two conditional branches:
    - The terminal-or-commarea source decision.
    - The `Q=` prefix test.
  - A third conditional controls the terminal confirmation screen.
  - Error handling relies on the RESP values being stored. The logic never tests them.

## 5. Paragraphs

**LGSTSQ (main procedure)**
- Purpose: CICS error-message logger. It writes one message record to the transient data queue CSMT and to a Genapp temporary storage queue. The default TS queue is GENAERRS. A leading `Q=nnnn` in the message selects queue GENAnnnn.
- The whole program is one main procedure, `LGSTSQ: Proc(COMM_AREA_PTR) Options(Main)`, with no internal sub-procedures or labeled paragraphs. The logic runs top to bottom in the blocks below.
- Declarations and linkage:
  - Working fields: `WS_FLAG`, `WS_RESP`, `WS_STARTCODE`, `WS_INVOKEPROG`, `WS_PRINSYSID`, `WS_RECV` (`WS_RECV_TRANID` and `WS_RECV_DATA`), and `WS_RECV_LEN` (initial value 80).
  - Output record: `WRITE_MSG` holds `WRITE_MSG_SYSID`, a one-character `FILLER_X` and `WRITE_MSG_MSG`.
  - Queue names and buffers: `STSQ`, `STDQ_NAME` (initial value 'CSMT'), `TEMPO` and `MSGEND`. `MSGEND` is declared but never used.
  - The commarea is addressed through `COMM_AREA_PTR` and the based structure `COMMA`, whose `COMMA_DATA` is 90 bytes.

**Initialization block**
- Purpose: clears the work areas before any data is gathered.
- Operations: sets `WRITE_MSG` and `WS_RECV` to blanks.
- Control flow: none.
- External interaction: none.

**Environment retrieval block (CICS ASSIGN commands)**
- Purpose: collects task and system information.
- Operations:
  - ASSIGN SYSID fills `WRITE_MSG_SYSID`.
  - ASSIGN STARTCODE fills `WS_STARTCODE`.
  - ASSIGN PRINSYSID fills `WS_PRINSYSID`.
  - ASSIGN INVOKINGPROG fills `WS_INVOKEPROG`.
  - Each command stores its response code in `WS_RESP`. The program never checks `WS_RESP`.
- Control flow: sequential.
- External interaction: CICS ASSIGN requests to the CICS region.

**Input source determination block**
- Purpose: decides whether the message came from a calling program or from a terminal.
- Control flow: IF `WS_INVOKEPROG <> ' '` ... ELSE.
  - Invoked by a program:
    - Sets `WS_FLAG` to 'C'.
    - Copies `COMMA_DATA` to `WRITE_MSG_MSG`.
    - Sets `WS_RECV_LEN` to `EIBCALEN`.
  - Started from a terminal:
    - Issues EXEC CICS RECEIVE INTO(`WS_RECV`) with LENGTH(`WS_RECV_LEN`). This is the only terminal input.
    - Sets `WS_FLAG` to 'R'.
    - Copies `WS_RECV_DATA` to `WRITE_MSG_MSG`.
    - Subtracts 5 from `WS_RECV_LEN` to drop the transaction ID portion.
- External interaction: the commarea, or terminal input through CICS RECEIVE.

**Queue name selection block**
- Purpose: picks the temporary storage queue name and removes the optional `Q=` prefix.
- Operations:
  - Sets `STSQ` to 'GENAERRS'.
  - If the first two characters of `WRITE_MSG_MSG` are 'Q=':
    - Copies characters 3–6 of the message into positions 5–8 of `STSQ`, which gives GENAnnnn.
    - Moves the text from position 8 onward through `TEMPO` back into `WRITE_MSG_MSG`.
    - Subtracts 7 from `WS_RECV_LEN` to drop the prefix.
- Control flow: a conditional IF/DO on the 'Q=' prefix.
- External interaction: none.

**Length adjustment**
- Purpose: makes the record length cover the header.
- Operations: adds 5 to `WS_RECV_LEN` for the system ID (4 bytes) and the filler (1 byte), giving the length written to both queues.

**Transient data queue write**
- Purpose: logs the message to the CICS TD queue.
- Operations: EXEC CICS WRITEQ TD QUEUE(`STDQ_NAME`) FROM(`WRITE_MSG`) LENGTH(`WS_RECV_LEN`) RESP(`WS_RESP`).
- Control flow: none. `WS_RESP` is not checked.
- External interaction: writes to the TD queue CSMT.

**Temporary storage queue write**
- Purpose: logs the same message to the Genapp TS queue.
- Operations: EXEC CICS WRITEQ TS QUEUE(`STSQ`) FROM(`WRITE_MSG`) LENGTH(`WS_RECV_LEN`) NOSUSPEND RESP(`WS_RESP`).
- Control flow: NOSUSPEND means the task does not wait for storage. If no space is available, the request is ignored.
- External interaction: writes to the TS queue named in `STSQ`.

**Terminal confirmation block**
- Purpose: clears the terminal after a terminal-initiated run.
- Control flow: runs only when `WS_FLAG = 'R'`.
- Operations: EXEC CICS SEND TEXT FROM(`FILLER_X`) with LENGTH(1), TERMINAL, WAIT, ERASE and FREEKB. This sends one blank character, erases the screen and frees the keyboard.
- External interaction: terminal output.

**Termination**
- Purpose: ends the task.
- Operations: EXEC CICS RETURN, followed by END LGSTSQ.
- External interaction: returns control to CICS or to the calling program.

## 6. Dependencies

- **CICS runtime environment**
  - CICS region and transaction services
    - The program is a main PL/I procedure, `LGSTSQ`, that runs as a CICS transaction. It needs an active CICS region and a transaction definition.
    - It ends with `EXEC CICS RETURN`, which gives control back to CICS.
  - CICS Execute Interface Block (EIB)
    - It reads `EIBCALEN` to get the length of the commarea passed by a calling program. This value sets the initial message length, `WS_RECV_LEN`.
  - CICS ASSIGN services
    - `ASSIGN SYSID` fills `WRITE_MSG_SYSID` with the local CICS system identifier. This identifier prefixes every logged record.
    - `ASSIGN STARTCODE` fills `WS_STARTCODE` with how the task was started. The program does not use this value afterward.
    - `ASSIGN PRINSYSID` fills `WS_PRINSYSID` with the principal facility system identifier. The program does not use this value afterward.
    - `ASSIGN INVOKINGPROG` fills `WS_INVOKEPROG` with the name of the calling program. The program tests this field to choose between commarea input and terminal input.
  - Response handling
    - Every CICS command returns its response code in `WS_RESP`. The program never checks this value, so it ignores CICS errors.

- **Input parameters and data sources**
  - Communication area from a calling program
    - `COMM_AREA_PTR` is the procedure parameter. It is a pointer to the commarea.
    - The `COMMA` based structure maps the commarea. `COMMA_DATA` is the 90-character message text.
    - The program uses this input only when `WS_INVOKEPROG` is not blank, meaning another program called it.
  - Terminal input
    - When no invoking program exists, the program runs `EXEC CICS RECEIVE INTO(WS_RECV)` to read what the user typed. The target structure is `WS_RECV`. It holds the 5-character `WS_RECV_TRANID` and the 74-character `WS_RECV_DATA`.
    - The receive length is `WS_RECV_LEN`. The program uses this length to size the logged record.
  - Embedded queue-selection parameter
    - If the message text begins with `Q=`, the next 4 characters (`nnnn`) are the suffix of the temporary storage queue name. The text after the 7-byte prefix becomes the logged message.
    - Without this prefix, the default queue is `GENAERRS`.

- **Output destinations (CICS queues)**
  - Transient data queue
    - `CSMT` is held in `STDQ_NAME`. It is the standard CICS message destination, and it must be defined in the CICS destination control table or as a resource definition.
    - The program writes the `WRITE_MSG` record to it with `EXEC CICS WRITEQ TD`.
  - Temporary storage queue
    - `GENAERRS`, or `GENAnnnn` when the `Q=nnnn` prefix is present, is held in `STSQ`. The queue is created dynamically or comes from a TS model or queue definition.
    - The program writes the same `WRITE_MSG` record to it with `EXEC CICS WRITEQ TS ... NOSUSPEND`. If no space is available, the request is ignored and the task does not wait.
  - Terminal output
    - When the input came from a terminal (`WS_FLAG` = 'R'), the program runs `EXEC CICS SEND TEXT` with `ERASE`, `WAIT`, `FREEKB` and `TERMINAL`. This clears the screen and unlocks the keyboard. It needs a terminal as the principal facility.

- **External programs, modules and copybooks**
  - There are no `%INCLUDE` members, copybooks, database access, file I/O or calls to other programs or subroutines.
  - The caller is the only program involved. Any program that logs errors by linking to `LGSTSQ` and passing a commarea depends on it.

- **Configuration values and constants**
  - Hard-coded values
    - The default TS queue name is `GENAERRS`. The prefix for custom queues is `GENA`.
    - The TD queue name is `CSMT`.
    - The `Q=` prefix marker is 7 bytes long. The system identifier header is 5 bytes: 4 for the sysid and 1 for a filler. Both are used in the length calculations.
    - The default record length is 80 bytes.
  - Unused declarations
    - `MSGEND` ("Transaction ended") is declared but not used.
    - `WS_STARTCODE` and `WS_PRINSYSID` are populated but not used.
  - Environment variables and external configuration files: none are used.

- **Build and runtime prerequisites**
  - The program must be compiled with the CICS translator or integrated CICS translation for PL/I. This handles the `EXEC CICS` statements and provides the EIB (`EIBCALEN`).
  - The program must be defined to CICS as a PROGRAM resource, with a TRANSACTION definition if it is started from a terminal.

## 7. Constraints

- **Input source and sequencing**
  - The message source depends on how the task was started.
    - The program issues `ASSIGN INVOKINGPROG` and tests `WS_INVOKEPROG <> ' '`.
    - A non-blank value means a program called it. It sets `WS_FLAG` to 'C', copies `COMMA_DATA` into `WRITE_MSG_MSG`, and takes `WS_RECV_LEN` from `EIBCALEN`.
    - A blank value means a terminal started it. It issues `EXEC CICS RECEIVE INTO(WS_RECV)`, sets `WS_FLAG` to 'R', and copies `WS_RECV_DATA` into `WRITE_MSG_MSG`.
  - The processing order is fixed.
    - It first clears `WRITE_MSG` and `WS_RECV`.
    - It then runs the four ASSIGN commands (SYSID, STARTCODE, PRINSYSID, INVOKINGPROG).
    - It then acquires the message, resolves the queue name, adjusts the length, and writes to the TDQ.
    - It then writes to the TSQ, sends the terminal confirmation, and returns.
  - The TDQ write always comes before the TSQ write. The terminal confirmation `SEND TEXT` comes only after both writes and only when `WS_FLAG = 'R'`.
  - `EXEC CICS RETURN` is the only exit path, and it ends the task after one message is logged.

- **Message format and size limits**
  - The log record `WRITE_MSG` has a fixed layout: a 4-byte `WRITE_MSG_SYSID`, a 1-byte `FILLER_X` separator, and a 90-byte `WRITE_MSG_MSG`. That is 95 bytes in total.
  - The message text is limited to 90 bytes.
    - The commarea field `COMMA_DATA` is `CHAR(90)`, so only the first 90 bytes of the caller's commarea are used.
    - The `COMMA` structure is BASED on `COMM_AREA_PTR`, so the commarea is assumed to be at least 90 bytes long. The program does not check this.
  - Terminal input is limited to 79 bytes.
    - It is received into `WS_RECV`, which is a 5-byte `WS_RECV_TRANID` plus a 74-byte `WS_RECV_DATA`.
    - The logged text is limited to 74 bytes. The transaction id prefix is dropped and not logged.
    - `WS_RECV_LEN` starts at 80, which is larger than the 79-byte receive area.
  - The logged text uses character fields padded with blanks. `WRITE_MSG` and `WS_RECV` are cleared to blanks before use, so unused positions are blank.
  - `WS_RECV_LEN` is a halfword (`FIXED BIN(15)`), so lengths are limited to that range.
  - The record length is adjusted to match the content.
    - For terminal input, 5 is subtracted for the transaction id.
    - If the text starts with a `Q=` prefix, 7 more is subtracted.
    - 5 is then added to cover the system id and separator header.
    - The result is passed as `LENGTH` on both WRITEQ commands.
  - Several fields are declared but not used in the logic.
    - `MSGEND`, `WS_STARTCODE`, and `WS_PRINSYSID` are declared. The last two are assigned but never used in a decision.
    - `WS_RECV_TRANID` is excluded from the logged text.

- **Queue routing rules**
  - The TDQ name is fixed.
    - `STDQ_NAME` is initialised to 'CSMT'.
    - It is never changed, so every message goes to the CSMT destination.
  - The TSQ name defaults to 'GENAERRS'. The program sets `STSQ = 'GENAERRS'` before checking the message prefix.
  - A message can override the TSQ name using the `Q=nnnn` prefix.
    - The check is `SUBSTR(WRITE_MSG_MSG,1,2) = 'Q='`.
    - The 4 characters at positions 3-6 replace `STSQ` positions 5-8, so the queue name becomes `GENAnnnn`.
    - The text from position 8 onward is moved through `TEMPO` back into `WRITE_MSG_MSG`, which removes the prefix.
    - The prefix is matched case-sensitively and only at the start of the message. A `Q=` anywhere else is treated as ordinary text.
  - The program does not validate the `nnnn` value. It is not checked for numeric content, blanks, or the validity of the resulting queue name.
  - The `Q=` prefix handling assumes a fixed 7-byte prefix. The code reads 83 bytes from position 8, but `SUBSTR(WRITE_MSG_MSG,8,83)` extends past the 90-byte field because 7 + 83 = 90. The bytes are used with the 90-byte `TEMPO`, so the tail is blank-padded.

- **Origin and identification**
  - Every record is prefixed with the originating CICS system id from `ASSIGN SYSID(WRITE_MSG_SYSID)`. The source region can therefore always be identified in the log.
  - Terminal messages carry the typed text only, and commarea messages carry the caller's text only. There is no separate timestamp, program name, or severity in the record.

- **Resource and storage behavior**
  - The TSQ write uses `NOSUSPEND`. If no storage is available, the task does not wait. The request is ignored and the task continues.
  - The TDQ write has no `NOSUSPEND` option and uses default CICS behavior.
  - The program does not retry failed writes.

- **Error handling constraints**
  - The program supplies `RESP(WS_RESP)` on every ASSIGN, RECEIVE, and WRITEQ command. This prevents CICS from abending on a failed command.
  - `WS_RESP` is never tested after any command, so failures are silently ignored.
    - A failed RECEIVE leaves the work area blank. The program still writes a record, using a length derived from the `WS_RECV_LEN` value.
    - A failed WRITEQ to the TDQ does not stop the TSQ write.
    - A failed WRITEQ to the TSQ does not stop the terminal confirmation.
  - There is no abend or error return code. The logger is designed not to disrupt the caller, which matches the repository convention that errors go to a queue rather than to an ABEND.
  - The program does not check `EIBCALEN`. It does not reject a zero-length commarea, and the repository convention of abending with `LGCA` for `EIBCALEN=0` does not apply here.

- **Terminal interaction constraints**
  - A confirmation screen is sent only when the message came from a terminal (`WS_FLAG = 'R'`).
  - The `SEND TEXT` sends a single byte, the `FILLER_X` blank, with `LENGTH(1)`.
    - It uses the `ERASE` option to clear the screen.
    - It uses `WAIT` to wait for the send to complete.
    - It uses `FREEKB` to unlock the keyboard.
  - Callers that pass a commarea (`WS_FLAG = 'C'`) never get any terminal output.

## 8. Error Handling

### 8.1 Response capture without evaluation
- Every CICS command in the program (the four ASSIGN commands, RECEIVE, WRITEQ TD and WRITEQ TS) specifies RESP and stores the result in one shared response field, WS_RESP.
  - The field is overwritten by each command and never tested afterwards.
  - Failures are therefore swallowed. The program has no branch, retry, abend or message for a non-normal response.
  - The RESP option does stop CICS from abending the task on a failed command, so the program always continues to the next step.
- The SEND TEXT command has no RESP option. A failure there would use default CICS condition handling, which is an abend of the task. The program has no handler for it.
- There are no HANDLE CONDITION, HANDLE ABEND or PL/I ON-unit constructs, so no custom exception routines exist.

### 8.2 Input source validation and routing
- The program checks whether the invoking program name, obtained from ASSIGN INVOKINGPROG, is blank. This decides where the message comes from.
  - If it is non-blank, the message is taken from the commarea and the flag is set to 'C'.
  - If it is blank, the program was started from a terminal. The message is received from the terminal and the flag is set to 'R'.
- No check is made that the commarea pointer is valid or that the commarea has data. The code relies on the caller passing a valid area of up to 90 bytes.
- No check is made that the terminal RECEIVE succeeded. If it fails, the working receive area stays at its initialized blank value, and a blank message would be logged.

### 8.3 Defaulting and fallback logic
- The target temporary storage queue name defaults to 'GENAERRS'. It is replaced by a 'GENAnnnn' name only if the message starts with the 'Q=' prefix.
  - The 4 characters after the prefix are not validated. A malformed or blank suffix produces a queue name that may be unintended.
- The queue name and the transient data queue name 'CSMT' are fixed or defaulted, which provides a predictable logging destination.
- Working fields and the output record are initialized to blanks at the start, so missing or unreadable data results in a blank message instead of garbage.
- The record length starts with a default of 80 and is adjusted. It is reduced for the terminal transaction ID, reduced for the 'Q=nnnn' prefix, and increased by 5 for the system identifier header.
  - The adjusted value is used as the write length.
  - No guard checks for a negative or out-of-range length.

### 8.4 Resource contention handling
- The write to the temporary storage queue uses NOSUSPEND, so the task does not wait if no storage is available. The code comment says the request is ignored in that case.
  - This is a deliberate degradation: the TS log entry is dropped, and the failure is silent because RESP is not checked.
- The transient data write has no equivalent option, and its outcome is also not checked.

### 8.5 Notification and logging
- The program is itself the application's error logger. It writes the message, prefixed with the originating system identifier, to two places:
  - The CSMT transient data queue, the standard CICS destination for terminal and error output.
  - The GENA temporary storage queue.
- When the program was started from a terminal (flag 'R'), it sends a minimal text screen to confirm completion. The screen is erased, waits for completion and frees the keyboard.
  - Callers invoking the logger from another program receive no confirmation or error indication.

### 8.6 Termination
- Processing ends with a normal CICS RETURN, whatever happened earlier. The program never sets a return code, abends the task or signals failure to the caller.
- The declared end-of-transaction message text (MSGEND) is never used, so it plays no part in error handling.

## 9. Examples

### 9.1 Example 1 — Program-to-Program LINK (commarea input, default queue)

**Scenario:** `LGIPDB01` encounters a Db2 error and calls `LGSTSQ` via `EXEC CICS LINK`, passing 90 bytes of message text in the commarea.

| Field | Value |
|---|---|
| Commarea (`COMMA_DATA`) | `'LGIPDB01 DB2 SELECT FAILED SQLCODE=-803              '` (padded to 90 chars) |
| `WS_INVOKEPROG` (set by CICS) | `'LGIPDB01'` |
| CICS SYSID | `'IYK3'` |

**What the program does:**
1. `ASSIGN INVOKINGPROG` returns `'LGIPDB01'` (non-blank), so `WS_FLAG = 'C'` and the message is copied from `COMMA_DATA` into `WRITE_MSG_MSG`. `WS_RECV_LEN` is set to `EIBCALEN` (90).
2. The first two characters of the message are `'LG'`, not `'Q='`, so `STSQ` stays `'GENAERRS'`.
3. `WS_RECV_LEN` is bumped by 5 (for the SYSID + filler prefix) → 95.
4. The assembled `WRITE_MSG` record (`'IYK3 LGIPDB01 DB2 SELECT FAILED SQLCODE=-803…'`) is written to TDQ `CSMT` and TS queue `GENAERRS` at length 95.
5. Because `WS_FLAG = 'C'`, no terminal `SEND TEXT` is issued; the program returns silently.

**Expected output:**
- One record appended to TDQ `CSMT`: `IYK3 LGIPDB01 DB2 SELECT FAILED SQLCODE=-803…`
- One record appended to TS queue `GENAERRS` with the same content.

---

### 9.2 Example 2 — Terminal-entered message with `Q=` prefix (custom queue)

**Scenario:** A support operator types a message directly at a CICS terminal using the transaction that invokes `LGSTSQ`. The message selects a custom TS queue.

| Field | Value |
|---|---|
| Terminal input (`WS_RECV`) | `'LGSQ Q=AUDTAUDIT POLICY REJECTED FOR CUST 0000000042'` (tranid `'LGSQ '` + 74-char data) |
| `WS_INVOKEPROG` (set by CICS) | `'        '` (blank — started from terminal) |
| `WS_RECV_LEN` after RECEIVE | `55` (example typed length) |
| CICS SYSID | `'IYK3'` |

**What the program does:**
1. `ASSIGN INVOKINGPROG` returns blanks, so the branch issues `EXEC CICS RECEIVE`. The first 5 bytes (`WS_RECV_TRANID`) are `'LGSQ '`; `WS_RECV_DATA` holds `'Q=AUDTAUDIT POLICY REJECTED FOR CUST 0000000042'`. `WS_FLAG = 'R'`, and `WS_RECV_LEN` is reduced by 5 (tranid stripped) → 50.
2. `WRITE_MSG_MSG` is set to `WS_RECV_DATA`. The first two characters are `'Q='`, so the custom-queue branch fires:
   - `SUBSTR(STSQ,5,4)` ← `SUBSTR(WRITE_MSG_MSG,3,4)` = `'AUDT'` → `STSQ` becomes `'GENAAUDT'`.
   - `TEMPO` ← characters 8–90 of the message (`'AUDIT POLICY REJECTED FOR CUST 0000000042…'`).
   - `WRITE_MSG_MSG` ← `TEMPO` (prefix stripped).
   - `WS_RECV_LEN` decremented by 7 → 43.
3. `WS_RECV_LEN` is bumped by 5 → 48.
4. `WRITE_MSG` (`'IYK3 AUDIT POLICY REJECTED FOR CUST 0000000042…'`) is written to TDQ `CSMT` and TS queue `GENAAUDT` at length 48.
5. Because `WS_FLAG = 'R'`, `EXEC CICS SEND TEXT … ERASE FREEKB` clears the terminal screen, then the program returns.

**Expected output:**
- One record appended to TDQ `CSMT`: `IYK3 AUDIT POLICY REJECTED FOR CUST 0000000042…`
- One record appended to TS queue `GENAAUDT` (not `GENAERRS`) with the same content.
- Terminal screen is erased and keyboard unlocked.

---

Generated by IBM Bob Premium Package for Z
