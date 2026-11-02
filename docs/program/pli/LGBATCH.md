## Table of Contents

- [1. Purpose](#1-purpose)
- [2. Inputs](#2-inputs)
  - [2.1 External Data Sources](#21-external-data-sources)
- [3. Outputs](#3-outputs)
- [4. Processing Logic](#4-processing-logic)
  - [4.1 Program Initialization and Resource Setup](#41-program-initialization-and-resource-setup)
  - [4.2 Sequential Batch File Processing Loop](#42-sequential-batch-file-processing-loop)
  - [4.3 Customer Record Processing (`PROCESS_RECORD`)](#43-customer-record-processing-process_record)
  - [4.4 Database Interaction and Status Handling (`GET_CUSTOMER_INFO`)](#44-database-interaction-and-status-handling-get_customer_info)
  - [4.5 Error Formatting (`WRITE_ERROR_MESSAGE`)](#45-error-formatting-write_error_message)
  - [4.6 Termination and Clean-up](#46-termination-and-clean-up)
- [5. Paragraphs](#5-paragraphs)
- [6. Dependencies](#6-dependencies)
- [7. Constraints](#7-constraints)
- [8. Error Handling](#8-error-handling)
- [9. Examples](#9-examples)

## 1. Purpose

LGBATCH is a batch (non-CICS) customer reporting program. It reads a sequential input file of customer numbers. For each number, it looks up the customer's personal and contact details in the DB2 CUSTOMER table. These details are first name, last name, date of birth, address, phone numbers and email. The program records the outcome of each lookup as a return code: success, customer not found, or database error. It passes the customer number, the shared communication area and a report line buffer to the LGBATCHF subroutine, which formats a report line. The program writes each line to the customer report output file. The result is a customer detail report covering every customer listed in the input file. A customer that isn't found is still reported, using whatever data remains in the communication area from the previous record. The program also builds a formatted error message for database failures, but it doesn't write that message anywhere. If the input file can't be opened, the program ends with a return code of 16.

## 2. Inputs

### 2.1 External Data Sources

- Sequential Files
  - `CUSTIN`
    - Input sequential dataset with fixed-block 10-byte records (`FB RECSIZE(10)`).
    - Contains individual customer records read sequentially into `CUSTIN_RECORD`.
    - `CUSTIN_CUSTNUMBER`: A 10-digit numeric field (`PIC '9999999999'`) holding the customer identifier used to drive database lookups.
- Relational Database (Db2)
  - `CUSTOMER` Table
    - Relational table queried via embedded SQL using the predicate `WHERE CUSTOMERNUMBER = :DB2_CUSTOMERNUMBER_INT`.
    - Provides customer profile data populated directly into the communication area structure (`CA_CUSTOMER_REQUEST`):
      - `FIRSTNAME` (`CA_FIRST_NAME`): Legal first name of the customer.
      - `LASTNAME` (`CA_LAST_NAME`): Legal surname or family name of the customer.
      - `DATEOFBIRTH` (`CA_DOB`): Customer date of birth.
      - `HOUSENAME` (`CA_HOUSE_NAME`): Residence or building name.
      - `HOUSENUMBER` (`CA_HOUSE_NUM`): Street or civic house number.
      - `POSTCODE` (`CA_POSTCODE`): Residential postal code.
      - `PHONEMOBILE` (`CA_PHONE_MOBILE`): Mobile contact telephone number.
      - `PHONEHOME` (`CA_PHONE_HOME`): Landline contact telephone number.
      - `EMAILADDRESS` (`CA_EMAIL_ADDRESS`): Customer email address.

## 3. Outputs

- **Customer report file (CUSTRPT)**
  - `CUSTRPT_LINE`
    - It is an 80-character record. `LGBATCHF` builds it, and the program writes it to CUSTRPT once for each customer number read from CUSTIN.
    - It is the program's only persisted output, and it is a formatted customer report line.
    - It is written even when the customer is not found or the SQL call fails. In those cases it is built from whatever the commarea still holds from the previous row.

- **Customer data passed to the report formatter (LGBATCHF)**
  - These fields are filled by the DB2 SELECT on the CUSTOMER table. The program passes the commarea to `LGBATCHF`, which uses them to build `CUSTRPT_LINE`.
  - Identification and personal details
    - `CA_FIRST_NAME`: customer first name.
    - `CA_LAST_NAME`: customer surname.
    - `CA_DOB`: date of birth.
  - Address details
    - `CA_HOUSE_NAME`: house or building name.
    - `CA_HOUSE_NUM`: house number.
    - `CA_POSTCODE`: postal code.
  - Contact details
    - `CA_PHONE_MOBILE`: mobile phone number.
    - `CA_PHONE_HOME`: home phone number.
    - `CA_EMAIL_ADDRESS`: email address.
  - Customer key and policy information
    - `CUSTIN_CUSTNUMBER`: customer number from the input record. It is passed to `LGBATCHF` as the customer being reported.
    - `CA_CUSTOMER_NUM`: customer number in the commarea header. This program never sets it, so it only has a value if `LGBATCHF` sets it.
    - `CA_NUM_POLICIES`: number of policies for the customer. This program never sets it, so it only has a value if `LGBATCHF` sets it.
  - Query key
    - `DB2_CUSTOMERNUMBER_INT`: integer host variable set from `CUSTIN_CUSTNUMBER`. It is the `CUSTOMERNUMBER` predicate in the SELECT, so it decides which customer's data is retrieved and reported.

- **Status and outcome indicator**
  - `CA_RETURN_CODE`
    - The program sets it after each SELECT: '00' when SQLCODE is 0, '01' when SQLCODE is 100 (customer not found), and '90' for any other SQLCODE.
    - It is passed to `LGBATCHF` in the commarea, so it can affect how the report line is formatted for found, not-found and error cases.
    - It is not written anywhere directly by this program.

- **Console messages**
  - The `DISPLAY` text 'FILE STATUS AT OPEN CUSTIN' appears when the UNDEFINEDFILE condition occurs on CUSTIN. It signals that the input file could not be opened.

- **Program return status**
  - The program calls `PLIRETC(16)` and then `STOP` when CUSTIN cannot be opened. This sets the job return code to 16 and ends the run without producing a report.
  - On normal completion the program closes both files and returns without setting a non-zero return code.

- **Error message structure (not externalized)**
  - `ERROR_MSG` is built when the SQL call returns an unexpected SQLCODE. It holds the date, time, program name, customer number, SQL request text and SQLCODE.
  - It is never displayed or written to a file, so it has no observable effect. Only `CA_RETURN_CODE` = '90' shows that an error occurred.

## 4. Processing Logic

### 4.1 Program Initialization and Resource Setup
- Addressability for Communication Area:
  - Sets `COMM_AREA_PTR` to reference the local working storage area `BATCH_COMM_AREA`, enabling access to the overlaid `COMM_AREA` and `CA_CUSTOMER_REQUEST` data structures.
- Subroutine Dynamic Loading:
  - Executes `FETCH LGBATCHF` to dynamically load the customer report formatting routine `LGBATCHF` into memory.
- File Exception Handling and File Opening:
  - Establishes an `ON UNDEFINEDFILE(CUSTIN)` exception unit that displays the diagnostic message `'FILE STATUS AT OPEN CUSTIN'`, invokes `PLIRETC(16)` to set the return code to 16, and executes `STOP` if the input dataset fails to open.
  - Opens the input customer sequential file `CUSTIN` (`RECORD INPUT SEQUENTIAL`, fixed blocked, 10 bytes).
  - Opens the output customer report sequential file `CUSTRPT` (`RECORD OUTPUT SEQUENTIAL`, fixed blocked, 80 bytes).
- End-of-File Condition Setup:
  - Establishes an `ON ENDFILE(CUSTIN)` exception unit that sets the boolean flag `FI_EOF` to `'1'B` upon reaching the end of the input file.

### 4.2 Sequential Batch File Processing Loop
- Initial Priming Read:
  - Reads the first 10-byte record from `CUSTIN` into `CUSTIN_RECORD`, extracting the 10-digit customer number `CUSTIN_CUSTNUMBER`.
- Iterative Record Processing:
  - Executes a `DO WHILE (^FI_EOF)` loop that continues until the end-of-file condition is raised on `CUSTIN`.
  - In each iteration:
    - Calls internal procedure `PROCESS_RECORD` to query the database, format output, and write the report line.
    - Executes `READ FILE(CUSTIN) INTO(CUSTIN_RECORD)` to retrieve subsequent customer records.

### 4.3 Customer Record Processing (`PROCESS_RECORD`)
- Data Conversion:
  - Converts the character/numeric display input field `CUSTIN_CUSTNUMBER` into binary integer format and assigns it to the Db2 host variable `DB2_CUSTOMERNUMBER_INT` (`FIXED BIN(31)`).
- Customer Data Retrieval:
  - Calls internal procedure `GET_CUSTOMER_INFO` to query customer information from Db2.
- Report Line Formatting and Output:
  - Invokes the external subroutine `LGBATCHF`, passing `CUSTIN_CUSTNUMBER`, `COMM_AREA_PTR`, and `CUSTRPT_LINE` to format the report text into an 80-character buffer.
  - Executes `WRITE FILE(CUSTRPT) FROM(CUSTRPT_LINE)` to write the formatted customer line out to the sequential dataset.

### 4.4 Database Interaction and Status Handling (`GET_CUSTOMER_INFO`)
- SQL Query Execution:
  - Sets `EM_SQLREQ` to `' SELECT CUSTOMER'`.
  - Executes an embedded Db2 `SELECT` statement querying `FIRSTNAME`, `LASTNAME`, `DATEOFBIRTH`, `HOUSENAME`, `HOUSENUMBER`, `POSTCODE`, `PHONEMOBILE`, `PHONEHOME`, and `EMAILADDRESS` from table `CUSTOMER` where `CUSTOMERNUMBER = :DB2_CUSTOMERNUMBER_INT`.
  - Populates the corresponding communication area host variables:
    - `CA_FIRST_NAME`
    - `CA_LAST_NAME`
    - `CA_DOB`
    - `CA_HOUSE_NAME`
    - `CA_HOUSE_NUM`
    - `CA_POSTCODE`
    - `CA_PHONE_MOBILE`
    - `CA_PHONE_HOME`
    - `CA_EMAIL_ADDRESS`
- SQL Condition Evaluation (`SELECT (SQLCODE)`):
  - `WHEN (0)`:
    - Sets `CA_RETURN_CODE` to `'00'` indicating successful data retrieval.
  - `WHEN (100)`:
    - Sets `CA_RETURN_CODE` to `'01'` indicating customer record not found. Note that `LGBATCHF` is still called, formatting whatever remains in `COMM_AREA`.
  - `OTHERWISE`:
    - Sets `CA_RETURN_CODE` to `'90'` indicating a database error.
    - Calls internal procedure `WRITE_ERROR_MESSAGE` to log SQL error diagnostics.

### 4.5 Error Formatting (`WRITE_ERROR_MESSAGE`)
- SQL Error Information Formatting:
  - Formats the diagnostic structure `ERROR_MSG` by capturing execution context:
    - Assigns `SQLCODE` to numeric display field `EM_SQLRC`.
    - Assigns date `WS_DATE` to `EM_DATE`.
    - Assigns time `WS_TIME` to `EM_TIME`.

### 4.6 Termination and Clean-up
- File Closing and Return:
  - Closes sequential files `CUSTIN` and `CUSTRPT`.
  - Executes `RETURN` to terminate main routine execution and return control to the operating system.

## 5. Paragraphs

- **LGBATCH (Main Procedure)**
  - **Purpose**: Acts as the main entry point for the batch program, driving the overall process of reading customer input records, coordinating customer data retrieval and formatting, and writing output report records.
  - **Operations Performed**:
    - Initializes the communication area pointer (`COMM_AREA_PTR`) to reference `BATCH_COMM_AREA`.
    - Dynamically loads (`FETCH`) the external subroutine `LGBATCHF`.
    - Establishes condition handlers for file exceptions (`UNDEFINEDFILE` and `ENDFILE` for `CUSTIN`).
    - Opens input file `CUSTIN` and output file `CUSTRPT`.
    - Reads customer records sequentially from `CUSTIN` and invokes `PROCESS_RECORD` for each record.
    - Closes both `CUSTIN` and `CUSTRPT` files upon reaching end-of-file and terminates execution.
  - **Control Flow Elements**:
    - `ON UNDEFINEDFILE(CUSTIN)` block to handle file opening errors by displaying a message, setting a return code of 16 via `PLIRETC`, and halting with `STOP`.
    - `ON ENDFILE(CUSTIN)` condition handler setting the end-of-file flag `FI_EOF` to `'1'B`.
    - `DO WHILE (^FI_EOF)` loop that iterates through the input file records until the end of file is reached.
  - **Interactions with External Systems**:
    - Sequential file input operations (`OPEN`, `READ`, `CLOSE`) on `CUSTIN`.
    - Sequential file output operations (`OPEN`, `CLOSE`) on `CUSTRPT`.
    - Dynamic program management via `FETCH` to load `LGBATCHF`.

- **PROCESS_RECORD**
  - **Purpose**: Processes a single customer record read from the input file by querying customer details, formatting the report line via an external subroutine, and writing the result to the report file.
  - **Operations Performed**:
    - Assigns the input customer number `CUSTIN_CUSTNUMBER` to the DB2 binary integer host variable `DB2_CUSTOMERNUMBER_INT`.
    - Calls `GET_CUSTOMER_INFO` to retrieve customer details from the database.
    - Calls the external subroutine `LGBATCHF`, passing the customer number, communication area pointer, and report output line buffer `CUSTRPT_LINE`.
    - Writes the formatted line buffer `CUSTRPT_LINE` to the `CUSTRPT` output file.
  - **Control Flow Elements**:
    - Linear sequential execution calling internal procedure `GET_CUSTOMER_INFO` followed by external routine `LGBATCHF`.
  - **Interactions with External Systems**:
    - Calls external routine `LGBATCHF`.
    - Performs sequential file write (`WRITE`) to `CUSTRPT`.

- **GET_CUSTOMER_INFO**
  - **Purpose**: Executes an embedded SQL SELECT query against DB2 to retrieve customer details for a given customer number and evaluates the outcome.
  - **Operations Performed**:
    - Sets SQL request descriptor `EM_SQLREQ` to `' SELECT CUSTOMER'`.
    - Executes an embedded SQL `SELECT` statement fetching `FIRSTNAME`, `LASTNAME`, `DATEOFBIRTH`, `HOUSENAME`, `HOUSENUMBER`, `POSTCODE`, `PHONEMOBILE`, `PHONEHOME`, and `EMAILADDRESS` into the corresponding communication area host variables (`CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_DOB`, `CA_HOUSE_NAME`, `CA_HOUSE_NUM`, `CA_POSTCODE`, `CA_PHONE_MOBILE`, `CA_PHONE_HOME`, `CA_EMAIL_ADDRESS`) using `DB2_CUSTOMERNUMBER_INT` as the search key.
    - Inspects `SQLCODE` and sets `CA_RETURN_CODE` accordingly (`'00'` for success, `'01'` for not found, `'90'` for database errors).
    - Calls `WRITE_ERROR_MESSAGE` when an unexpected database error occurs.
  - **Control Flow Elements**:
    - `SELECT (SQLCODE)` construct branching on:
      - `WHEN (0)`: Sets `CA_RETURN_CODE` to `'00'`.
      - `WHEN (100)`: Sets `CA_RETURN_CODE` to `'01'`.
      - `OTHERWISE`: Sets `CA_RETURN_CODE` to `'90'` and invokes `WRITE_ERROR_MESSAGE`.
  - **Interactions with External Systems**:
    - Executes embedded DB2 SQL `SELECT` against the `CUSTOMER` table.

- **WRITE_ERROR_MESSAGE**
  - **Purpose**: Populates the internal error message data structure with diagnostic details following a database query failure.
  - **Operations Performed**:
    - Captures the SQL return code by moving `SQLCODE` to `EM_SQLRC`.
    - Populates the error timestamp fields `EM_DATE` and `EM_TIME` from working storage variables `WS_DATE` and `WS_TIME`.
  - **Control Flow Elements**:
    - Sequential assignment operations with no conditional branching or looping.
  - **Interactions with External Systems**:
    - None (constructs error information entirely within memory).

## 6. Dependencies

- **Input and output files**
  - CUSTIN
    - Sequential, fixed-block input file with a record size of 10 bytes. The program opens it for input.
    - Each record holds a 10-digit customer number, `CUSTIN_CUSTNUMBER`, which drives the batch loop.
    - An `UNDEFINEDFILE` handler displays "FILE STATUS AT OPEN CUSTIN", sets the return code to 16 through `PLIRETC`, and stops the run if the file can't be opened.
    - An `ENDFILE` handler sets the `FI_EOF` flag to end the processing loop.
    - The DD or dataset allocation for it must be supplied in the run JCL.
  - CUSTRPT
    - Sequential, fixed-block output file with an 80-byte record size.
    - The program writes one formatted report line, `CUSTRPT_LINE`, for each customer processed.
    - Its DD allocation must also be supplied in the run JCL.

- **Database dependencies (Db2 for z/OS)**
  - Db2 subsystem and connection
    - The program contains embedded SQL, so it needs a Db2 precompile or coprocessor step. It also needs a bound plan or package and an attached Db2 environment at run time, such as DSN/TSO batch.
    - The bind step is in the project JCL (`GenAppBind.jcl`).
  - CUSTOMER table
    - Queried with a singleton SELECT that filters on `CUSTOMERNUMBER = :DB2_CUSTOMERNUMBER_INT`.
    - Columns read: FIRSTNAME, LASTNAME, DATEOFBIRTH, HOUSENAME, HOUSENUMBER, POSTCODE, PHONEMOBILE, PHONEHOME and EMAILADDRESS.
    - The table is defined by the DDL job (`GenAppDDL.jcl`).
  - SQL host variables
    - Input: `DB2_CUSTOMERNUMBER_INT` (FIXED BIN(31)) holds the customer number predicate.
    - Output: `CA_FIRST_NAME`, `CA_LAST_NAME`, `CA_DOB`, `CA_HOUSE_NAME`, `CA_HOUSE_NUM`, `CA_POSTCODE`, `CA_PHONE_MOBILE`, `CA_PHONE_HOME` and `CA_EMAIL_ADDRESS` receive the selected columns.
  - SQLCA
    - Included with `EXEC SQL INCLUDE SQLCA`.
    - `SQLCODE` is evaluated after the SELECT. 0 sets `CA_RETURN_CODE` to '00', 100 (not found) sets it to '01', and any other value sets it to '90' and calls `WRITE_ERROR_MESSAGE`.

- **External programs and subroutines**
  - LGBATCHF (`pli/LGBATCHF.pli`)
    - The program loads it dynamically with `FETCH` and calls it as `ENTRY(PIC '9999999999', PTR, CHAR(80))`.
    - The parameters are the customer number, the pointer to the commarea (`COMM_AREA_PTR`), and the 80-character report line buffer, which it fills.
    - It must be available in the load library or STEPLIB concatenation at run time.
  - PLIRETC
    - PL/I built-in used to set the program return code to 16 when CUSTIN can't be opened.

- **Include files and data structure definitions**
  - SQLCA
    - Standard Db2 communication area, pulled in by the SQL INCLUDE.
  - LGCMAREA (`plinc/LGCMAREA.inc`)
    - Commarea layout, expanded inline into the source.
    - Defines the `COMM_AREA` structure over `BATCH_COMM_AREA`, a 32,500-byte work area.
    - The program uses these fields:
      - `CA_RETURN_CODE`
      - `CA_CUSTOMER_NUM`
      - `CA_NUM_POLICIES`
      - the customer request fields listed under the host variables
    - Because the commarea is shared with `LGBATCHF`, both programs depend on the same layout.

- **Runtime and environment dependencies**
  - Enterprise PL/I runtime
    - Supports `FETCH`, record I/O, `ON` conditions, `DISPLAY`, `STOP` and `PLIRETC`.
  - Execution context
    - It runs as a batch, non-CICS main program (`Options(Main)`).
    - Run JCL must provide the Db2 attachment, the load libraries, and the CUSTIN and CUSTRPT DD statements.
  - Working variables
    - `WS_DATE` and `WS_TIME` are initialised to blanks and never populated.
    - They are copied into the `ERROR_MSG` structure, which is built but never written anywhere.
  - Carry-over commarea state
    - If a customer isn't found, the report still uses whatever the previous row left in the commarea.
    - The behavior therefore depends on the earlier SELECT results persisting in `BATCH_COMM_AREA`.

## 7. Constraints

- **Input file constraints**
  - CUSTIN must be a sequential, fixed-blocked record file with 10-byte records.
    - It is declared `FILE RECORD INPUT SEQUENTIAL ENV(FB RECSIZE(10))`.
    - Each record holds one customer number.
  - The customer number (`CUSTIN_CUSTNUMBER`) must be a 10-digit numeric picture field.
    - It is declared `PIC '9999999999'`.
    - The program does no explicit check for non-numeric content, blanks or zero values.
  - CUSTIN must open successfully.
    - An `ON UNDEFINEDFILE(CUSTIN)` unit displays 'FILE STATUS AT OPEN CUSTIN'.
    - The unit then calls `PLIRETC(16)` and runs `STOP`, which ends the program with return code 16.
  - Only CUSTIN has an open-failure handler.
    - There is no equivalent handler for CUSTRPT.
    - There is also no handler for READ, WRITE or other I/O errors.

- **Output file constraints**
  - CUSTRPT must be a sequential, fixed-blocked record file with 80-byte records.
    - It is declared `FILE RECORD OUTPUT SEQUENTIAL ENV(FB RECSIZE(80))`.
  - Every report line is written from the 80-character buffer `CUSTRPT_LINE`.
    - The buffer is passed to `LGBATCHF` and then written with `WRITE FILE(CUSTRPT) FROM(CUSTRPT_LINE)`.
    - Any formatted output that exceeds 80 characters is limited by the buffer size.

- **Sequencing and control-flow constraints**
  - The commarea pointer must be set before any processing.
    - `COMM_AREA_PTR = ADDR(BATCH_COMM_AREA)` runs first.
    - The based `COMM_AREA` structure overlays the work area.
  - The `LGBATCHF` subroutine must be loaded dynamically before it is called.
    - `fetch lgbatchf` runs before the files are opened and before any record is processed.
  - Files must be opened before use.
    - Both files are opened before the first read.
  - The processing order is fixed.
    - CUSTIN is read ahead of the loop and again at the end of each iteration.
    - `PROCESS_RECORD` runs only while `FI_EOF` is false.
  - Each customer is processed in a fixed order inside `PROCESS_RECORD`.
    - The customer number is moved to the DB2 host variable.
    - `GET_CUSTOMER_INFO` runs the SELECT.
    - `LGBATCHF` formats the line.
    - The line is written to CUSTRPT.
  - End of input is detected through the `ON ENDFILE(CUSTIN)` unit.
    - The unit sets `FI_EOF = '1'B`.
    - The loop then stops.
  - Both files are closed after the loop ends, and then the program returns.

- **Data retrieval and database constraints**
  - Customer data is retrieved by exact key match.
    - The query is `WHERE CUSTOMERNUMBER = :DB2_CUSTOMERNUMBER_INT`.
    - It returns at most one row into the host variables.
    - A multi-row result would produce a non-zero SQLCODE (such as -811), which falls into the error path.
  - The customer number is converted to `FIXED BIN(31)` for the query.
    - `DB2_CUSTOMERNUMBER_INT` is initialised to 0.
    - It is assigned from `CUSTIN_CUSTNUMBER` for each record.
  - The SELECT retrieves a fixed set of columns into fixed-length fields.
    - `CA_FIRST_NAME` is 10 characters.
    - `CA_LAST_NAME` is 20 characters.
    - `CA_DOB` is 10 characters in YYYY-MM-DD format.
    - `CA_HOUSE_NAME` is 20 characters.
    - `CA_HOUSE_NUM` is 4 characters.
    - `CA_POSTCODE` is 8 characters.
    - `CA_PHONE_MOBILE` is 20 characters.
    - `CA_PHONE_HOME` is 20 characters.
    - `CA_EMAIL_ADDRESS` is 100 characters.
    - Values longer than these lengths are subject to truncation by the host-variable definitions.
  - `CA_NUM_POLICIES` is declared in the commarea but is not selected or populated by this program.
  - `CA_CUSTOMER_NUM` is also declared in the commarea header but is not set by this program.
    - The customer number reaches `LGBATCHF` only as a separate parameter.

- **Return code and error handling constraints**
  - SQLCODE is mapped to a business return code (`CA_RETURN_CODE`, 2-digit numeric).
    - SQLCODE 0 sets '00', meaning success.
    - SQLCODE 100 sets '01', meaning customer not found.
    - Any other SQLCODE sets '90', meaning database error, and calls `WRITE_ERROR_MESSAGE`.
  - Error messages are built but never output.
    - `WRITE_ERROR_MESSAGE` only fills `EM_SQLRC`, `EM_DATE` and `EM_TIME`.
    - The message is not written to any file or display.
    - `WS_DATE` and `WS_TIME` are never populated and stay blank.
    - `EM_CUSNUM` and `EM_POLNUM` are never populated either.
  - Database errors do not stop processing.
    - The loop continues with the next record.
    - The record is still passed to `LGBATCHF` and written to the report.

- **Business rules and processing restrictions**
  - A customer that is not found, or whose SELECT fails, is still reported.
    - `LGBATCHF` is called and a line is written regardless of `CA_RETURN_CODE`.
    - The commarea fields keep the values left from the previous successful row.
    - This is documented in the program header.
    - A not-found customer is therefore reported with stale data from the previous customer. The not-found status is carried only in `CA_RETURN_CODE`.
  - One report line is produced for every input record.
    - There is no filtering, de-duplication or skipping.

- **Resource and size limits**
  - The commarea work area is limited to 32,500 bytes.
    - `BATCH_COMM_AREA` is `CHAR(32500)`.
    - The `COMM_AREA` structure is a based union over the same 32,500 bytes.
    - The request-specific portion is fixed at 32,482 bytes.
  - Customer request fields are bounded by their declared lengths.
    - `CA_POLICY_DATA` is 32,267 characters.
  - The report line passed to `LGBATCHF` is fixed at 80 characters.
  - The subroutine interface is fixed.
    - `LGBATCHF` is declared with `ENTRY(PIC '9999999999', PTR, CHAR(80))`.
    - The call must pass a 10-digit numeric customer number, a pointer and an 80-character buffer.
  - The program does not set a limit on the number of input records.

- **Environment constraints**
  - The program is a batch, non-CICS program (`Proc Options(Main)`).
  - It requires DB2 access with the SQLCA included.
  - The `CUSTOMER` table must be available.

## 8. Error Handling

- **Input file open error handling**
  - An UNDEFINEDFILE ON-unit is established for the customer input file (CUSTIN) before it is opened.
  - If CUSTIN cannot be opened, the ON-unit displays the message "FILE STATUS AT OPEN CUSTIN" on the console.
  - It then sets the program return code to 16 through the PLIRETC built-in and issues STOP to end the run.
  - The handler is a fail-fast termination path. It has no retry or alternate-file fallback.
  - No equivalent UNDEFINEDFILE handler exists for the output report file (CUSTRPT). A failure to open it is left to default PL/I runtime handling.

- **End-of-file handling**
  - An ENDFILE ON-unit for CUSTIN sets the end-of-file switch (FI_EOF) to '1'B when the read runs out of records.
  - The main processing loop is a DO WHILE that tests that switch. Reading therefore stops cleanly, and the program moves on to normal closing.
  - A read is done before the loop and another at the end of each iteration. An empty input file therefore skips processing and ends normally.

- **Database (DB2) error handling**
  - The SQL return code (SQLCODE) is evaluated in a SELECT/WHEN construct right after the customer SELECT in the customer retrieval routine.
    - SQLCODE 0 sets the business return code (CA_RETURN_CODE) to '00' for success.
    - SQLCODE 100 (row not found) sets CA_RETURN_CODE to '01' for customer not found.
    - Any other SQLCODE falls into the OTHERWISE branch. It sets CA_RETURN_CODE to '90' for a database error and calls the error message routine.
  - The SQL request description (EM_SQLREQ) is set to "SELECT CUSTOMER" before the SQL call. This gives context for any error message.
  - The SQLCA is included so that SQLCODE is available for these checks.

- **Error message construction and notification**
  - The error message routine (WRITE_ERROR_MESSAGE) fills the structured error message record (ERROR_MSG) in three steps:
    - It copies SQLCODE into the formatted numeric field EM_SQLRC.
    - It copies the working date (WS_DATE) into EM_DATE.
    - It copies the working time (WS_TIME) into EM_TIME.
  - The message layout also has the program name, customer number, policy number and SQL request text.
  - The routine only builds the message in memory. It does not write it to a file, display it or log it, so an unexpected DB2 error is not visibly reported.
  - WS_DATE and WS_TIME are initialized to blanks and never populated in this program. The date and time in any message would therefore be blank.
  - The customer number and policy number fields (EM_CUSNUM, EM_POLNUM) are never populated either.

- **Recovery and fallback behavior**
  - A customer not found in DB2 (return code '01') is not skipped or rejected. Processing continues, and the report subroutine (LGBATCHF) is still called.
  - The communication area keeps whatever the previous successful row left in it, so the report line for a missing customer may show stale data.
  - A DB2 error (return code '90') also does not stop the run. The program continues with the next input record.
  - The return code is passed to the report formatting step through the communication area. The report routine can use it to adjust the output.
  - No retry logic, rollback or abort is implemented for SQL errors.

- **Limitations and unhandled conditions**
  - There is no explicit handling for errors on READ or WRITE other than ENDFILE, such as transmission or record conditions.
  - There is no handling for a failure of the dynamic FETCH of the LGBATCHF subroutine.
  - There is no ON ERROR or ON CONDITION handler for other conditions raised by the CALL to LGBATCHF.
  - There is no check of the file close operations.
  - The program returns normally after closing the files. It does not set a non-zero return code when records hit the not-found or DB2-error paths.

## 9. Examples

<invoke name="read_file">
<parameter name="context_focus_query">LGBATCHF subroutine logic: how it formats the report line (CUSTRPT_LINE) using customer data from COMM_AREA, what fields are included, and how CA_RETURN_CODE affects output</parameter>
<parameter name="path">c:\git\github.com\GenAppRulPLI\.bobz\expanded\pli\pli\LGBATCH.pli</parameter>
</invoke>
<invoke name="list_files">
<parameter name="path">c:\git\github.com\GenAppRulPLI\.bobz\expanded\pli\pli</parameter>
</invoke>

---

Generated by IBM Bob Premium Package for Z
