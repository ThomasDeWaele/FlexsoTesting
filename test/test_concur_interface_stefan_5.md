# SAP Cloud Integration Flow Technical Specification

This document details the technical specifications for the integration flows developed in SAP Cloud Integration, based on the provided XML configuration.  The flows are designed to integrate data from SuccessFactors (SFSF), SAP ERP, and potentially a Mock system, ultimately generating a CSV file for Concur.

## Process Overview

The integration flow consists of several interconnected processes:

1. **Start from ProcessDirect:** Initiates the main flow via ProcessDirect adapter.  This flow configures crucial header properties before calling the main process.
2. **Scheduler:**  Initiates the main flow based on a scheduled timer.  Similar to ProcessDirect, this also sets header properties.
3. **Main Process:** The core process orchestrating data retrieval, transformation, and upload to Concur.  This process utilizes several subprocesses:
    * **Get Employees from SFSF:** Retrieves employee data from SuccessFactors.
    * **Get Vendors from SAP ERP:** Retrieves vendor data from SAP ERP.
    * **Enrich from SFSF:** Enriches employee data with additional information from SuccessFactors.
    * **XML to CSV:** Transforms the combined employee and vendor data into CSV format.
4. **Get Employees from SFSF:** Retrieves employee data from SuccessFactors using two different APIs (SOAP and a test API).
5. **Get Vendors from SAP ERP:** Retrieves vendor data from SAP ERP via a SOAP API. A mock API is also available for testing purposes.
6. **Enrich from SFSF:** Enriches the employee data from a SuccessFactors OData V2 API.
7. **XML to CSV:** This process is responsible for the XML to CSV transformation. It contains three sub-processes focusing on header, employee, and delegate information.

**Diagrammatic Overview:**

```
                                              +-----------------+
                                              | Start from     |
                                              | ProcessDirect   |----->  +-----------------+
                                              +-----------------+       | Main Process     |----->  +-----------------+
                                                                        |                 |       | Concur SFTP      |-----> Concur SFTP
                                                                        +-----------------+       +-----------------+
                                                   ^                                                 |
                                                   |                                                 |
                                                   |                                                 |
                                              +-----------------+                                                 |
                                              |       Scheduler      |------------------------------------------------>|
                                              +-----------------+                                                 |
                                                   |                                                 |
                                                   |                                                 |
                                             +-----------------+          +-----------------+         +-----------------+
                                             | Get Employees  |--------> | Enrich from     |-------->| XML to CSV       |
                                             | from SFSF      |          | SFSF           |         +-----------------+
                                             +-----------------+          +-----------------+
                                                                        |
                                                                        |
                                                                    +-----------------+
                                                                    | Get Vendors     |
                                                                    | from SAP ERP    |
                                                                    +-----------------+

```


## Integration Flow Initiation

The integration flow can be initiated in two ways:

1. **ProcessDirect:** The `Start from ProcessDirect` process uses a ProcessDirect sender adapter, allowing external systems to trigger the flow.

2. **Scheduler:** The `Scheduler` process utilizes a timer-based start event, initiating the flow at predefined intervals.


## Detailed Process Description

### 1. Start from ProcessDirect

* **Start 5:** The integration flow begins with a StartEvent triggered by the ProcessDirect adapter.
* **Config Flow from Headers:**  This ContentModifier step creates several header properties.  See the detailed property table below.
* **Main Process:** Calls the `Main Process` (Process_82344329).
* **End:** A final EndEvent signifying the flow's completion.


### 2. Scheduler

* **Start Timer 1:**  The process starts based on a scheduled timer event.
* **Config Flow:** This ContentModifier sets the same header properties as in "Start from ProcessDirect", potentially using different mechanisms (constants vs. headers). See the detailed property table below.
* **Main Process:** Calls the `Main Process`.
* **End 5:** The flow ends.


### 3. Main Process

* **Start 6:** StartEvent initiating the main process.
* **Get Employees from SFSF:** Calls the `Get Employees from SFSF` process (Process_9431).  This retrieves employee data from SuccessFactors. Uses two different APIs: a standard SOAP API and a test API (`SFSF_Test`).
* **Get Vendors from SAP ERP:** Calls the `Get Vendors from SAP ERP` process (Process_9472). This retrieves vendor information from SAP ERP (or a Mock system during testing).
* **Enrich from SFSF:** Calls the `Enrich from SFSF` process (Process_9448) to enrich the employee data from SuccessFactors, using an OData V2 API.  This step calls two OData services to enrich.
* **XML to CSV:** Calls the `XML to CSV` process (Process_82344268) to transform the combined data into CSV format.
* **Remove nodes without Person:** Removes XML nodes without a "Person" element.
* **Remove incomplete Employees:** Filters out employees with incomplete data.
* **Remove employees with company code C628 and country code not Norway:** Filters out specific employee records based on company and country codes.
* **Keep newest employment:** Selects the most recent employment record for each employee.
* **Set Highest Direct Reports:** Sets the highest direct report for each employee.
* **AddBOM:** Adds a Bill of Materials (BOM) – this step’s functionality needs further clarification.
* **Router 1:** Routes the flow based on the `skipEncryption` property. If true, skips encryption; otherwise, proceeds to encryption.
* **PGPEncryptor 1:** Encrypts the data using PGP encryption.
* **Skip encryption for testing:** Skips encryption if the `skipEncryption` property is true.
* **Log Msg:** Logs a message (further details required).
* **Concur SFTP:** Uploads the CSV data to Concur via SFTP.
* **End 6:** EndEvent signifying completion.


