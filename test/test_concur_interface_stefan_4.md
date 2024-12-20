# Technical Specification: SAP Cloud Integration Flows

This document details the technical specifications for the SAP Cloud Integration flows described in the provided XML file.  The specifications are designed to be easily understood by a new team member.

## Process Overview

The integration flows described in this document handle the processing of employee data from SuccessFactors (SFSF) and SAP ERP, transforming it into a CSV file and uploading it to Concur via SFTP.  The process involves several steps, including API calls to external systems, data transformations, and routing logic.  There are three main processes:

* **XML to CSV:** Transforms XML employee data into a CSV format.
* **Main Process:** Orchestrates the entire data flow, from retrieving data to uploading to Concur.
* **Get Vendors from SAP ERP:** Retrieves vendor data from SAP ERP.  Additional processes handle data retrieval from SFSF and enrichment operations.

## Integration Flow Initiation

The integration flows can be initiated in three ways, each corresponding to a different Sender Adapter:

1. **ProcessDirect:** The 'Start from ProcessDirect' process initiates the flow directly through a ProcessDirect adapter. This is likely used for manual triggering or testing purposes.

2. **Timer:** The 'Scheduler' process uses a timer to trigger the main process at scheduled intervals. This enables automated execution.

3. **Event-based (Implicit):**  The other processes are triggered by the completion of preceding processes in the flow. This is implicit and is managed by the sequence flows defined in each process.


## Detailed Process Description

### 1. XML to CSV

This process transforms XML employee data into a CSV file.

* **Start 4:** The process starts.
* **XML to CSV Header:** Processes the header section of the XML data. This step likely extracts header information and prepares it for CSV conversion.
* **XML to CSV Employee:** Processes the employee details in the XML. This step likely handles the transformation of employee records into a CSV-compatible structure.
* **XML to CSV Delegate:** Processes the delegate details within the XML, transforming this into a format for inclusion in the final CSV.
* **Set empty body:** Sets an empty body to ensure consistent message structure prior to further processing.
* **Gather 1:** Aggregates the processed header, employee, and delegate data into a single message ready for CSV conversion.
* **End 4:** The process completes.


### 2. Start from ProcessDirect

This process starts the main data flow when triggered via the ProcessDirect adapter.

* **Start 5:** The integration flow starts via the ProcessDirect adapter named "ProcessDirect".
* **Config Flow from Headers:** This step creates header properties with values derived likely from the input message header.
    | Property Name             | Action | Type    | Value                     | Default | Datatype | Description                                          |
    |--------------------------|--------|---------|---------------------------|---------|----------|------------------------------------------------------|
    | Concur_SFTP_Directory    | Create | header  | Concur_SFTP_Directory     |          |          | SFTP directory path for Concur upload.            |
    | SkipDelegates             | Create | header  | SkipDelegates              |          |          | Flag to skip processing of delegates (boolean).    |
    | CashAdvanceAccountCode    | Create | header  | CashAdvanceAccountCode    |          |          | Account code for cash advances.                     |
    | TestUsersSFSF             | Create | header  | TestUsersSFSF             |          |          | Flag for test user data (boolean, likely).         |
    | CompanyListSFSF           | Create | header  | CompanyListSFSF           |          |          | List of companies to include (likely).             |
    | InactiveDaysSF            | Create | header  | InactiveDaysSF            |          |          | Number of inactive days considered.                 |
    | skipEncryption            | Create | header  | skipEncryption            |          |          | Flag to skip PGP encryption (boolean).             |
    | local_log                 | Create | header  | local_log                 |          |          | Flag to enable local logging (boolean, likely).    |
    | LogicalSystem             | Create | header  | LogicalSystem             |          |          | Logical system identifier.                         |
* **Main Process:** Calls the 'Main Process' (Process_82344329).
* **End:** The process ends.


### 3. Scheduler

This process triggers the main data flow at scheduled intervals.

* **Start Timer 1:** The process starts based on a scheduled timer.
* **Config Flow:** Creates constant and expression-based properties, similar to step 2 but with potentially different value sourcing.  These properties may be pulled from Integration Flow properties or other configuration.
    | Property Name             | Action | Type      | Value                           | Default | Datatype | Description                                          |
    |--------------------------|--------|-----------|---------------------------------|---------|----------|------------------------------------------------------|
    | Concur_SFTP_Directory    | Create | constant  | {{Concur_SFTP_Directory}}       |          |          | SFTP directory path for Concur upload.            |
    | SkipDelegates             | Create | constant  | {{SkipDelegates}}                |          |          | Flag to skip processing of delegates (boolean).    |
    | CashAdvanceAccountCode    | Create | constant  | {{CashAdvanceAccountCode}}        |          |          | Account code for cash advances.                     |
    | TestUsersSFSF             | Create | expression | {{TestUsersSFSF}}                 |          |          | Flag for test user data (boolean, likely).         |
    | CompanyListSFSF           | Create | constant  | {{CompanyListSFSF}}               |          |          | List of companies to include (likely).             |
    | InactiveDaysSF            | Create | constant  | {{InactiveDaysSF}}                |          |          | Number of inactive days considered.                 |
    | skipEncryption            | Create | constant  | {{skip_encryption}}              |          |          | Flag to skip PGP encryption (boolean).             |
    | local_log                 | Create | constant  | {{enable_log}}                   |          |          | Flag to enable local logging (boolean, likely).    |
    | LogicalSystem             | Create | constant  | {{LogicalSystem}}                 |          |          | Logical system identifier.                         |
