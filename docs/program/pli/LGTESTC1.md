## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
  - [2.1 Function Parameters and System Interface](#21-function-parameters-and-system-interface)
  - [2.2 User Inputs (BMS Map `SSMAPC1` / Terminal Input)](#22-user-inputs-bms-map-ssmapc1--terminal-input)
  - [2.3 Backend Program Interface Inputs (`LGCMAREA`)](#23-backend-program-interface-inputs-lgcmarea)
  - [2.4 External Data Sources (CICS Temporary Storage Queue)](#24-external-data-sources-cics-temporary-storage-queue)
- [3. Outputs](#3-outputs)
  - [3.1 Screen Outputs (BMS Map `SSMAPC1`)](#31-screen-outputs-bms-map-ssmapc1)
  - [3.2 Terminal Messages](#32-terminal-messages)
  - [3.3 Program-to-Program Communication Area (`COMM_AREA`)](#33-program-to-program-communication-area-comm_area)
  - [3.4 Temporary Storage Queue (`GENACNTL`)](#34-temporary-storage-queue-genacntl)
- [4. Processing Logic](#4-processing-logic)
  - [4.1 Initialisation and First-Invocation Detection](#41-initialisation-and-first-invocation-detection)
  - [4.2 Main Receive-and-Dispatch Loop](#42-main-receive-and-dispatch-loop)
  - [4.3 Option Selection and Transaction Routing (`SELECT` on `ENT1OPTO`)](#43-option-selection-and-transaction-routing-select-on-ent1opto)
  - [4.4 Error Handling Paths](#44-error-handling-paths)
  - [4.5 Terminal Navigation and Session Control](#45-terminal-navigation-and-session-control)
  - [4.6 `WRITE_GENACNTL` Internal Procedure — TS Queue Control Record Maintenance](#46-write_genacntl-internal-procedure--ts-queue-control-record-maintenance)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
- [7. Constraints](#7-constraints)
- [8. Error Handling](#8-error-handling)
- [9. Examples](#9-examples)
  - [9.1 Example 1: Inquire Customer Details (Option 1)](#91-example-1-inquire-customer-details-option-1)
  - [9.2 Example 2: Add New Customer (Option 2)](#92-example-2-add-new-customer-option-2)
  - [9.3 Example 3: Invalid Option Selection](#93-example-3-invalid-option-selection)

## 1. Purpose

Este programa CICS en PL/I (transacción SSC1) implementa el menú interactivo de clientes de la aplicación GenApp. Presenta una pantalla 3270 en la que el usuario elige consultar (opción 1), añadir (opción 2) o actualizar (opción 4) un cliente. Para cada opción delega el trabajo en los servicios de negocio LGICUS01, LGACUS01 y LGUCUS01, a través de un área de comunicación compartida. Muestra en pantalla los datos del cliente devueltos o un mensaje de confirmación o error. Rechaza cualquier otra opción con un aviso y reinicia la pantalla tras un fallo. Tras añadir un cliente con éxito, mantiene en la cola de almacenamiento temporal de control GENACNTL el rango de números de cliente emitidos, es decir, el número más bajo y el más alto. Para ello serializa el acceso a la cola. El programa conserva el contexto entre interacciones y finaliza con un mensaje de fin de transacción cuando el usuario sale.

## 2. Inputs

### 2.1 Function Parameters and System Interface
- `DFHEIPTR`
  - A CICS Execute Interface Block pointer implicitly passed to CICS application programs, providing execution context and communication details (such as `EIBCALEN`).
- `COMM_AREA_PTR`
  - A pointer to the CICS communication area (`COMM_AREA` / `LGCMAREA`), used to pass data across pseudo-conversational transactions (`SSC1`) and when linking to backend service programs.

### 2.2 User Inputs (BMS Map `SSMAPC1` / Terminal Input)
- `ENT1OPTI` / `ENT1OPTO`
  - Option selection input from the screen specifying the customer transaction to execute ('1' for Inquire, '2' for Add, '4' for Update).
- `ENT1CNOI` / `ENT1CNOO`
  - Customer number entered by the user on the screen to identify an existing customer for inquiry ('1') or update ('4') operations.
- `ENT1FNAI`
  - Customer's first name entered or modified by the user for Add ('2') and Update ('4') operations.
- `ENT1LNAI`
  - Customer's last name entered or modified by the user for Add ('2') and Update ('4') operations.
- `ENT1DOBI`
  - Customer's date of birth entered or modified by the user for Add ('2') and Update ('4') operations.
- `ENT1HNMI`
  - House name entered or modified by the user for Add ('2') and Update ('4') operations.
- `ENT1HNOI`
  - House number entered or modified by the user for Add ('2') and Update ('4') operations.
- `ENT1HPCI`
  - Postcode entered or modified by the user for Add ('2') and Update ('4') operations.
- `ENT1HP1I`
  - Primary (home) telephone number entered or modified by the user for Add ('2') and Update ('4') operations.
- `ENT1HP2I`
  - Secondary (mobile) telephone number entered or modified by the user for Add ('2') and Update ('4') operations.
- `ENT1HMOI`
  - Email address entered or modified by the user for Add ('2') and Update ('4') operations.

### 2.3 Backend Program Interface Inputs (`LGCMAREA`)
- `CA_RETURN_CODE`
  - Return code returned from invoked backend services (`LGICUS01`, `LGACUS01`, `LGUCUS01`) indicating success (`00`) or failure conditions.
- `CA_CUSTOMER_NUM`
  - Unique customer identifier generated and returned by the customer addition service (`LGACUS01`), subsequently used to update terminal output and control queues.
- `CA_FIRST_NAME`
  - Customer first name returned by the inquiry service (`LGICUS01`) to populate screen fields.
- `CA_LAST_NAME`
  - Customer last name returned by the inquiry service (`LGICUS01`) to populate screen fields.
- `CA_DOB`
  - Customer date of birth returned by the inquiry service (`LGICUS01`) to populate screen fields.
- `CA_HOUSE_NAME`
  - Customer house name returned by the inquiry service (`LGICUS01`) to populate screen fields.
- `CA_HOUSE_NUM`
  - Customer house number returned by the inquiry service (`LGICUS01`) to populate screen fields.
- `CA_POSTCODE`
  - Customer postcode returned by the inquiry service (`LGICUS01`) to populate screen fields.
- `CA_PHONE_HOME`
  - Customer home telephone number returned by the inquiry service (`LGICUS01`) to populate screen fields.
- `CA_PHONE_MOBILE`
  - Customer mobile telephone number returned by the inquiry service (`LGICUS01`) to populate screen fields.
- `CA_EMAIL_ADDRESS`
  - Customer email address returned by the inquiry service (`LGICUS01`) to populate screen fields.

### 2.4 External Data Sources (CICS Temporary Storage Queue)
- `READ_MSG` (`READ_MSG_MSG` / `READ_CUST_HIGH`)
  - Records read from the CICS Temporary Storage Queue `GENACNTL` (`STSQ_NAME`) to inspect and track high customer number control entries.

## 3. Outputs

### 3.1 Screen Outputs (BMS Map `SSMAPC1`)

- `SSMAPC1O` (BMS Output Map Structure): Delivers formatted customer data, system messages, and screen attributes back to the 3270 user terminal.
  - `ENT1CNOO` / `ENT1CNOI`: Displays the 10-digit customer number on the screen (initialised to `'0000000000'` on first entry, or updated with `CA_CUSTOMER_NUM` following an inquiry, add, or update operation).
  - `ENT1FNAI` / `ENT1FNAO`: Displays the customer's given name returned from the customer service.
  - `ENT1LNAI` / `ENT1LNAO`: Displays the customer's surname returned from the customer service.
  - `ENT1DOBI` / `ENT1DOBO`: Displays the customer's date of birth returned from the customer service.
  - `ENT1HNMI` / `ENT1HNMO`: Displays the house or building name returned from the customer service.
  - `ENT1HNOI` / `ENT1HNOO`: Displays the house number returned from the customer service.
  - `ENT1HPCI` / `ENT1HPCO`: Displays the postal code returned from the customer service.
  - `ENT1HP1I` / `ENT1HP1O`: Displays the primary/home telephone number returned from the customer service.
  - `ENT1HP2I` / `ENT1HP2O`: Displays the mobile telephone number returned from the customer service.
  - `ENT1HMOI` / `ENT1HMOO`: Displays the customer's email address returned from the customer service.
  - `ENT1OPTI` / `ENT1OPTO`: Cleared to blank after successful add or update transactions, or flagged for cursor repositioning when an invalid menu option is entered.
  - `ERRFLDO`: Displays user notification and error status messages on the terminal screen, including:
    - `'New Customer Inserted'`
    - `'Customer details updated'`
    - `'Please enter a valid option'`
    - `'No data was returned.'`
    - `'Error Adding Customer'`
    - `'Error Updating Customer'`

### 3.2 Terminal Messages

- `MSGEND`: Transmits the literal text `'Transaction ended'` to the terminal when the user presses PF3 or when a map failure occurs, terminating the session.

### 3.3 Program-to-Program Communication Area (`COMM_AREA`)

- `COMM_AREA`: Serves as the outgoing data contract passed via `EXEC CICS LINK` to orchestrator programs (`LGICUS01`, `LGACUS01`, `LGUCUS01`) and preserved across pseudo-conversational transactions (`TRANSID('SSC1')`).
  - `CA_REQUEST_ID`: Sets the target business function code (`'01ICUS'` for inquiry, `'01ACUS'` for add, `'01UCUS'` for update).
  - `CA_RETURN_CODE`: Initialised to `0` and evaluated to detect service failures or data-not-found conditions.
  - `CA_CUSTOMER_NUM`: Passes the target customer number for inquiry and update requests, or receives the newly generated customer number from `LGACUS01` after an add operation.
  - `CA_FIRST_NAME`: Passes the entered first name for customer creation or update.
  - `CA_LAST_NAME`: Passes the entered surname for customer creation or update.
  - `CA_DOB`: Passes the customer date of birth for creation or update.
  - `CA_HOUSE_NAME`: Passes the residence/house name for creation or update.
  - `CA_HOUSE_NUM`: Passes the residence/house number for creation or update.
  - `CA_POSTCODE`: Passes the uppercase postal code for creation or update.
  - `CA_PHONE_HOME`: Passes the home phone number for creation or update.
  - `CA_PHONE_MOBILE`: Passes the mobile phone number for creation or update.
  - `CA_EMAIL_ADDRESS`: Passes the email address for creation or update.

### 3.4 Temporary Storage Queue (`GENACNTL`)

- `STSQ_NAME`: Identifies the CICS Temporary Storage Queue (`'GENACNTL'`) used to maintain customer number tracking records across transactions.
  - `WRITE_MSG_E`: Writes the queue header record (`'**** GENAPP CNTL    '`) when initialising the control queue.
  - `WRITE_MSG_L` (`WRITE_MSG_LOW`): Writes the initial low customer boundary record (`'LOW CUSTOMER='` followed by `CA_CUSTOMER_NUM`) during queue initialisation.
  - `WRITE_MSG_H` (`WRITE_MSG_HIGH`): Updates or writes the high customer boundary record (`'HIGH CUSTOMER='` followed by `CA_CUSTOMER_NUM`) with the newly assigned customer identifier upon a successful customer addition.

## 4. Processing Logic

### 4.1 Initialisation and First-Invocation Detection

- On the very first invocation the CICS EIB field `EIBCALEN` is zero, signalling that no COMMAREA was passed
  - A local base COMMAREA is established by pointing `COMM_AREA_PTR` at `BASE_COMM_AREA`
  - Both the input (`SSMAPC1I`) and output (`SSMAPC1O`) BMS map structures are cleared to spaces
  - `CA_REQUEST_ID`, `CA_RETURN_CODE`, `CA_CUSTOMER_NUM`, and `CA_REQUEST_INITIALISER` are reset to their null/zero values
  - `ENT1CNOO` (the customer number output field) is pre-filled with `'0000000000'`
  - The customer BMS map `SSMAPC1` (mapset `SSMAP`) is sent to the terminal with `ERASE` to present a clean screen before the program suspends

### 4.2 Main Receive-and-Dispatch Loop

- After the first-time send, or on every subsequent pseudo-conversational re-entry, execution falls through to label `A_GAIN`
- `EXEC CICS HANDLE AID CLEAR(CLEARIT) PF3(ENDIT)` registers attention-key handlers:
  - Pressing **CLEAR** routes control to `CLEARIT`
  - Pressing **PF3** routes control to `ENDIT`
- `EXEC CICS HANDLE CONDITION MAPFAIL(ENDIT)` routes a MAPFAIL condition (e.g., no data entered) to `ENDIT`
- `EXEC CICS RECEIVE MAP` reads the terminal input back into `SSMAPC1I` from mapset `SSMAP`

### 4.3 Option Selection and Transaction Routing (`SELECT` on `ENT1OPTO`)

The single-character option field `ENT1OPTO` drives a `SELECT` statement that dispatches to one of three customer operations or an error path:

- **Option `'1'` – Inquire Customer**
  - Sets `CA_REQUEST_ID = '01ICUS'` and loads `CA_CUSTOMER_NUM` from the screen field `ENT1CNOO`
  - Issues `EXEC CICS LINK PROGRAM('LGICUS01')` passing the 32 500-byte COMMAREA
  - If `CA_RETURN_CODE > 0`, branches to `NO_DATA` (error path)
  - On success, maps returned COMMAREA fields back to the screen output map:
    - `CA_FIRST_NAME` → `ENT1FNAI`, `CA_LAST_NAME` → `ENT1LNAI`, `CA_DOB` → `ENT1DOBI`
    - `CA_HOUSE_NAME` → `ENT1HNMI`, `CA_HOUSE_NUM` → `ENT1HNOI`, `CA_POSTCODE` → `ENT1HPCI`
    - `CA_PHONE_HOME` → `ENT1HP1I`, `CA_PHONE_MOBILE` → `ENT1HP2I`, `CA_EMAIL_ADDRESS` → `ENT1HMOI`
  - Sends the populated map back to the terminal and branches to `ENDIT_STARTIT`

- **Option `'2'` – Add Customer**
  - Sets `CA_REQUEST_ID = '01ACUS'` and zeros `CA_CUSTOMER_NUM` (new customer)
  - Loads all customer detail fields from screen input fields into the corresponding COMMAREA fields (`CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_DOB`, `CA_HOUSE_NAME`, `CA_HOUSE_NUM`, `CA_POSTCODE`, `CA_PHONE_HOME`, `CA_PHONE_MOBILE`, `CA_EMAIL_ADDRESS`)
  - Performs two data-cleansing operations on the raw COMMAREA before linking:
    - Translates all null bytes (`'00'X`) to spaces via `TRANSLATE(COMM_AREA_RAW,' ','00'X)`
    - Converts `CA_POSTCODE` to uppercase with `UPPERCASE(CA_POSTCODE)`
  - Issues `EXEC CICS LINK PROGRAM('LGACUS01')` passing the COMMAREA
  - If `CA_RETURN_CODE > 0`, issues `EXEC CICS SYNCPOINT ROLLBACK` to undo any partial work, then branches to `NO_ADD`
  - On success:
    - Calls the internal procedure `WRITE_GENACNTL` to record the new customer number in the control TS queue
    - Reflects the assigned customer number back to the screen (`ENT1CNOI = CA_CUSTOMER_NUM`)
    - Clears the option field (`ENT1OPTI = ' '`) and sets the error field to `'New Customer Inserted'`
    - Sends the updated map and branches to `ENDIT_STARTIT`

- **Option `'4'` – Update Customer**
  - First performs a read (inquire) by setting `CA_REQUEST_ID = '01ICUS'`, loading `CA_CUSTOMER_NUM`, and linking to `LGICUS01`
  - If `CA_RETURN_CODE > 0`, branches to `NO_DATA`
  - On successful inquiry, populates all screen fields with the current customer data and sends the map, allowing the user to review and edit
  - Issues a second `EXEC CICS RECEIVE MAP` to collect the user's modifications
  - Sets `CA_REQUEST_ID = '01UCUS'` and repopulates all COMMAREA fields from the (now-modified) screen fields
  - Again applies null-to-space translation and postcode uppercasing before linking
  - Issues `EXEC CICS LINK PROGRAM('LGUCUS01')` for the update
  - If `CA_RETURN_CODE > 0`, branches to `NO_UPD`
  - On success, refreshes `ENT1CNOI`, clears the option field, sets `ERRFLDO = 'Customer details updated'`, sends the map, and branches to `ENDIT_STARTIT`

- **OTHERWISE** – Invalid Option
  - Sets `ERRFLDO = 'Please enter a valid option'` and `ENT1OPTL = -1` (cursor positioning attribute)
  - Sends the map with `CURSOR` to position the cursor at the option field and branches to `ENDIT_STARTIT`

### 4.4 Error Handling Paths

- **`NO_UPD`**: sets `ERRFLDO = 'Error Updating Customer'`, falls through to `ERROR_OUT`
- **`NO_ADD`**: sets `ERRFLDO = 'Error Adding Customer'`, falls through to `ERROR_OUT`
- **`NO_DATA`**: sets `ERRFLDO = 'No data was returned.'`, falls through to `ERROR_OUT`
- **`ERROR_OUT`**:
  - Sends the current map (showing the error message) to the terminal
  - Clears both map structures, resets the full COMMAREA raw area and key COMMAREA fields (`CA_REQUEST_ID`, `CA_RETURN_CODE`, `CA_CUSTOMER_NUM`, `CA_REQUEST_INITIALISER`) to null/zero values
  - Branches to `ENDIT_STARTIT` to restart the pseudo-conversational cycle

### 4.5 Terminal Navigation and Session Control

- **`ENDIT_STARTIT`**: issues `EXEC CICS RETURN TRANSID('SSC1') COMMAREA(COMM_AREA)` — suspends and schedules a re-entry under the same transaction ID, preserving the COMMAREA state across pseudo-conversational interactions
- **`ENDIT`**: sends the literal text `'Transaction ended'` via `EXEC CICS SEND TEXT … ERASE FREEKB`, then issues `EXEC CICS RETURN` with no TRANSID, ending the transaction cleanly
- **`CLEARIT`**: blanks `SSMAPC1I`, redisplays the empty map using `MAPONLY` (no data overlay), then returns with `TRANSID('SSC1')` and the COMMAREA to remain in the same transaction

### 4.6 `WRITE_GENACNTL` Internal Procedure — TS Queue Control Record Maintenance

This procedure serialises access to the `GENACNTL` temporary storage queue and maintains the high-customer-number record:

- **Serialisation**: acquires an exclusive CICS enqueue on `STSQ_NAME` (`'GENACNTL'`) before touching the queue
- **Read and scan loop**:
  - Reads item 1 of the queue to check whether it exists (`DFHRESP(NORMAL)`)
  - If the queue exists, iterates forward using `READQ TS … NEXT` in a `DO WHILE (WS_RESP = 0)` loop
  - For each item read, increments `WS_ITEM_COUNT`
  - If the item's first 13 characters equal `'HIGH CUSTOMER'`, the program has found the high-customer record:
    - Sets `WRITE_MSG_HIGH = CA_CUSTOMER_NUM` (the newly assigned customer number)
    - Clears `WS_FLAG_TSQH` to `' '` to record that the record was found and updated
    - Rewrites that specific item in place with `WRITEQ TS … REWRITE ITEM(WS_ITEM_COUNT)`
    - Forces `WS_RESP = 99` to exit the loop immediately
- **Queue creation path** (queue not found or high-customer record not located, `WS_FLAG_TSQH` still `'Y'`):
  - Writes a header record (`WRITE_MSG_E` — `'**** GENAPP CNTL    '`) as a new item (`NOSUSPEND`)
  - Sets both `WRITE_MSG_LOW` and `WRITE_MSG_HIGH` to `CA_CUSTOMER_NUM`, initialising both bounds to the first customer number
  - Writes the low-customer record (`WRITE_MSG_L`) and the high-customer record (`WRITE_MSG_H`) as additional new items
- **Release**: issues `EXEC CICS DEQ` on `STSQ_NAME` to release the enqueue lock

## 5. Paragraphs

**LGTESTC1 (main procedure, entry and initial display)**
- Purpose: Main routine of the CICS customer menu transaction SSC1. It is the pseudo-conversational driver for customer inquiry, add and update. It is declared `Options(Main,Reentrant)` and receives `DFHEIPTR` and `COMM_AREA_PTR`.
- Operations:
  - Declares the working storage. This includes the message-end text, the TS queue control records `READ_MSG` and `WRITE_MSG`, the queue name `STSQ_NAME` ('GENACNTL'), and the BMS map structures from the SSMAP include.
  - Declares the commarea layout from the LGCMAREA include and a `BASE_COMM_AREA` buffer sized to it.
  - If `EIBCALEN = 0` (first entry), it points `COMM_AREA_PTR` at `BASE_COMM_AREA`.
  - It then blanks the map areas `SSMAPC1I` and `SSMAPC1O`. It clears `CA_REQUEST_ID`, `CA_RETURN_CODE`, `CA_CUSTOMER_NUM` and the request initialiser area. It sets `ENT1CNOO` to '0000000000'.
- Control flow: A single IF decides whether this is the first invocation. The commented-out LINK to ADDIPGM1 is not executed. After the IF block, execution falls through to `A_GAIN`.
- External interaction: `EXEC CICS SEND MAP('SSMAPC1') MAPSET('SSMAP') ERASE` shows the main menu on the first call.

**A_GAIN (receive and dispatch)**
- Purpose: Receives the user's input and dispatches the requested customer transaction.
- Operations:
  - Registers `HANDLE AID` so CLEAR goes to `CLEARIT` and PF3 goes to `ENDIT`.
  - Registers `HANDLE CONDITION MAPFAIL(ENDIT)`.
  - Receives the map `SSMAPC1` into `SSMAPC1I`.
  - Evaluates `ENT1OPTO` in a SELECT statement.
- Control flow: SELECT with the branches below. After the SELECT, the code executes `EXEC CICS RETURN` as a fall-through. In practice every branch ends in a GO TO, so this RETURN is a safeguard.
  - **WHEN '1' (inquire customer)**
    - Sets `CA_REQUEST_ID` to '01ICUS' and `CA_CUSTOMER_NUM` to `ENT1CNOO`.
    - Links to LGICUS01 with the commarea (length 32500).
    - If `CA_RETURN_CODE > 0`, it goes to `NO_DATA`.
    - Otherwise it copies first name, last name, DOB, house name, house number, postcode, home phone, mobile phone and email from the commarea to the map input fields.
    - Sends the map and goes to `ENDIT_STARTIT`.
  - **WHEN '2' (add customer)**
    - Sets `CA_REQUEST_ID` to '01ACUS' and `CA_CUSTOMER_NUM` to 0.
    - Moves the screen fields into the commarea.
    - Translates nulls (X'00') to blanks in `COMM_AREA_RAW` and uppercases `CA_POSTCODE`.
    - Links to LGACUS01.
    - If `CA_RETURN_CODE > 0`, it issues `SYNCPOINT ROLLBACK` and goes to `NO_ADD`.
    - On success it calls `WRITE_GENACNTL` to update the control queue.
    - It then sets `ENT1CNOI` to the new customer number, clears `ENT1OPTI`, and sets `ERRFLDO` to 'New Customer Inserted'.
    - Sends the map and goes to `ENDIT_STARTIT`.
  - **WHEN '4' (update customer)**
    - First performs the same inquiry as option 1 (LGICUS01 link, error check to `NO_DATA`, fields moved to the screen).
    - Sends the map with the current data and issues a second `RECEIVE MAP` to get the user's edits.
    - Sets `CA_REQUEST_ID` to '01UCUS' and moves the edited fields into the commarea.
    - Translates nulls to blanks and uppercases the postcode.
    - Links to LGUCUS01.
    - If `CA_RETURN_CODE > 0`, it goes to `NO_UPD`.
    - Otherwise it sets `ENT1CNOI`, clears `ENT1OPTI`, and sets `ERRFLDO` to 'Customer details updated'.
    - Sends the map and goes to `ENDIT_STARTIT`.
  - **OTHERWISE (invalid option)**
    - Sets `ERRFLDO` to 'Please enter a valid option' and `ENT1OPTL` to -1 to position the cursor.
    - Sends the map with CURSOR and goes to `ENDIT_STARTIT`.
- External interaction: CICS terminal I/O (RECEIVE MAP, SEND MAP) and LINKs to the LGICUS01, LGACUS01 and LGUCUS01 services, which perform the customer data access.

**ENDIT_STARTIT**
- Purpose: Normal pseudo-conversational exit that restarts the transaction on the next user input.
- Operations: `EXEC CICS RETURN TRANSID('SSC1') COMMAREA(COMM_AREA)`. This passes the commarea to the next invocation.
- Control flow: Reached only through GO TO from the dispatch branches and from `ERROR_OUT`. There is no fall-through into it.
- External interaction: CICS task return and transaction scheduling.

**ENDIT**
- Purpose: Terminates the transaction when the user presses PF3 or a MAPFAIL occurs.
- Operations: Sends the text 'Transaction ended' (`MSGEND`) with ERASE and FREEKB, then issues `EXEC CICS RETURN`.
- Control flow: Reached through the HANDLE AID PF3 and HANDLE CONDITION MAPFAIL branches.
- External interaction: Terminal output through SEND TEXT.

**CLEARIT**
- Purpose: Handles the CLEAR key by resetting the screen.
- Operations: Blanks `SSMAPC1I`, sends the map with MAPONLY to show an empty menu, then returns with TRANSID('SSC1') and the commarea.
- Control flow: Reached through the HANDLE AID CLEAR branch.
- External interaction: Terminal output (SEND MAP) and CICS RETURN.

**NO_UPD**
- Purpose: Error label for a failed customer update.
- Operations: Sets `ERRFLDO` to 'Error Updating Customer' and goes to `ERROR_OUT`.
- Control flow: Reached through GO TO when the update returns `CA_RETURN_CODE > 0`.

**NO_ADD**
- Purpose: Error label for a failed customer add.
- Operations: Sets `ERRFLDO` to 'Error Adding Customer' and goes to `ERROR_OUT`.
- Control flow: Reached through GO TO after the rollback.

**NO_DATA**
- Purpose: Error label for an inquiry that returned no data. It is used by both option 1 and option 4.
- Operations: Sets `ERRFLDO` to 'No data was returned.' and goes to `ERROR_OUT`.
- Control flow: Reached through GO TO when the inquiry returns `CA_RETURN_CODE > 0`.

**ERROR_OUT**
- Purpose: Common error display and reset routine.
- Operations:
  - Sends the map with the error message.
  - Blanks the map areas and `COMM_AREA_RAW`.
  - Resets `CA_REQUEST_ID` to blank and `CA_RETURN_CODE` and `CA_CUSTOMER_NUM` to 0.
  - Clears the request initialiser.
  - Goes to `ENDIT_STARTIT`.
- Control flow: Unconditional GO TO at the end, so the transaction restarts with a clean commarea.
- External interaction: Terminal output through SEND MAP.

**WRITE_GENACNTL (internal procedure)**
- Purpose: Updates the GENACNTL temporary storage queue after a successful customer add. The queue holds the customer number range (low and high).
- Operations:
  - Takes an exclusive lock on the queue name with `EXEC CICS ENQ RESOURCE(STSQ_NAME)`.
  - Initialises `WS_FLAG_TSQH` to 'Y' and `WS_ITEM_COUNT` to 1.
  - Reads item 1 of the queue into `READ_MSG` with RESP(`WS_RESP`).
  - If the read is NORMAL, it loops while `WS_RESP = 0`:
    - Reads the next item and increments `WS_ITEM_COUNT`.
    - If the read is NORMAL and the first 13 characters of the item are 'HIGH CUSTOMER', it sets `WRITE_MSG_HIGH` to `CA_CUSTOMER_NUM` and clears `WS_FLAG_TSQH`.
    - It then rewrites that item with `WRITEQ TS ... REWRITE ITEM(WS_ITEM_COUNT)` from `WRITE_MSG_H` (length 24).
    - It sets `WS_RESP` to 99 to exit the loop.
  - If `WS_FLAG_TSQH` is still 'Y', no high-customer record was found, so the queue is initialised. It writes three items with NOSUSPEND:
    - The header `WRITE_MSG_E` (length 20).
    - The low-customer record `WRITE_MSG_L` (length 23), after setting `WRITE_MSG_LOW` and `WRITE_MSG_HIGH` to `CA_CUSTOMER_NUM`.
    - The high-customer record `WRITE_MSG_H` (length 24).
  - Releases the lock with `EXEC CICS DEQ RESOURCE(STSQ_NAME)`.
- Control flow: Nested IFs, a DO WHILE loop over the queue items, and a flag-driven branch that creates the queue contents if the high record is missing.
- External interaction: CICS enqueue/dequeue for serialisation, and temporary storage queue reads (READQ TS) and writes (WRITEQ TS, including REWRITE).

## 6. Dependencies

- **Called programs (CICS LINK)**
  - `LGICUS01`
    - Customer inquiry service.
    - Linked with `CA_REQUEST_ID` = '01ICUS' for option 1 (inquire) and for option 4 (update), which fetches the current details first.
    - Reads the customer number from `CA_CUSTOMER_NUM`.
    - Returns the customer's name, date of birth, address, phone and email fields and a return code.
  - `LGACUS01`
    - Customer add service.
    - Linked with `CA_REQUEST_ID` = '01ACUS' and `CA_CUSTOMER_NUM` = 0.
    - Receives the customer details from the screen.
    - Returns the newly assigned customer number in `CA_CUSTOMER_NUM`.
  - `LGUCUS01`
    - Customer update service.
    - Linked with `CA_REQUEST_ID` = '01UCUS' and the edited customer details.
    - Reports success or failure in `CA_RETURN_CODE`.
  - `ADDIPGM1`
    - Counter-initialisation program that would be linked on first entry.
    - The `EXEC CICS LINK` is commented out, so it is not currently a runtime dependency.
  - All LINK calls use `COMM_AREA` as the COMMAREA, with a length of 32500.
  - Every service reports its outcome in `CA_RETURN_CODE`. A value greater than 0 sends the menu to the error paths (`NO_DATA`, `NO_ADD`, `NO_UPD`). After a failed add, the menu also issues `SYNCPOINT ROLLBACK`.

- **Internal procedure**
  - `WRITE_GENACNTL`
    - Internal procedure called after a successful add.
    - It maintains the customer number range control information in the `GENACNTL` temporary storage queue.

- **Include files and data contracts**
  - `SSMAP` (`plinc/SSMAP.inc`)
    - PL/I symbolic map for mapset `SSMAP`.
    - Declares the input and output structures for `SSMAPC1` (customer menu) and `SSMAPP1` to `SSMAPP5`.
    - Supplies the screen fields used by this program: `ENT1OPTI`, `ENT1OPTO`, `ENT1CNOI`, `ENT1CNOO`, `ENT1FNAI`, `ERRFLDO` and others.
  - `LGCMAREA` (`plinc/LGCMAREA.inc`)
    - COMMAREA layout, 32500 bytes in total, shared with the called services.
    - Defines `COMM_AREA_PTR`, `COMM_AREA`, `CA_REQUEST_ID`, `CA_RETURN_CODE`, `CA_CUSTOMER_NUM` and the customer fields.
    - The customer fields are `CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_DOB`, `CA_HOUSE_NAME`, `CA_HOUSE_NUM`, `CA_POSTCODE`, `CA_PHONE_HOME`, `CA_PHONE_MOBILE` and `CA_EMAIL_ADDRESS`.
    - It is the only data contract between this menu and the services it links to.
  - Include resolution
    - The compile parameters resolve `%INCLUDE` names against the `plinc/*.inc` search path.
    - A change to either include requires an impact build or full build.

- **Input parameters and runtime context**
  - `DFHEIPTR`
    - Pointer to the CICS EXEC interface block.
    - Supplies `EIBCALEN`, used to detect the first invocation (`EIBCALEN` = 0).
  - `COMM_AREA_PTR`
    - COMMAREA pointer passed in by CICS and carried between pseudo-conversational invocations.
    - On first entry, it is set to the local `BASE_COMM_AREA` storage.
  - Terminal input from mapset `SSMAP`
    - Map `SSMAPC1` is received from the terminal.
    - Menu option `ENT1OPTO` selects the function: '1' inquire, '2' add, '4' update. Any other value is rejected.
    - Customer details are keyed into the screen fields.
  - Terminal attention keys and conditions
    - `CLEAR` branches to the `CLEARIT` routine.
    - `PF3` branches to `ENDIT`.
    - The `MAPFAIL` condition branches to `ENDIT`.
  - `DFHRESP(NORMAL)`
    - CICS response constant, used to test queue read results.

- **CICS terminal and BMS resources**
  - Mapset `SSMAP`, map `SSMAPC1`
    - BMS source is not in this repository.
    - Used with `SEND MAP`, `RECEIVE MAP` (ASIS TERMINAL) and `SEND MAP ... MAPONLY`.
    - Menu, result and error messages are sent through it.
  - Transaction `SSC1`
    - Re-invoked by `EXEC CICS RETURN TRANSID('SSC1') COMMAREA(COMM_AREA)` to continue the pseudo-conversation.
    - Needs a CSD definition that points to this program.
  - `SEND TEXT`
    - Sends the end-of-transaction message held in `MSGEND` ('Transaction ended') on PF3 or MAPFAIL.
  - CICS syncpoint services
    - `SYNCPOINT ROLLBACK` is issued when the add service returns a failure code.

- **Temporary storage queue and serialisation**
  - TS queue `GENACNTL` (held in `STSQ_NAME`)
    - Holds the customer number range control records.
    - Records: the 20-byte header '**** GENAPP CNTL', 'LOW CUSTOMER=nnnnnnnnnn' and 'HIGH CUSTOMER=nnnnnnnnnn'.
    - Operations:
      - `READQ TS` by item and with NEXT, into `READ_MSG`, to find the 'HIGH CUSTOMER' record.
      - `WRITEQ TS ... REWRITE` of `WRITE_MSG_H`, to update the high customer number.
      - `WRITEQ TS ... NOSUSPEND` of `WRITE_MSG_E`, `WRITE_MSG_L` and `WRITE_MSG_H`, to create the queue when it does not exist.
    - Outcomes are captured in `WS_RESP`.
  - `ENQ` / `DEQ` on resource `GENACNTL`
    - Serialises updates to the queue so concurrent adds do not corrupt the high customer number.
    - Uses the `STSQ_NAME` value and its length.

- **Data dependencies on other programs' results**
  - `CA_CUSTOMER_NUM` returned by `LGACUS01` after an add
    - Used to set `WRITE_MSG_LOW` and `WRITE_MSG_HIGH`, and displayed on the menu.
  - Customer details returned by `LGICUS01`
    - Populate the update screen and the inquiry display.

- **Downstream resources reached indirectly**
  - Db2 tables `CUSTOMER` and `POLICY`, and the VSAM and security components, are reached through the linked services.
  - The menu has no SQL or file I/O of its own.
  - `LGACUS01` writes `CUSTOMER` and `POLICY`. `LGICUS01` reads `CUSTOMER`. `LGUCUS01` updates `CUSTOMER`.
  - The customer number counter (`GENACUSTNUM`, pool `GENA`) is used on the add path by the services further down the chain.

- **Built-in functions and runtime environment**
  - `TRANSLATE`
    - Replaces X'00' nulls with blanks in `COMM_AREA_RAW` before the add and update links.
  - `UPPERCASE`
    - Converts `CA_POSTCODE` to uppercase before the add and update links.
  - `STG`, `ADDR` and `SUBSTR`
    - Used for lengths, pointers and queue-record matching.
  - Compile and runtime settings
    - Compiled with `SYSTEM(CICS)` and the CICS and macro preprocessors.
    - Declared `Options(Main,Reentrant)`, so it must run as a reentrant CICS program.
    - Deployed as a `CICSLOAD` module.
  - Configuration values
    - No environment variables are used.
    - The hard-coded literals are mapset `SSMAP`, map `SSMAPC1`, transaction `SSC1`, queue `GENACNTL`, the program names above and the COMMAREA length of 32500.

## 7. Constraints

- **Menu option validation**
  - Only three transaction options are accepted. The program evaluates `ENT1OPTO` in a `SELECT` statement:
    - '1' inquires on a customer.
    - '2' adds a customer.
    - '4' updates a customer.
  - Any other value, including '3', blank and non-digits, goes to the `OTHERWISE` branch:
    - The program sets `ERRFLDO` to "Please enter a valid option".
    - It sets `ENT1OPTL = -1` to place the cursor on the option field.
    - It redisplays the map with the `CURSOR` option.
    - It returns to the `SSC1` transaction without calling any service.
  - Inquiry and update both use the customer number from the screen (`ENT1CNOO`). The program does not check that the number is present, numeric or in range. Those checks are left to the called services.

- **Input data handling and formatting**
  - **Null-to-blank translation**
    - For add and update, the program runs `TRANSLATE(COMM_AREA_RAW,' ','00'X)` over the whole commarea before the link.
    - This replaces every null byte (X'00') with a blank, so unmodified screen fields do not pass nulls to the services.
  - **Postcode uppercasing**
    - For add and update, `CA_POSTCODE = UPPERCASE(CA_POSTCODE)` runs before the link.
    - The postcode is therefore always uppercase when the service receives it.
  - **Customer number on add**
    - `CA_CUSTOMER_NUM` is forced to 0 for an add. The user cannot choose the number, and the called service assigns it.
  - **Field size limits**
    - The screen and commarea fields have fixed lengths:
      - first name: 10
      - last name: 20
      - date of birth: 10
      - house name: 20
      - house number: 4
      - postcode: 8
      - phone numbers: 20 each
      - email: 100
      - customer number: 10 digits (`PIC '9999999999'`)
    - Values are copied into these fixed fields by assignment, so any other length is truncated or padded. The program does not report this.

- **Request identification and commarea contract**
  - **Request codes**
    - The program sets `CA_REQUEST_ID` to a fixed 6-character code before every link:
      - '01ICUS' for inquiry. Option 4 sends it first, to fetch the current data.
      - '01ACUS' for add.
      - '01UCUS' for update.
    - Each code is paired with its target program:
      - `LGICUS01` for inquiry
      - `LGACUS01` for add
      - `LGUCUS01` for update
  - **Commarea size**
    - Every link passes `COMMAREA(COMM_AREA)` with `LENGTH(32500)`.
    - The commarea is 32500 bytes (`COMM_AREA_RAW CHAR(32500)`). It is a union, so every branch must stay within that size.
    - Reserved space is held in the request-specific filler, and its length must be kept in step with any field added.
  - **Return code convention**
    - `CA_RETURN_CODE` is `PIC '99'`. A value greater than 0 means the service failed.
    - The program does not tell individual failure codes apart.

- **Conditional processing rules**
  - **Inquiry (option 1)**
    - If the return code is above 0, the program shows "No data was returned." through `NO_DATA`.
    - Otherwise it copies the returned fields to the map and sends it.
    - The map is sent without `ERASE`.
  - **Update (option 4)**
    - Inquiry must succeed first. The program links to `LGICUS01`, and a return code above 0 goes to `NO_DATA`.
    - The user then edits the displayed data. Only after the second `RECEIVE MAP` does the program link to `LGUCUS01`.
    - If the update returns a code above 0, the program shows "Error Updating Customer" through `NO_UPD`.
    - On success it shows "Customer details updated". It also copies `CA_CUSTOMER_NUM` to `ENT1CNOI` and clears `ENT1OPTI` to blank.
    - The update takes the customer number from `ENT1CNOI`, the second receive. The earlier inquiry used `ENT1CNOO`.
  - **Add (option 2)**
    - If the return code is above 0, the program issues `EXEC CICS SYNCPOINT ROLLBACK`. It then shows "Error Adding Customer" through `NO_ADD`.
    - On success it calls `WRITE_GENACNTL` before building the response. The sequence is:
      - It shows the new customer number.
      - It clears the option field.
      - It shows "New Customer Inserted".
  - **Error exit**
    - `ERROR_OUT` sends the error map, then resets all of these:
      - the input and output maps
      - the whole commarea
      - the request ID
      - the return code
      - the customer number
      - the request-specific initialiser
    - It then returns to `SSC1` with a clean commarea, so failed state does not carry into the next interaction.

- **Sequencing and transaction flow**
  - **First invocation**
    - If `EIBCALEN = 0`, the program points `COMM_AREA_PTR` at the local `BASE_COMM_AREA` storage. It then initialises the maps and the commarea fields and sends the menu with `ERASE`.
    - The customer number field is pre-set to '0000000000'.
  - **Receive before processing**
    - `HANDLE AID` and `HANDLE CONDITION` are established before the `RECEIVE MAP`. The map is received with `ASIS` and `TERMINAL`, so the text is not folded to uppercase.
  - **Pseudo-conversational design**
    - After each processed request, the program ends with `RETURN TRANSID('SSC1') COMMAREA(COMM_AREA)`.
    - The terminal state therefore passes to the next iteration through the commarea.
  - **Update uses two receives**
    - Option 4 does `SEND MAP`, then `RECEIVE MAP`, within one task. It waits for the user's edits before it sends the update.
  - **Rollback before error display**
    - On an add failure, the rollback happens before the error message is shown.
  - **Control-queue update after add**
    - `WRITE_GENACNTL` runs only after a successful add, and before the confirmation map is sent.

- **Terminal key and exception handling**
  - **PF3**
    - PF3 branches to `ENDIT`. The program sends the text "Transaction ended" with `ERASE` and `FREEKB`, then returns without a transaction ID. The transaction ends.
  - **CLEAR**
    - CLEAR branches to `CLEARIT`. The program blanks the input map, sends the map with `MAPONLY` and returns to `SSC1`.
  - **MAPFAIL**
    - `MAPFAIL` branches to `ENDIT`. If the user presses Enter without entering data, the transaction ends.
  - **Other conditions**
    - Other CICS conditions are not handled. Examples are the `LINK` conditions (PGMIDERR and the like), `SEND MAP` errors and the `ENQ`/`DEQ` calls.
    - The program has no `RESP` checks on them, so the default CICS action (abend) applies.

- **Control queue (GENACNTL) consistency and serialisation**
  - **Serialised access**
    - `WRITE_GENACNTL` takes `EXEC CICS ENQ RESOURCE(STSQ_NAME)` (resource 'GENACNTL', length from `STG`) before reading or writing the queue. It releases it with `DEQ` at the end.
    - Only one task at a time can update the customer number range.
  - **Queue layout**
    - The queue holds three items:
      - a header, `**** GENAPP CNTL`, 20 bytes
      - the low customer number, 'LOW CUSTOMER=' plus 10 digits, 23 bytes
      - the high customer number, 'HIGH CUSTOMER=' plus 10 digits, 24 bytes
    - The read area `READ_MSG_MSG` is 80 bytes.
  - **Locating the high-customer item**
    - The program reads item 1, then reads sequentially with `NEXT`. It tracks the item number in `WS_ITEM_COUNT`.
    - It matches an item by the first 13 characters of the message, `SUBSTR(READ_MSG_MSG,1,13) = 'HIGH CUSTOMER'`, and only when `WS_RESP` is normal.
    - On a match it rewrites that item in place:
      - `WRITE_MSG_HIGH` is set to `CA_CUSTOMER_NUM`.
      - The `WRITEQ TS REWRITE` uses `ITEM(WS_ITEM_COUNT)` and length 24.
      - It sets `WS_RESP = 99` to end the loop.
    - A read-ahead test can reach the end of the queue. The loop ends when `WS_RESP` is not 0.
  - **Queue initialisation**
    - If the first read fails, or no HIGH item is found, `WS_FLAG_TSQH` stays 'Y'. The program then creates the queue contents with three `WRITEQ TS ... NOSUSPEND` calls:
      - the header (20 bytes)
      - the low entry (23 bytes)
      - the high entry (24 bytes)
    - It sets both `WRITE_MSG_LOW` and `WRITE_MSG_HIGH` to the new customer number. The first customer becomes both the low and the high value.
  - **High-water mark rule**
    - The high customer number is overwritten with the number from the latest add.
    - The program does not compare it with the current high value. It assumes that numbers only increase.
    - The low value is never changed after the queue is created.
  - **Unchecked responses**
    - `RESP` is captured on the queue writes but not tested after them. A failed queue update does not stop the add or change the reply to the user.

- **Resource and capacity limits**
  - **Commarea**
    - The commarea is limited to 32500 bytes. The inquiry, add and update services all use this one size.
  - **Customer number**
    - It is limited to 10 digits (`PIC '9999999999'`).
  - **Queue read buffer**
    - `READ_MSG` is limited to 80 bytes.
  - **Queue item count**
    - `WS_ITEM_COUNT` is `FIXED BIN(15)`, so it is limited to 32767 items.
  - **Error message**
    - The error and info message field `ERRFLDO` is limited to 40 characters.
  - **Fixed names**
    - The map is `SSMAPC1` in mapset `SSMAP`. The queue is `GENACNTL`. The re-entry transaction is `SSC1`.
  - **Reentrancy**
    - The program is `Options(Main,Reentrant)`.
    - The queue name and the exit message are `STATIC` constants, and no other static state is changed.

## 8. Error Handling

-  **Input validation and menu option checking**
  - The option selector is evaluated in a multi-way selection. Only '1' (inquire), '2' (add) and '4' (update) are accepted.
  - Any other value goes to the default branch. That branch sets the error field to "Please enter a valid option" and positions the cursor on the option field by setting its length indicator to -1.
  - The menu is redisplayed with the cursor on the option field, and the transaction restarts through the pseudo-conversational return.
  - On the first invocation (empty communication area length), the screen maps and communication area are cleared. The customer number is set to zeros, and a clean menu is sent. This is a defensive initialisation that prevents stale or garbage data from being processed.

-  **Terminal and screen-level exception handling (CICS condition and AID handling)**
  - A handler for the CLEAR key routes control to the CLEARIT routine. That routine blanks the input map, resends the map with the map-only option, and returns to the same transaction with the communication area retained.
  - A handler for PF3 routes control to the ENDIT routine. That routine sends the "Transaction ended" text, erases the screen, frees the keyboard and returns without a transaction id, which ends the conversation.
  - A handler for the MAPFAIL condition also routes control to ENDIT. When the terminal returns no map data, the transaction is terminated cleanly instead of abending.
  - These handlers are established at the A_GAIN label, before each receive of the map.

-  **Business service return code checks**
  - After every link to a called service, the return code in the communication area is tested. A value above zero means the service failed.
    - Inquire (option 1): a non-zero return code branches to the NO_DATA path. No screen fields are populated from the failed response.
    - Add (option 2): a non-zero return code triggers a syncpoint rollback and then branches to the NO_ADD path.
    - Update (option 4): the initial inquiry (used to pre-populate the screen) is checked and branches to NO_DATA on failure. The subsequent update link is checked and branches to NO_UPD on failure.
  - The add path is the only one that explicitly rolls back the unit of work. The rollback undoes any partial database changes made by the called add service before the user is told of the failure.
  - The control queue update (WRITE_GENACNTL) runs only after a successful add, so the high customer number is never advanced for a failed add.

-  **User notification and error messages**
  - Dedicated error labels set a message in the error field of the output map:
    - "No data was returned." for a failed inquiry, or a failed pre-update retrieval.
    - "Error Adding Customer" for a failed add.
    - "Error Updating Customer" for a failed update.
    - "Please enter a valid option" for an invalid menu selection.
  - All error labels fall through to a common ERROR_OUT routine that sends the menu with the message displayed.
  - Success messages follow the same mechanism: "New Customer Inserted" and "Customer details updated". These are also placed in the error field. After a successful add or update, the option field is cleared and the customer number field is refreshed from the communication area.

-  **Recovery, reset and fallback logic**
  - The ERROR_OUT routine, after showing the message, clears the input and output maps, the entire communication area, the request id, the return code, the customer number and the request-specific area. This avoids carrying failed or partial data into the next interaction.
  - The program then returns to the same transaction (SSC1) with the communication area, so the user can retry from a clean menu.
  - Before linking for add and update, null bytes in the communication area are translated to blanks, and the postcode is converted to uppercase. These normalisations prevent downstream rejection caused by low-values or inconsistent case.
  - The program always ends its processing path with an explicit return, either to the same transaction or with a plain return, so control never falls off the end of the main logic.

-  **Temporary storage queue error handling (WRITE_GENACNTL)**
  - Serialisation: the queue name is enqueued before reading or updating the queue and dequeued afterwards. This guards against concurrent updates corrupting the customer number range.
  - Response code checking: the first read of item 1 uses a response variable, and its value is compared with the normal response. If the queue or item does not exist, or the read fails, the scanning loop is bypassed and nothing is raised as an abend.
  - The loop reads successive items until a non-normal response is returned. This is how the end of the queue (item not found) is detected. A non-normal response is therefore used as a loop-termination signal, not as an error.
  - When the "HIGH CUSTOMER" record is found, the program rewrites that item with the new high customer number. It then forces the response variable to 99 to leave the loop.
  - A flag, initialised to 'Y', indicates that the high-customer record has not been found. If it is still 'Y' after the scan, the queue is treated as missing or uninitialised. Fallback logic then writes the header, low-customer and high-customer records, using the new customer number for both low and high values. This is self-healing initialisation of the control queue.
  - The fallback writes use the NOSUSPEND option so that the program does not wait if the queue cannot be written immediately.
  - The response values from the fallback writes and the rewrite are captured but not tested afterwards. Failures of those operations are silently ignored and do not affect the user message.

-  **Limitations and gaps in error handling**
  - There is no general ON-unit or ERROR condition handling. Unexpected runtime exceptions and unhandled CICS conditions (other than MAPFAIL) rely on default CICS abend handling.
  - The called services are not protected by response options on the link commands. A failure of the link itself, such as the program not being found, is not trapped locally.
  - The second map receive in the update path is covered by the earlier MAPFAIL and AID handlers. It has no separate failure branch.
  - The dequeue is not conditioned on the outcome of the queue operations, so the resource is always released at the end of the procedure.

## 9. Examples

### 9.1 Example 1: Inquire Customer Details (Option 1)

#### 9.1.1 Input Data
* **Map Option (`ENT1OPTO`):** `'1'`
* **Customer Number (`ENT1CNOO`):** `'0000000001'`

#### 9.1.2 Expected Output
* **First Name (`ENT1FNAI`):** `'JOHN      '`
* **Last Name (`ENT1LNAI`):** `'DOE                 '`
* **Date of Birth (`ENT1DOBI`):** `'1985-05-12'`
* **House Name (`ENT1HNMI`):** `'THE COTTAGE         '`
* **House Number (`ENT1HNOI`):** `'12        '`
* **Postcode (`ENT1HPCI`):** `'SW1A 1AA'`
* **Home Phone (`ENT1HP1I`):** `'02079460000         '`
* **Mobile Phone (`ENT1HP2I`):** `'07700900000         '`
* **Email Address (`ENT1HMOI`):** `'john.doe@example.com                                                                               '`
* **CICS Return State:** Returns to CICS with `TRANSID('SSC1')` and COMMAREA preserved.

#### 9.1.3 Logic Flow
1. The program receives the screen map input and enters the `SELECT (ENT1OPTO)` block on case `'1'`.
2. It sets `CA_REQUEST_ID = '01ICUS'` and assigns `CA_CUSTOMER_NUM` from `ENT1CNOO` (`'0000000001'`).
3. It links to `LGICUS01` with the 32,500-byte COMMAREA.
4. Upon return with `CA_RETURN_CODE = 0`, the returned customer details from `COMM_AREA` are populated into the output map fields (`ENT1FNAI`, `ENT1LNAI`, etc.).
5. The map `SSMAPC1` is sent back to the terminal, and control returns to CICS setting pseudo-conversational re-entry under `TRANSID('SSC1')`.

---

### 9.2 Example 2: Add New Customer (Option 2)

#### 9.2.1 Input Data
* **Map Option (`ENT1OPTO`):** `'2'`
* **First Name (`ENT1FNAI`):** `'ALICE     '`
* **Last Name (`ENT1LNAI`):** `'SMITH               '`
* **Date of Birth (`ENT1DOBI`):** `'1990-10-25'`
* **House Name (`ENT1HNMI`):** `'HILLSIDE HOUSE     '`
* **House Number (`ENT1HNOI`):** `'45        '`
* **Postcode (`ENT1HPCI`):** `'e1 6an  '`
* **Home Phone (`ENT1HP1I`):** `'02079460111         '`
* **Mobile Phone (`ENT1HP2I`):** `'07700900111         '`
* **Email Address (`ENT1HMOI`):** `'alice.smith@example.com                                                                            '`

#### 9.2.2 Expected Output
* **Customer Number Field (`ENT1CNOI`):** Generated customer number (e.g., `'0000000042'`)
* **Option Field (`ENT1OPTI`):** `' '` (cleared)
* **Message Field (`ERRFLDO`):** `'New Customer Inserted'`
* **Control Queue (`GENACNTL`):** Updated with high customer number `'0000000042'`.
* **CICS Return State:** Returns to CICS with `TRANSID('SSC1')` and COMMAREA preserved.

#### 9.2.3 Logic Flow
1. The user selects option `'2'`. The program initializes `CA_REQUEST_ID = '01ACUS'` and `CA_CUSTOMER_NUM = 0`.
2. Screen input values are mapped into COMMAREA fields, null bytes (`00'X`) are translated to spaces, and `CA_POSTCODE` is converted to uppercase (`'E1 6AN  '`).
3. The program links to `LGACUS01` to persist the new record.
4. Upon successful return (`CA_RETURN_CODE = 0`), the generated `CA_CUSTOMER_NUM` is updated in the temporary storage queue `GENACNTL` via `WRITE_GENACNTL`.
5. The map updates `ENT1CNOI` with the newly assigned number, clears the option field, displays `'New Customer Inserted'`, and re-issues `EXEC CICS RETURN TRANSID('SSC1')`.

---

### 9.3 Example 3: Invalid Option Selection

#### 9.3.1 Input Data
* **Map Option (`ENT1OPTO`):** `'9'`

#### 9.3.2 Expected Output
* **Message Field (`ERRFLDO`):** `'Please enter a valid option'`
* **Cursor Position:** Repositioned at the option input field (`ENT1OPTL = -1`).
* **CICS Return State:** Returns to CICS with `TRANSID('SSC1')`.

#### 9.3.3 Logic Flow
1. The map input is evaluated in `SELECT (ENT1OPTO)`.
2. Because `'9'` does not match `'1'`, `'2'`, or `'4'`, the `OTHERWISE` block is executed.
3. The program sets `ERRFLDO = 'Please enter a valid option'`, places the cursor on the option field (`ENT1OPTL = -1`), sends `SSMAPC1` with the `CURSOR` option, and returns to CICS with `TRANSID('SSC1')`.

---

Generated by IBM Bob Premium Package for Z