### 4. XML to CSV

* **Start 4:** StartEvent of the XML to CSV process.
* **Set empty body:** Sets the message body to empty. This step's purpose needs further clarification.
* **XML to CSV Header:** Processes the header part of the XML data and transforms it to CSV.
* **XML to CSV Employee:** Processes the employee data from the XML and transforms it to CSV.
* **XML to CSV Delegate:** Processes the delegate data from the XML and transforms it to CSV.
* **Gather 1:** Aggregates the transformed header, employee, and delegate data.
* **Router 2:** Routes the flow based on the `SkipDelegates` property. If true, skips processing delegate data; otherwise, proceeds to process it.
* **End 4:** EndEvent of the XML to CSV process.

### 5. Get Employees from SFSF

* **Start 1:** Starts the process.
* **GetEmployees:** Retrieves employee data from SuccessFactors via a SOAP API.
* **Test Employees:** Retrieves data from a SuccessFactors test API (`SFSF_Test`).
* **End 1:** Ends the process.


### 6. Get Vendors from SAP ERP

* **Start 3:** StartEvent of the process.
* **Remove XML Declaration:** Removes the XML declaration from the input.
* **Save XML:** Saves the XML payload in a message property named 'employeeXML'.
* **Generate Request GetVendorId:** Generates a request to get Vendor Ids.
* **GetVendors:** Retrieves Vendor data from SAP ERP using a SOAP API.
* **GetVendorMock:** Retrieves Vendor data from a Mock API.  Used for testing.
* **Combine Vendors & Employees:** Combines the retrieved Vendor and Employee data.
* **Content Modifier 1:** Modifies the combined data (further details required).
* **Merge Vendors & Employees:** Merges employee and vendor data.
* **Filter:** Filters the combined data.
* **End 3:** EndEvent of the process.


### 7. Enrich from SFSF

* **Start 2:** StartEvent of the process.
* **Enrich FOCompany:** Enriches the data with FOCompany information using an OData V2 API call to SuccessFactors.
* **Enrich person_id_ext for relation:** Enriches the data with `person_id_ext` information using an OData V2 API call to SuccessFactors.
* **End 2:** EndEvent of the process.


## Content Modifier Property Table

The following tables detail the properties created in the ContentModifier steps.

**Start from ProcessDirect & Scheduler - Header Properties:**

| Property Name             | Action | Type    | Value                     | Default | Datatype | Description                                     |
|--------------------------|--------|---------|-----------------------------|---------|----------|-------------------------------------------------|
| Concur_SFTP_Directory    | Create | header  | Concur_SFTP_Directory      |          |          | Concur SFTP Directory path                    |
| SkipDelegates            | Create | header  | SkipDelegates              |          |          | Flag to skip delegate processing               |
| CashAdvanceAccountCode   | Create | header  | CashAdvanceAccountCode      |          |          | Cash advance account code                     |
| TestUsersSFSF            | Create | header  | TestUsersSFSF              |          |          | Flag for SuccessFactors test users              |
| CompanyListSFSF          | Create | header  | CompanyListSFSF            |          |          | List of companies in SuccessFactors             |
| InactiveDaysSF           | Create | header  | InactiveDaysSF             |          |          | Number of inactive days in SuccessFactors       |
| skipEncryption           | Create | header  | skipEncryption              |          |          | Flag to skip encryption                        |
| local_log                | Create | header  | local_log                   |          |          | Flag to enable local logging                   |
| LogicalSystem            | Create | header  | LogicalSystem               |          |          | Logical system identifier                     |


**Save XML - Expression Property:**

| Property Name | Action | Type    | Value          | Default | Datatype | Description                        |
|---------------|--------|---------|-----------------|---------|----------|------------------------------------|
| employeeXML   | Create | expression | `${in.body}`    |          |          | Stores the incoming XML message body |


**Note:** The `Datatype` column is left blank in the XML, indicating that the datatype is not explicitly specified.  This should be clarified and added to the specification.


This detailed technical specification provides a comprehensive overview of the SAP Cloud Integration flows.  Further clarification on specific steps and their functionalities might be required for complete implementation understanding.