* **Main Process:** Calls the 'Main Process' (Process_82344329).
* **End 5:** The process ends.


### 4. Get Vendors from SAP ERP

This process retrieves vendor data from SAP ERP.

* **Start 3:** The process begins.
* **Remove XML Declaration:** Removes the XML declaration from the incoming XML message.  This is a common preprocessing step.
* **Generate Request GetVendorId:** Generates the request for retrieving vendor IDs based on the processed employee XML.
* **GetVendors:** Calls the SAP ERP API (SOAP Adapter) to fetch vendor data.  This uses an API credential named `{{ECC_CRED}}`.
* **GetVendorMock:** Calls a mock API (SOAP Adapter) to retrieve vendor data, presumably for testing purposes.
* **Save XML:** Saves the employee XML to a property named `employeeXML`.
* **Combine Vendors & Employees:** Combines the vendor data with employee data.
* **Content Modifier 1:** Performs additional content modifications. Details not provided in XML.
* **Merge Vendors & Employees:** Merges the vendor and employee data.  The exact merging logic is not provided in the XML.
* **Filter:** Filters the combined data, the criteria is unknown from the XML
* **End 3:** The process ends.


### 5. Main Process

This is the core process orchestrating the entire data flow.

* **Start 6:** The process begins.
* **Get Employees from SFSF:** Calls 'Get Employees from SFSF' (Process_9431).  Retrieves employee data from SuccessFactors using a SuccessFactors Adapter.
* **Remove nodes without Person:** Removes any nodes from the XML structure that don't contain person information.
* **Remove incomplete Employees:** Removes employees with incomplete data.
* **Remove employees with company code C628 and country code not Norway:** Filters employees based on company code and country.
* **Remove unpaid leave employees:** Removes employees on unpaid leave.
* **Keep newest employment:** Keeps only the newest employment record for each employee.
* **Set Highest Direct Reports:** Sets the highest direct reports for each employee.
* **Filter current nodes:** Filters the employee nodes further - criteria not described in XML.
* **Employee SF to Concur:** Transforms and prepares employee data for Concur.
* **Get Vendors from SAP ERP:** Calls 'Get Vendors from SAP ERP' (Process_9472).
* **XML to CSV:** Calls the 'XML to CSV' process (Process_82344268).
* **AddBOM:** Adds a Bill of Materials (BOM), function unknown without further details.
* **Router 1:** Routes the message based on the `skipEncryption` property. If `true`, it skips encryption; otherwise, it proceeds with encryption.
* **PGPEncryptor 1:** Encrypts the data using PGP encryption.
* **Skip encryption for testing:** Skips PGP encryption (if `skipEncryption` was true in the Router 1 step).
* **Log Msg:** Logs a message, purpose not clear from the XML
* **Concur SFTP:** Uploads the (potentially encrypted) data to Concur using an SFTP adapter.
* **End 6:** The process ends.


### 6. Enrich from SFSF

This process enriches employee data from SuccessFactors using OData API calls.

* **Start 2:** The process starts.
* **Enrich FOCompany:** Enriches the data using an OData V2 API call to `{{SF_URL}}` endpoint `FOCompany` using credential alias `{{SF_CRED}}`.
* **Enrich person_id_ext for relation:**  Enriches data with further information using another OData V2 API call to `{{SF_URL}}` endpoint `EmpEmployment` using credential alias `{{SF_CRED}}`.
* **End 2:** The process ends.


### 7. Get Employees from SFSF

This process retrieves employee data from SuccessFactors.

* **Start 1:** The process starts.
* **GetEmployees:** Calls SuccessFactors API (SuccessFactors Adapter) to fetch employee data from SFSF_SOAP endpoint.
* **Test Employees:** Calls a test endpoint within SuccessFactors (SFSF_Test). Likely for testing purposes.
* **End 1:** The process ends.


This detailed specification provides a comprehensive understanding of the integration flows.  Further details may be required to fully understand certain steps, particularly the data transformations and filtering criteria which aren't completely specified within the XML.  The placeholders `{{...}}` indicate values that should be configured externally.
