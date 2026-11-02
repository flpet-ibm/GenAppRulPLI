## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
- [3. Outputs](#3-outputs)
  - [3.1 Terminal and Screen Display Outputs](#31-terminal-and-screen-display-outputs)
  - [3.2 Inter-Program Communication and Service Interface Outputs](#32-inter-program-communication-and-service-interface-outputs)
  - [3.3 Transactional and Execution Side Effects](#33-transactional-and-execution-side-effects)
- [4. Processing Logic](#4-processing-logic)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
- [7. Constraints](#7-constraints)
  - [7.1 Input Data Validation Constraints](#71-input-data-validation-constraints)
  - [7.2 Processing Flow and Operational Constraints](#72-processing-flow-and-operational-constraints)
  - [7.3 Transactional and Error Handling Constraints](#73-transactional-and-error-handling-constraints)
  - [7.4 Sequencing and Timing Constraints](#74-sequencing-and-timing-constraints)
- [8. Error Handling](#8-error-handling)
- [9. Examples](#9-examples)
  - [9.1 Example 1: Inquire Endowment Policy Details (Option '1')](#91-example-1-inquire-endowment-policy-details-option-1)
  - [9.2 Example 2: Add a New Endowment Policy (Option '2')](#92-example-2-add-a-new-endowment-policy-option-2)
  - [9.3 Example 3: Invalid Menu Option Selection](#93-example-3-invalid-menu-option-selection)

## 1. Purpose

`LGTESTP2` serves as the CICS 3270 screen presentation and user interface driver for endowment life insurance policy operations within the GenApp application. It is designed to provide an interactive menu interface that allows end users to inquire on, add, delete, and update endowment policy records. By capturing user selections and input parameters—such as customer numbers, policy numbers, coverage amounts, policy terms, and investment fund selections—from BMS map screens and coordinating with backend policy management services, the program facilitates the complete lifecycle management of endowment policies while returning transaction status and validation feedback to the terminal.

## 2. Inputs

- **Procedure parameters (CICS environment)**
  - `DFHEIPTR`: CICS-supplied pointer to the EXEC interface block. The program reads `EIBCALEN` from it to tell whether this is the first invocation or a re-entry.
  - `COMM_AREA_PTR`: pointer to the CICS COMMAREA passed in by the previous pseudo-conversational iteration of transaction SSP2. It addresses the `COMM_AREA` structure, which is 32,500 bytes.
    - On first entry (`EIBCALEN = 0`) the program ignores this pointer and points it at its own working storage area.
    - `CA_REQUEST_ID`: 6-character code showing which endowment operation is requested. It is blank on first entry.
    - `CA_CUSTOMER_NUM`: 10-digit customer number carried in the COMMAREA.
    - `CA_POLICY_NUM`: 10-digit policy number carried in the COMMAREA.

- **User inputs (3270 terminal screen, map `SSMAPP2` in mapset `SSMAP`)**
  - The program reads the Endowment Policy screen into the input map `SSMAPP2I` with `EXEC CICS RECEIVE MAP`. If no map data is available (MAPFAIL), the transaction ends.
  - Menu selection
    - `ENP2OPTO`: one-character option that selects the operation.
      - `1`: inquire.
      - `2`: add.
      - `3`: delete.
      - `4`: update.
      - Any other value produces the message "Please enter a valid option".
  - Key identifiers
    - `ENP2PNOO`: 10-character policy number. It is the key for inquire, delete and update. It is set to zeros when the screen is first displayed.
    - `ENP2CNOO`: 10-character customer number linked to the policy. It is set to zeros when the screen is first displayed. It is copied to the COMMAREA customer number for inquire, delete and the inquiry step of update.
    - `ENP2CNOI`: customer number typed on the screen. It is used for add requests and for the update step after the second screen receive.
  - Policy details entered on the screen (used for add, and for update after the second `RECEIVE MAP`)
    - `ENP2IDAI`: policy issue date, which becomes `CA_ISSUE_DATE`.
    - `ENP2EDAI`: policy expiry date, which becomes `CA_EXPIRY_DATE`.
    - `ENP2FNMI`: fund name, which becomes `CA_E_FUND_NAME`.
    - `ENP2TERI`: term in years, which becomes `CA_E_TERM`.
    - `ENP2SUMI`: sum assured, which becomes `CA_E_SUM_ASSURED`.
    - `ENP2LIFI`: life assured name, which becomes `CA_E_LIFE_ASSURED`.
    - `ENP2WPRI`: with-profits indicator, which becomes `CA_E_WITH_PROFITS`.
    - `ENP2EQUI`: equities indicator, which becomes `CA_E_EQUITIES`.
    - `ENP2MANI`: managed fund indicator, which becomes `CA_E_MANAGED_FUND`.
  - Terminal attention keys
    - PF3: ends the transaction.
    - CLEAR: clears the screen and returns to the menu.

- **External programs whose returned data the program depends on (CICS LINK with the COMMAREA)**
  - `LGIPOL01` (policy inquiry), used for options 1 and 4.
    - Returns `CA_RETURN_CODE`.
    - Returns the policy details `CA_ISSUE_DATE`, `CA_EXPIRY_DATE`, `CA_E_FUND_NAME`, `CA_E_TERM`, `CA_E_SUM_ASSURED`, `CA_E_LIFE_ASSURED`, `CA_E_WITH_PROFITS`, `CA_E_MANAGED_FUND` and `CA_E_EQUITIES`.
  - `LGAPOL01` (policy add), used for option 2.
    - Returns `CA_RETURN_CODE`, with 70 meaning the customer does not exist.
    - Returns the generated `CA_POLICY_NUM`.
    - Returns `CA_CUSTOMER_NUM`.
    - Returns `CA_E_FUND_NAME`.
  - `LGDPOL01` (policy delete), used for option 3.
    - Returns `CA_RETURN_CODE`.
  - `LGUPOL01` (policy update), used for option 4.
    - Returns `CA_RETURN_CODE`.
    - Returns `CA_CUSTOMER_NUM`.
    - Returns `CA_POLICY_NUM`.

- **Defaults set by the program that go into the COMMAREA before a call**
  - `CA_PAYMENT`: set to 0 for add and update.
  - `CA_BROKERID`: set to 0 for add and update.
  - `CA_BROKERSREF`: set to blank for add and update.

## 3. Outputs

### 3.1 Terminal and Screen Display Outputs

- BMS Screen Map Output (`SSMAPP2O` / `SSMAP`)
  - Transmits formatted endowment policy data, input echoes, and status messages to the 3270 user terminal.
  - `ENP2PNOO` / `ENP2PNOI`: Displays the unique 10-digit endowment policy number, either initialised to `'0000000000'` on first entry, preserved during inquiry/update, or assigned/returned from policy creation.
  - `ENP2CNOO` / `ENP2CNOI`: Displays the 10-digit customer number linked to the policy, echoing the input or displaying the value returned by backend transactions.
  - `ENP2IDAO` / `ENP2IDAI`: Displays the policy issue date retrieved via inquiry or cleared upon deletion.
  - `ENP2EDAO` / `ENP2EDAI`: Displays the policy maturity/expiry date retrieved via inquiry or cleared upon deletion.
  - `ENP2FNMO` / `ENP2FNMI`: Displays the investment fund name associated with the endowment policy.
  - `ENP2TERO` / `ENP2TERI`: Displays the term of the policy in years.
  - `ENP2SUMO` / `ENP2SUMI`: Displays the guaranteed sum assured amount.
  - `ENP2LIFO` / `ENP2LIFI`: Displays the name of the life assured.
  - `ENP2WPRO` / `ENP2WPRI`: Displays the with-profits indicator flag.
  - `ENP2EQUI` / `ENP2EQUO`: Displays the equities investment indicator flag.
  - `ENP2MANI` / `ENP2MANO`: Displays the managed fund indicator flag.
  - `ENP2OPTO` / `ENP2OPTI`: Cleared or highlighted to prompt for menu option input.
  - `ENP2OPTL`: Cursor positioning flag; set to `-1` when an invalid option is entered to place the screen cursor at the option field.
  - `ERP2FLDO`: Screen message field that presents user feedback and error descriptions:
    - `'New Life Policy Inserted'`: Successful policy addition.
    - `'Life Policy Deleted'`: Successful policy deletion.
    - `'Life Policy Updated'`: Successful policy update.
    - `'Please enter a valid option'`: Validation failure for unsupported menu options.
    - `'Customer does not exist'`: Add operation failed because the customer number was not found (return code 70).
    - `'Error Adding Life Policy'`: General failure during policy creation.
    - `'Error Updating Life Policy'`: General failure during policy modification.
    - `'Error Deleting Life Policy'`: General failure during policy deletion.
    - `'No data was returned.'`: Inquire operation failure or policy record not found.

- Terminal Plain Text Message
  - `MSGEND`: Transmits `'Transaction ended'` to the terminal when the user presses `PF3` or when a `MAPFAIL` condition occurs, signaling normal transaction termination.

### 3.2 Inter-Program Communication and Service Interface Outputs

- CICS Communication Area (`COMM_AREA`)
  - Parameters passed across program links to policy orchestrators (`LGIPOL01`, `LGAPOL01`, `LGDPOL01`, `LGUPOL01`) and preserved across pseudo-conversational returns under transaction ID `SSP2`.
  - `CA_REQUEST_ID`: Operation action code passed to target subprograms (`'01IEND'` for Inquire, `'01AEND'` for Add, `'01DEND'` for Delete, `'01UEND'` for Update).
  - `CA_CUSTOMER_NUM`: Passes the target customer number to backend programs and receives the confirmed customer number back.
  - `CA_POLICY_NUM`: Passes the policy identifier for inquiry, deletion, and update, and receives the newly generated policy number on add.
  - `CA_PAYMENT`: Initialised to `0` prior to calling policy add and update services.
  - `CA_BROKERID`: Initialised to `0` prior to calling policy add and update services.
  - `CA_BROKERSREF`: Initialised to spaces prior to calling policy add and update services.
  - `CA_ISSUE_DATE`: Passes the requested policy issue date for add and update requests.
  - `CA_EXPIRY_DATE`: Passes the requested policy maturity/expiry date for add and update requests.
  - `CA_E_FUND_NAME`: Passes the fund name for add and update requests.
  - `CA_E_TERM`: Passes the policy term length for add and update requests.
  - `CA_E_SUM_ASSURED`: Passes the guaranteed sum assured value for add and update requests.
  - `CA_E_LIFE_ASSURED`: Passes the life assured name for add and update requests.
  - `CA_E_WITH_PROFITS`: Passes the with-profits indicator for add and update requests.
  - `CA_E_MANAGED_FUND`: Passes the managed fund indicator for add and update requests.
  - `CA_E_EQUITIES`: Passes the equities investment indicator for add and update requests.

### 3.3 Transactional and Execution Side Effects

- Database and Resource State Controls
  - `EXEC CICS SYNCPOINT ROLLBACK`: Issued when policy creation (`LGAPOL01`) or deletion (`LGDPOL01`) returns a non-zero return code, backing out any partial uncommitted updates made during the unit of work.
- CICS Pseudo-Conversational State
  - `EXEC CICS RETURN TRANSID('SSP2') COMMAREA(...)`: Sets the next transaction identifier to `SSP2` with the updated communication area to maintain conversation continuity across user interactions.

## 4. Processing Logic

- **Initialization and first-time entry**
  - The program is the CICS main routine for the Endowment Policy menu, transaction SSP2. It is reentrant and works on the shared communication area through a pointer.
  - It first checks whether a communication area was passed in (`EIBCALEN = 0`).
    - If none was passed, this is the first entry of the conversation.
      - The communication area pointer is set to a local base storage area.
      - The output map `SSMAPP2O` is blanked.
      - `CA_REQUEST_ID` is blanked.
      - `CA_RETURN_CODE` and `CA_CUSTOMER_NUM` are set to zero.
      - The request-specific area is cleared with `CA_REQUEST_INITIALISER`.
      - The screen policy number `ENP2PNOO` and customer number `ENP2CNOO` are set to '0000000000'.
      - The menu map `SSMAPP2` in mapset `SSMAP` is sent with ERASE.
    - If a communication area was passed, the program skips the initial display and goes straight to the receive step.

- **Terminal input handling and AID/condition setup**
  - The label `A_GAIN` marks the start of the input-processing section.
  - `HANDLE AID` routes the CLEAR key to `CLEARIT` and PF3 to `ENDIT`.
  - `HANDLE CONDITION` routes a MAPFAIL to `ENDIT`. A MAPFAIL means the terminal sent no map data.
  - `RECEIVE MAP('SSMAPP2')` reads the user's entries into the input map `SSMAPP2I`.

- **Option dispatch**
  - A `SELECT` on the option field `ENP2OPTO` chooses one of five paths: '1' (inquire), '2' (add), '3' (delete), '4' (update), or any other value.
  - Each valid option sets `CA_REQUEST_ID` and fills the communication area. It then uses `EXEC CICS LINK` with a 32500-byte COMMAREA to call a back-end program.
  - After each LINK, the program tests `CA_RETURN_CODE`. A value above zero sends the flow to an error path.

- **Option 1: Inquire policy**
  - `CA_REQUEST_ID` is set to '01IEND'.
  - `CA_CUSTOMER_NUM` is taken from `ENP2CNOO` and `CA_POLICY_NUM` from `ENP2PNOO`.
  - The program links to `LGIPOL01`.
  - If the return code is above zero, it goes to `NO_DATA`.
  - Otherwise the returned data is moved to the screen fields:
    - `CA_ISSUE_DATE` goes to `ENP2IDAI`.
    - `CA_EXPIRY_DATE` goes to `ENP2EDAI`.
    - `CA_E_FUND_NAME` goes to `ENP2FNMI`.
    - `CA_E_TERM` goes to `ENP2TERI`.
    - `CA_E_SUM_ASSURED` goes to `ENP2SUMI`.
    - `CA_E_LIFE_ASSURED` goes to `ENP2LIFI`.
    - `CA_E_WITH_PROFITS` goes to `ENP2WPRI`.
    - `CA_E_MANAGED_FUND` goes to `ENP2MANI`.
    - `CA_E_EQUITIES` goes to `ENP2EQUI`.
  - The map is sent, and control goes to `ENDIT_STARTIT`.

- **Option 2: Add policy**
  - `CA_REQUEST_ID` is set to '01AEND'.
  - `CA_CUSTOMER_NUM` is taken from `ENP2CNOI`.
  - `CA_PAYMENT` and `CA_BROKERID` are reset to zero, and `CA_BROKERSREF` is blanked.
  - The screen values are copied into the communication area:
    - the issue and expiry dates;
    - the fund name;
    - the term;
    - the sum assured;
    - the life assured;
    - the with-profits, managed-fund and equities indicators.
  - The program links to `LGAPOL01`.
  - If the return code is above zero:
    - It issues `SYNCPOINT ROLLBACK` to undo any updates.
    - It goes to `NO_ADD`.
  - On success:
    - The customer number, the new policy number and the fund name are echoed back to the screen.
    - The option field is blanked.
    - The message "New Life Policy Inserted" is set.
    - The map is sent, and control goes to `ENDIT_STARTIT`.

- **Option 3: Delete policy**
  - `CA_REQUEST_ID` is set to '01DEND'.
  - The customer and policy numbers come from `ENP2CNOO` and `ENP2PNOO`.
  - The program links to `LGDPOL01`.
  - If the return code is above zero:
    - It issues `SYNCPOINT ROLLBACK`.
    - It goes to `NO_DELETE`.
  - On success:
    - All the policy detail fields on the screen are blanked.
    - The message "Life Policy Deleted" is set.
    - The map is sent, and control goes to `ENDIT_STARTIT`.

- **Option 4: Update policy (two-step conversation)**
  - Step one retrieves the current data.
    - `CA_REQUEST_ID` is set to '01IEND', and the customer and policy numbers come from the output fields.
    - The program links to `LGIPOL01`.
    - If the return code is above zero, it goes to `NO_DATA`.
    - Otherwise the policy details are moved to the screen and the map is sent.
  - Step two receives the user's edits and applies them.
    - A second `RECEIVE MAP` reads the changed values.
    - `CA_REQUEST_ID` is set to '01UEND'.
    - `CA_CUSTOMER_NUM` is taken from `ENP2CNOI`.
    - `CA_PAYMENT` and `CA_BROKERID` are reset to zero, and `CA_BROKERSREF` is blanked.
    - The screen values are copied into the communication area as in the add path.
    - The program links to `LGUPOL01`.
    - If the return code is above zero, it goes to `NO_UPD`. There is no explicit rollback on this path.
    - On success:
      - The customer and policy numbers are echoed back.
      - The option field is blanked.
      - The message "Life Policy Updated" is set.
      - The map is sent, and control goes to `ENDIT_STARTIT`.

- **Invalid option handling**
  - Any other option value sets the message "Please enter a valid option".
  - The cursor is positioned on the option field by setting `ENP2OPTL = -1`.
  - The map is sent with `CURSOR`, and control goes to `ENDIT_STARTIT`.

- **Error handling routines**
  - `NO_ADD` examines `CA_RETURN_CODE` with a `SELECT`.
    - Code 70 sets "Customer does not exist".
    - Any other code sets "Error Adding Life Policy".
    - Both go to `ERROR_OUT`.
  - `NO_UPD` sets "Error Updating Life Policy" and goes to `ERROR_OUT`.
  - `NO_DELETE` sets "Error Deleting Life Policy" and goes to `ERROR_OUT`.
  - `NO_DATA` sets "No data was returned." and goes to `ERROR_OUT`.
  - `ERROR_OUT` does the following:
    - It sends the map with the error message.
    - It blanks the input map and the raw communication area.
    - It resets `CA_REQUEST_ID`, `CA_RETURN_CODE`, `CA_CUSTOMER_NUM` and `CA_REQUEST_INITIALISER`.
    - It goes to `ENDIT_STARTIT`.

- **Conversation control and termination**
  - `ENDIT_STARTIT` is the pseudo-conversational exit used after every successful or failed request.
    - It issues `RETURN TRANSID('SSP2') COMMAREA(COMM_AREA)`.
    - This makes the next terminal input restart the same transaction with the communication area preserved.
  - `ENDIT` is reached by PF3 or MAPFAIL.
    - It sends the text "Transaction ended" with ERASE and FREEKB.
    - It then returns to CICS without a transaction ID, which ends the conversation.
  - `CLEARIT` is reached by the CLEAR key.
    - It blanks the input map.
    - It sends the map with `MAPONLY`, which shows the empty screen.
    - It returns with `TRANSID('SSP2')` and the communication area.
  - The plain `EXEC CICS RETURN` after the `SELECT` is never reached in normal flow, because every branch jumps away first.

- **Control flow structures**
  - There are no explicit loops. Repetition comes from the pseudo-conversational pattern: the program returns to CICS with the transaction ID and is restarted for each user interaction.
  - Branching uses `IF`, `SELECT/WHEN/OTHERWISE` and `GO TO` to the labels `NO_DATA`, `NO_ADD`, `NO_UPD`, `NO_DELETE`, `ERROR_OUT`, `ENDIT_STARTIT`, `ENDIT` and `CLEARIT`.

- **Interactions with external systems**
  - **Terminal and BMS:**
    - `SEND MAP`, `RECEIVE MAP` and `SEND TEXT` exchange data with the 3270 terminal through mapset `SSMAP`, map `SSMAPP2`.
  - **Called business programs:**
    - `LGIPOL01` inquires.
    - `LGAPOL01` adds.
    - `LGDPOL01` deletes.
    - `LGUPOL01` updates.
    - All four are called through `EXEC CICS LINK`.
    - These programs are presumably where any database access happens. This program contains no direct file or database I/O.
  - **Transaction management:**
    - `SYNCPOINT ROLLBACK` backs out a failed add or delete.
    - `RETURN TRANSID` and the communication area carry the conversation across interactions.

## 5. Paragraphs

- **LGTESTP2 (main procedure entry and declarations)**
  - Purpose: Main reentrant PL/I CICS program for the Endowment Policy menu (transaction SSP2). It drives inquire, add, delete and update of endowment policies.
  - Declares the static message `MSGEND` ("Transaction ended"). Declares the BMS map structures for the SSMAP mapset (input `...I` and output `...O` overlays for SSMAPC1, P1, P2, P3, P4 and P5). Declares the `COMM_AREA` structure, which is a union based on `COMM_AREA_PTR`. It holds the request ID, return code, customer number and policy request data, including the endowment fields. `BASE_COMM_AREA` provides the storage used on first entry.
  - External interaction: none in the declarations. The program works with the BMS mapset SSMAP and the COMMAREA.

- **First-time initialization (IF EIBCALEN = 0)**
  - Purpose: Prepares and displays the empty endowment menu screen when the transaction starts with no COMMAREA.
  - Points `COMM_AREA_PTR` at `BASE_COMM_AREA`. Blanks the output map `SSMAPP2O`. Resets `CA_REQUEST_ID`, `CA_RETURN_CODE`, `CA_CUSTOMER_NUM` and `CA_REQUEST_INITIALISER`. Sets `ENP2CNOO` and `ENP2PNOO` to zeros.
  - Control flow: runs only when `EIBCALEN = 0`. Execution then falls through to `A_GAIN`.
  - External interaction: CICS SEND MAP('SSMAPP2') MAPSET('SSMAP') with ERASE.

- **A_GAIN (receive and dispatch)**
  - Purpose: Main processing point. It receives the user's input and dispatches on the menu option.
  - Registers HANDLE AID for CLEAR (to `CLEARIT`) and PF3 (to `ENDIT`). Registers HANDLE CONDITION for MAPFAIL (to `ENDIT`). Receives map SSMAPP2 into `SSMAPP2I`. Then evaluates `SELECT (ENP2OPTO)`.
  - Control flow: the SELECT/WHEN structure has four options and an OTHERWISE branch, described below. After the SELECT, an unconditional `EXEC CICS RETURN` follows. In practice every branch transfers control before reaching it.
  - External interaction: CICS RECEIVE MAP.
  - **WHEN ('1') – Inquire**
    - Sets `CA_REQUEST_ID` to '01IEND'. Moves the customer and policy numbers from the screen to the COMMAREA.
    - Links to LGIPOL01 with COMMAREA (length 32500).
    - If `CA_RETURN_CODE > 0`, goes to `NO_DATA`.
    - Otherwise it copies the issue date, expiry date, fund name, term, sum assured, life assured, with-profits, managed-fund and equities values to the screen fields. It sends the map and goes to `ENDIT_STARTIT`.
    - External interaction: CICS LINK to LGIPOL01 and CICS SEND MAP.
  - **WHEN ('2') – Add**
    - Sets `CA_REQUEST_ID` to '01AEND'. Moves the customer number from the screen input. Zeroes payment and broker ID, and blanks broker reference. Loads the issue date, expiry date and the endowment fields from the screen.
    - Links to LGAPOL01.
    - If `CA_RETURN_CODE > 0`, issues SYNCPOINT ROLLBACK and goes to `NO_ADD`.
    - Otherwise it echoes the customer number, policy number and fund name back to the screen. It clears the option, sets "New Life Policy Inserted" and sends the map. Then it goes to `ENDIT_STARTIT`.
    - External interaction: CICS LINK to LGAPOL01, CICS SYNCPOINT ROLLBACK and CICS SEND MAP.
  - **WHEN ('3') – Delete**
    - Sets `CA_REQUEST_ID` to '01DEND'. Moves the customer and policy numbers from the screen.
    - Links to LGDPOL01.
    - If `CA_RETURN_CODE > 0`, issues SYNCPOINT ROLLBACK and goes to `NO_DELETE`.
    - Otherwise it blanks the detail screen fields, sets "Life Policy Deleted" and sends the map. Then it goes to `ENDIT_STARTIT`.
    - External interaction: CICS LINK to LGDPOL01, CICS SYNCPOINT ROLLBACK and CICS SEND MAP.
  - **WHEN ('4') – Update**
    - First performs an inquire using '01IEND' with a link to LGIPOL01. A non-zero return code goes to `NO_DATA`. Otherwise it displays the current policy data and sends the map.
    - Then it receives the user's modified screen. It sets `CA_REQUEST_ID` to '01UEND', loads the customer number and the changed fields, and zeroes payment and broker ID.
    - Links to LGUPOL01. If `CA_RETURN_CODE > 0`, goes to `NO_UPD`.
    - Otherwise it echoes the customer and policy numbers, clears the option, sets "Life Policy Updated" and sends the map. Then it goes to `ENDIT_STARTIT`.
    - External interaction: CICS LINK to LGIPOL01 and LGUPOL01, CICS SEND MAP and CICS RECEIVE MAP.
  - **OTHERWISE – Invalid option**
    - Sets the message "Please enter a valid option". Sets `ENP2OPTL` to -1 to position the cursor on the option field. Sends the map with CURSOR and goes to `ENDIT_STARTIT`.

- **ENDIT_STARTIT**
  - Purpose: Pseudo-conversational return point. It ends the current task and schedules transaction SSP2 for the next user input.
  - Issues CICS RETURN with TRANSID('SSP2') and COMMAREA(COMM_AREA).
  - External interaction: CICS RETURN.

- **ENDIT**
  - Purpose: Normal termination, reached by PF3 or by a MAPFAIL condition.
  - Sends the text "Transaction ended" (`MSGEND`) with ERASE and FREEKB. Then it issues a plain CICS RETURN, which ends the transaction without a TRANSID.
  - External interaction: CICS SEND TEXT and CICS RETURN.

- **CLEARIT**
  - Purpose: Handles the CLEAR key.
  - Blanks the input map `SSMAPP2I`. Sends the map with MAPONLY, which displays only the screen layout. Then it returns with TRANSID('SSP2') and the COMMAREA.
  - External interaction: CICS SEND MAP (MAPONLY) and CICS RETURN.

- **NO_ADD**
  - Purpose: Error handler for a failed add.
  - Control flow: SELECT on `CA_RETURN_CODE`. When the code is 70, the message is "Customer does not exist". Otherwise it is "Error Adding Life Policy". Both paths go to `ERROR_OUT`.

- **NO_UPD**
  - Purpose: Error handler for a failed update.
  - Sets "Error Updating Life Policy" and goes to `ERROR_OUT`.

- **NO_DELETE**
  - Purpose: Error handler for a failed delete.
  - Sets "Error Deleting Life Policy" and goes to `ERROR_OUT`.

- **NO_DATA**
  - Purpose: Error handler for an inquiry that returned no data. It is also used by the first step of update.
  - Sets "No data was returned." and goes to `ERROR_OUT`.

- **ERROR_OUT**
  - Purpose: Common error display and reset routine.
  - Sends map SSMAPP2 from `SSMAPP2O`, which carries the error message. Then it blanks the input map and the whole COMMAREA (`COMM_AREA_RAW`). It resets the request ID, return code, customer number and request initialiser. Then it goes to `ENDIT_STARTIT`.
  - External interaction: CICS SEND MAP, then control passes to the CICS RETURN with TRANSID in `ENDIT_STARTIT`.

## 6. Dependencies

- **CICS runtime environment**
  - CICS transaction SSP2: The program runs as this transaction and pseudo-conversationally re-invokes itself with `EXEC CICS RETURN TRANSID('SSP2')`, passing the COMMAREA back each time.
  - CICS PL/I translator and EXEC interface: The program receives the EIB pointer `DFHEIPTR` and the COMMAREA pointer `COMM_AREA_PTR` as parameters. It reads `EIBCALEN` to tell whether this is the first invocation (length 0) or a re-entry.
  - Reentrant main-procedure attributes: `Options(Main,Reentrant)` requires a reentrant-capable CICS PL/I runtime.
  - CICS terminal and 3270 session: The program sends and receives maps and text to and from the terminal that started the transaction.
  - CICS AID and condition handling: `HANDLE AID` routes CLEAR to the `CLEARIT` routine and PF3 to `ENDIT`. `HANDLE CONDITION MAPFAIL` routes to `ENDIT`.
  - CICS syncpoint services: `SYNCPOINT ROLLBACK` backs out recoverable work when the add or delete link returns a non-zero return code.

- **BMS mapset and screen definitions**
  - Mapset SSMAP: The compiled BMS mapset must be defined to CICS. It is the source of the screen layouts used by every `SEND MAP` and `RECEIVE MAP`.
  - Map SSMAPP2: This is the Endowment Policy menu screen. The program uses it for the initial display, for receiving user input, and for all result and error displays.
    - Input structure `SSMAPP2I` receives terminal data. It carries the option, customer number, policy number, issue and expiry dates, fund name, term, sum assured, life assured, and the with-profits, equities and managed-fund indicators.
    - Output structure `SSMAPP2O` is overlaid on the input structure and drives screen output. It includes `ENP2OPTO`, `ENP2PNOO`, `ENP2CNOO` and the message field `ERP2FLDO`.
  - Other maps in SSMAP (SSMAPC1, SSMAPP1, SSMAPP3, SSMAPP4, SSMAPP5): Their structures are declared through the include, but this program does not use them.
  - Standard 3270 terminal output: `SEND TEXT` with `ERASE` and `FREEKB` shows the "Transaction ended" message (`MSGEND`) when the user exits.

- **Included source members (copybooks)**
  - SSMAP include (`plinc/SSMAP.inc`): Supplies the symbolic map structures for the SSMAP mapset. These are the `…I` input and `…O` output layouts.
  - LGCMAREA include (`plinc/LGCMAREA.inc`): Supplies the shared COMMAREA layout, `COMM_AREA`. The layout is a union that covers customer, security, policy, endowment, house, motor, commercial and claim data.
    - Fields used by this program: `CA_REQUEST_ID`, `CA_RETURN_CODE`, `CA_CUSTOMER_NUM`, `CA_POLICY_NUM`, `CA_ISSUE_DATE`, `CA_EXPIRY_DATE`, `CA_PAYMENT`, `CA_BROKERID`, `CA_BROKERSREF`, and the `CA_E_*` endowment fields.
    - Fields used for initialisation and reset: `CA_REQUEST_INITIALISER` and `COMM_AREA_RAW`.

- **Called programs (CICS LINK targets)**
  - LGIPOL01: The policy inquiry program. It is called with `CA_REQUEST_ID = '01IEND'` for inquire (option 1) and for the pre-update read (option 4). It returns the endowment details, which the program shows on the screen.
  - LGAPOL01: The policy add program. It is called with `CA_REQUEST_ID = '01AEND'` (option 2) and returns the new policy number. A return code of 70 means the customer does not exist. Any other non-zero return code is reported as a generic add error.
  - LGDPOL01: The policy delete program. It is called with `CA_REQUEST_ID = '01DEND'` (option 3) and receives the customer number and policy number.
  - LGUPOL01: The policy update program. It is called with `CA_REQUEST_ID = '01UEND'` (option 4) after the user edits the inquired data.
  - Interface contract: All four programs are linked with `COMMAREA(COMM_AREA)` and `LENGTH(32500)`. They must set `CA_RETURN_CODE`, where a non-zero value signals failure.
  - Downstream resources: The called programs are expected to access the policy data store, which is probably Db2. The program does not reference that store directly.

- **Input parameters and data sources**
  - Terminal user input from map SSMAPP2: This is the main runtime input.
    - Menu option, `ENP2OPTO`: 1 = inquire, 2 = add, 3 = delete, 4 = update. Any other value gives an invalid-option message.
    - Customer number: `ENP2CNOO` or `ENP2CNOI`. It is moved to `CA_CUSTOMER_NUM`, 10 digits.
    - Policy number: `ENP2PNOO`. It is moved to `CA_POLICY_NUM`, 10 digits.
    - Policy details: issue date, expiry date, fund name, term, sum assured, life assured, with-profits flag, equities flag and managed-fund flag. These feed add and update requests.
  - Attention keys: CLEAR resets the screen and PF3 ends the transaction.
  - COMMAREA passed on re-entry:
    - `EIBCALEN = 0` triggers first-time initialisation and the initial map display.
    - Otherwise the COMMAREA from the previous pseudo-conversational step supplies the shared storage `COMM_AREA`.
  - Working storage for the first invocation: `BASE_COMM_AREA` is sized with `STG(COMM_AREA)`. It provides storage for the COMMAREA when none is passed in.

- **Configuration values and hard-coded identifiers**
  - Transaction ID 'SSP2': This must be defined in the CICS PCT or CSD and associated with LGTESTP2. It is also the TRANSID used for the pseudo-conversational return.
  - Mapset name 'SSMAP' and map name 'SSMAPP2': These must be installed and available in CICS.
  - Program names 'LGIPOL01', 'LGAPOL01', 'LGDPOL01' and 'LGUPOL01': These must be defined as programs in the CICS region.
  - Request ID codes '01IEND', '01AEND', '01DEND' and '01UEND': These are the contract values that tell the called programs which operation to run.
  - COMMAREA length constant 32500: It must match the length of `COMM_AREA`.
  - Return-code convention: 0 = success. Any value above 0 is an error, and 70 specifically means customer not found.
  - Message literals: 'Transaction ended', 'New Life Policy Inserted', 'Life Policy Deleted', 'Life Policy Updated', 'Error Adding Life Policy', 'Error Updating Life Policy', 'Error Deleting Life Policy', 'Customer does not exist', 'No data was returned.' and 'Please enter a valid option'.
  - No environment variables, external files, queues, or direct database or API access appear in this program.

## 7. Constraints

### 7.1 Input Data Validation Constraints

- Menu Option Selection
  - The menu option field `ENP2OPTO` must contain one of the single-character values `'1'` (Inquiry), `'2'` (Add), `'3'` (Delete), or `'4'` (Update).
  - Any value other than `'1'`, `'2'`, `'3'`, or `'4'` is rejected in the `SELECT (ENP2OPTO)` statement under `OTHERWISE`. The program sets the error message `'Please enter a valid option'` into `ERP2FLDO`, sets the cursor indicator `ENP2OPTL = -1` to position the cursor at the option field, and re-sends map `SSMAPP2`.
- Customer Number and Policy Number Formats
  - `ENP2CNOO` / `ENP2CNOI` and `ENP2PNOO` / `ENP2PNOI` are defined as 10-character alphanumeric fields mapping into 10-digit numeric fields (`CA_CUSTOMER_NUM` with picture `'9999999999'` and `CA_POLICY_NUM` with picture `'9999999999'`).
  - On initial entry (`EIBCALEN = 0`), customer and policy number output fields are strictly initialized to zero strings (`'0000000000'`).
- Endowment Detail Field Formats and Lengths
  - Issue Date (`ENP2IDAI`) and Expiry Date (`ENP2EDAI`) are constrained to 10-character formats mapped to `CA_ISSUE_DATE` and `CA_EXPIRY_DATE`.
  - Fund Name (`ENP2FNMI`) is constrained to a 10-character string mapped to `CA_E_FUND_NAME`.
  - Policy Term (`ENP2TERI`) is constrained to a 2-character field mapped to `CA_E_TERM` (picture `'99'`).
  - Sum Assured (`ENP2SUMI`) is constrained to a 6-character field mapped to `CA_E_SUM_ASSURED` (picture `'999999'`).
  - Life Assured (`ENP2LIFI`) is defined as a 25-character screen field mapped into the 31-character communication area field `CA_E_LIFE_ASSURED`.
  - Indicators for With-Profits (`ENP2WPRI`), Managed Fund (`ENP2MANI`), and Equities (`ENP2EQUI`) are restricted to single-character values (`CHAR(1)`).

### 7.2 Processing Flow and Operational Constraints

- First-Time Entry Initialization
  - When invoked with `EIBCALEN = 0`, the program suppresses transaction processing, establishes pointer addressing for `COMM_AREA_PTR` via `BASE_COMM_AREA`, clears map output buffer `SSMAPP2O`, initializes commarea fields (`CA_REQUEST_ID = ' '`, `CA_RETURN_CODE = 0`, `CA_CUSTOMER_NUM = 0`), sets `ENP2CNOO` and `ENP2PNOO` to `'0000000000'`, sends map `SSMAPP2` with `ERASE`, and exits without linking to backend services.
- Conversational / Pseudo-Conversational Flow
  - Normal transaction termination returns control to CICS via `EXEC CICS RETURN TRANSID('SSP2') COMMAREA(COMM_AREA)`, enforcing pseudo-conversational state preservation under transaction ID `SSP2`.
  - Session exit via PF3 or MAPFAIL sends the termination text `'Transaction ended'` and executes `EXEC CICS RETURN` without `TRANSID`, ending the transaction flow.
  - Session reset via CLEAR key (`CLEARIT`) clears `SSMAPP2I`, sends the map with `MAPONLY`, and re-invokes `SSP2` with the commarea.
- Hardcoded Request Identifiers
  - Specific operation codes must be populated into `CA_REQUEST_ID` before invoking orchestrators:
    - `'01IEND'` for Inquiry (Option 1) and pre-update fetch (Option 4).
    - `'01AEND'` for Add (Option 2).
    - `'01DEND'` for Delete (Option 3).
    - `'01UEND'` for Update (Option 4).
- Forced Defaults for Add and Update Operations
  - During Add (`WHEN ('2')`) and Update (`WHEN ('4')`) operations, `CA_PAYMENT` is enforced to `0`, `CA_BROKERID` is enforced to `0`, and `CA_BROKERSREF` is enforced to spaces (`' '`) prior to calling the backend services.

### 7.3 Transactional and Error Handling Constraints

- Synchronous Orchestrator Invocations
  - External program calls must pass an exact commarea length of 32,500 bytes:
    - Option 1 invokes `LGIPOL01` with `LENGTH(32500)`.
    - Option 2 invokes `LGAPOL01` with `LENGTH(32500)`.
    - Option 3 invokes `LGDPOL01` with `LENGTH(32500)`.
    - Option 4 invokes `LGIPOL01` followed by `LGUPOL01`, both with `LENGTH(32500)`.
- Transaction Rollback on Failure
  - For Add (`WHEN ('2')`) and Delete (`WHEN ('3')`), if `CA_RETURN_CODE > 0`, the program issues an explicit `EXEC CICS SYNCPOINT ROLLBACK` before branching to error routines, ensuring transactional atomicity and preventing partial database updates.
- Return Code Handling and Message Mapping
  - For Inquiry (`WHEN ('1')` or Option 4 fetch), any `CA_RETURN_CODE > 0` branches to `NO_DATA`, setting message `'No data was returned.'`.
  - For Add (`WHEN ('2')`), a return code of `70` sets `ERP2FLDO = 'Customer does not exist'`, while any other non-zero code sets `ERP2FLDO = 'Error Adding Life Policy'`.
  - For Delete (`WHEN ('3')`), any non-zero return code sets `ERP2FLDO = 'Error Deleting Life Policy'`.
  - For Update (`WHEN ('4')`), any non-zero return code during the update call sets `ERP2FLDO = 'Error Updating Life Policy'`.
  - When an error occurs (`ERROR_OUT`), the program re-sends the map, reinitializes input buffer `SSMAPP2I`, clears `COMM_AREA_RAW`, resets `CA_RETURN_CODE` and `CA_CUSTOMER_NUM` to `0`, and re-invokes `SSP2`.

### 7.4 Sequencing and Timing Constraints

- Attention Identifier (AID) and Condition Interception
  - `EXEC CICS HANDLE AID` and `EXEC CICS HANDLE CONDITION` must be executed prior to `EXEC CICS RECEIVE MAP` to capture `CLEAR`, `PF3`, and `MAPFAIL`.
- Multi-Step Update Sequence
  - Option 4 (Update) strictly requires a two-phase flow:
    - Phase 1: An inquiry call (`01IEND`) via `LGIPOL01` retrieves the existing policy data and displays it on the screen.
    - Phase 2: A synchronous `RECEIVE MAP` captures updated values from the terminal before linking to `LGUPOL01` with request code `01UEND`.
- Screen Reset Following Mutation
  - Upon successful Add (`WHEN ('2')`), fields `ENP2CNOI`, `ENP2PNOI`, and `ENP2FNMI` are echoed back from the commarea, the option field `ENP2OPTI` is blanked out, and `ERP2FLDO` is set to `'New Life Policy Inserted'`.
  - Upon successful Delete (`WHEN ('3')`), all endowment policy detail screen fields (`ENP2IDAI`, `ENP2EDAI`, `ENP2FNMI`, `ENP2TERI`, `ENP2SUMI`, `ENP2LIFI`, `ENP2WPRI`, `ENP2MANI`, `ENP2EQUI`) are cleared to spaces, and `ERP2FLDO` is set to `'Life Policy Deleted'`.
  - Upon successful Update (`WHEN ('4')`), `ENP2CNOI` and `ENP2PNOI` are updated from the commarea, `ENP2OPTI` is blanked out, and `ERP2FLDO` is set to `'Life Policy Updated'`.

## 8. Error Handling

-   **Terminal key and screen-input handling (CICS AID and condition handling)**
    -   A CICS HANDLE AID routes the CLEAR key to a clear-screen routine and PF3 to the program's termination routine.
        -   The CLEAR routine blanks the input map structure and resends the map with the map data only. It then returns to transaction SSP2 and passes the communication area back, so the session continues with a clean screen.
        -   The PF3 routine sends the "Transaction ended" text to the terminal. It erases the screen, frees the keyboard, and returns without a transaction ID, which ends the conversation.
    -   A CICS HANDLE CONDITION routes a MAPFAIL on the map receive to the same termination routine. This covers the case where the user sends no data. The program ends cleanly instead of abending.
    -   No RESP or RESP2 checks follow any CICS command. Any CICS condition other than MAPFAIL, such as a failure on a LINK or SEND, is not handled in the program. It falls to the CICS default action, which is normally an abend.

-   **Input validation**
    -   The option field (ENP2OPTO) is evaluated in a SELECT. Only '1' (inquire), '2' (add), '3' (delete) and '4' (update) are accepted.
    -   Any other value takes the OTHERWISE branch.
        -   The program puts "Please enter a valid option" in the screen error field (ERP2FLDO).
        -   It sets the option field length to -1, so the cursor is placed on the option field.
        -   It resends the map with the CURSOR option.
        -   It then returns to the same transaction so the user can retry.
    -   No other field is validated here. This includes policy number, customer number, dates and amounts. Those values are passed to the called programs, which are expected to validate them.

-   **Return-code checking after called programs**
    -   After each CICS LINK to a back-end policy program, the program tests the return code in the communication area (CA_RETURN_CODE). A value greater than zero means failure.
    -   Inquire (option 1), through LGIPOL01: a non-zero return code branches to the "no data" routine.
    -   Add (option 2), through LGAPOL01: a non-zero return code first triggers a CICS SYNCPOINT ROLLBACK and then branches to the add-error routine.
    -   Delete (option 3), through LGDPOL01: a non-zero return code first triggers a SYNCPOINT ROLLBACK and then branches to the delete-error routine.
    -   Update (option 4) works in two steps.
        -   The pre-update inquiry through LGIPOL01 branches to the "no data" routine on a non-zero return code.
        -   The update itself, through LGUPOL01, branches to the update-error routine on a non-zero return code. This path has no SYNCPOINT ROLLBACK in the calling program.

-   **Error message selection and notification to the user**
    -   Failures are shown to the user through the error field on the policy menu screen.
        -   "No data was returned." is shown for a failed inquiry, including the inquiry that precedes an update.
        -   "Error Adding Life Policy" is the default message for a failed add.
        -   "Customer does not exist" is a specific message for a failed add. It is chosen when the return code is exactly 70.
        -   "Error Updating Life Policy" is shown for a failed update.
        -   "Error Deleting Life Policy" is shown for a failed delete.
    -   Only the add path distinguishes between return codes. A SELECT on the return code separates code 70 from all other codes. The inquire, update and delete paths use a single generic message each.
    -   Success is also reported through the same field, which doubles as a status line. The messages are "New Life Policy Inserted", "Life Policy Updated" and "Life Policy Deleted".

-   **Recovery and fallback logic**
    -   Transactional recovery: on add and delete failures, SYNCPOINT ROLLBACK undoes any database changes made by the linked program before the error screen is shown. This keeps the data consistent.
    -   Common error exit: all error routines go to one shared routine. It does the following.
        -   It sends the map, including the error message, to the terminal.
        -   It resets the working state by blanking the input map, the whole communication area and the request ID.
        -   It zeroes the return code and customer number, and clears the request-specific area.
        -   It returns to transaction SSP2 with the clean communication area. The user can retry from a known state.
    -   First-time initialization: when the communication area length is zero (EIBCALEN = 0), the program points to a locally declared base communication area. It then initializes the following before sending the initial menu.
        -   The output map is blanked.
        -   The request ID is set to blanks and the return code to zero.
        -   The customer number is set to zero.
        -   The request-specific area is blanked.
        -   The customer and policy number screen fields are set to zeros.
    -   Pseudo-conversational restart: after every handled outcome, the program returns with the SSP2 transaction ID and the communication area. The menu is redisplayed and the error state does not persist to the next interaction.

-   **Program termination paths**
    -   Normal end after PF3 or MAPFAIL: the "Transaction ended" message is sent and the program returns.
    -   Defensive plain return: a CICS RETURN follows the end of the SELECT block. Normal flow does not reach it, because each branch ends with a jump to the restart routine.

-   **Notable gaps**
    -   There are no PL/I ON-unit conditions, such as ERROR, CONVERSION, SIZE or ENDFILE. There is no SIGNAL handling and no explicit ABEND call.
    -   Return codes are only tested for being greater than zero. Apart from code 70 on add, the program does not interpret them, and it does not log them.
    -   Because the program has no logging, error details are visible only to the terminal user.

## 9. Examples

### 9.1 Example 1: Inquire Endowment Policy Details (Option '1')

Demonstrates looking up and displaying existing endowment policy details for a given customer and policy number.

#### 9.1.1 Sample Input Data
* **Screen Option (`ENP2OPTO`):** `'1'`
* **Screen Policy Number (`ENP2PNOO`):** `'0000000042'`
* **Screen Customer Number (`ENP2CNOO`):** `'0000000101'`

#### 9.1.2 Expected Output
* **Issue Date (`ENP2IDAI`):** `'2020-01-15'`
* **Expiry Date (`ENP2EDAI`):** `'2040-01-15'`
* **Fund Name (`ENP2FNMI`):** `'GROWTHFUND'`
* **Term (`ENP2TERI`):** `'20'`
* **Sum Assured (`ENP2SUMI`):** `'100000'`
* **Life Assured (`ENP2LIFI`):** `'Jane Doe'`
* **With Profits Indicator (`ENP2WPRI`):** `'Y'`
* **Managed Fund Indicator (`ENP2MANI`):** `'N'`
* **Equities Indicator (`ENP2EQUI`):** `'Y'`
* **Message / Error Field (`ERP2FLDO`):** `''` (blank)

#### 9.1.3 Logic Flow
1. The program receives the map `SSMAPP2` and detects option `'1'`.
2. It sets `CA_REQUEST_ID` to `'01IEND'`, moves `ENP2CNOO` to `CA_CUSTOMER_NUM`, and moves `ENP2PNOO` to `CA_POLICY_NUM`.
3. It issues `EXEC CICS LINK PROGRAM('LGIPOL01')` passing `COMM_AREA`.
4. Upon successful return (`CA_RETURN_CODE = 0`), it populates the screen fields (`ENP2IDAI`, `ENP2EDAI`, `ENP2FNMI`, `ENP2TERI`, `ENP2SUMI`, `ENP2LIFI`, `ENP2WPRI`, `ENP2MANI`, `ENP2EQUI`) from the returned commarea fields and sends the populated map back to the terminal.

---

### 9.2 Example 2: Add a New Endowment Policy (Option '2')

Demonstrates inserting a new endowment policy for an existing customer.

#### 9.2.1 Sample Input Data
* **Screen Option (`ENP2OPTI`):** `'2'`
* **Customer Number (`ENP2CNOI`):** `'0000000101'`
* **Issue Date (`ENP2IDAI`):** `'2023-06-01'`
* **Expiry Date (`ENP2EDAI`):** `'2043-06-01'`
* **Fund Name (`ENP2FNMI`):** `'BALANCED01'`
* **Term (`ENP2TERI`):** `'20'`
* **Sum Assured (`ENP2SUMI`):** `'050000'`
* **Life Assured (`ENP2LIFI`):** `'John Smith'`
* **With Profits Indicator (`ENP2WPRI`):** `'Y'`
* **Managed Fund Indicator (`ENP2MANI`):** `'Y'`
* **Equities Indicator (`ENP2EQUI`):** `'N'`

#### 9.2.2 Expected Output
* **Assigned Policy Number (`ENP2PNOI`):** `'0000000105'`
* **Customer Number (`ENP2CNOI`):** `'0000000101'`
* **Fund Name (`ENP2FNMI`):** `'BALANCED01'`
* **Screen Option (`ENP2OPTI`):** `' '` (cleared)
* **Message Field (`ERP2FLDO`):** `'New Life Policy Inserted'`

#### 9.2.3 Logic Flow
1. The program evaluates option `'2'` and initializes `CA_REQUEST_ID` to `'01AEND'`, `CA_PAYMENT` to `0`, and `CA_BROKERID` to `0`.
2. It transfers the customer number, dates, term, sum assured, life assured name, and fund indicators from the map into the commarea endowment structure.
3. It links to `LGAPOL01`.
4. Upon receiving `CA_RETURN_CODE = 0`, it retrieves the generated `CA_POLICY_NUM`, sets the success message `'New Life Policy Inserted'`, clears the option field, and re-displays the map.

---

### 9.3 Example 3: Invalid Menu Option Selection

Demonstrates error handling when an unsupported option code is supplied.

#### 9.3.1 Sample Input Data
* **Screen Option (`ENP2OPTO`):** `'9'`
* **Policy Number (`ENP2PNOO`):** `'0000000042'`
* **Customer Number (`ENP2CNOO`):** `'0000000101'`

#### 9.3.2 Expected Output
* **Message Field (`ERP2FLDO`):** `'Please enter a valid option'`
* **Cursor Position:** Repositioned at the option field (`ENP2OPTL = -1`)

#### 9.3.3 Logic Flow
1. The program tests `ENP2OPTO` against valid choices (`'1'`, `'2'`, `'3'`, `'4'`).
2. Option `'9'` triggers the `OTHERWISE` branch.
3. It sets the error message `'Please enter a valid option'`, flags the option field length as `-1` to reposition the screen cursor, and issues `EXEC CICS SEND MAP ('SSMAPP2')` with the `CURSOR` option.

---

Generated by IBM Bob Premium Package for Z
