# Technical Specification: SAP Cloud Integration Flow

This document provides a technical specification for the integration flows defined in the provided XML file.  It details the processes, their initiation methods, and the steps involved.


## Process Overview

The integration flow consists of several interconnected processes:

* **XML to CSV:** Converts XML data to CSV format.
* **Start from ProcessDirect:** Initiated via ProcessDirect, configures flow headers, and calls the Main Process.
* **Scheduler:** A scheduled process that configures the flow and calls the Main Process.
* **Get Vendors from SAP ERP:** Retrieves vendor data from SAP ERP (or a Mock system).
* **Main Process:** The central process orchestrating data transformation, enrichment, and file transfer to Concur SFTP.
* **Enrich from SFSF:** Enriches data from SuccessFactors.
* **Get Employees from SFSF:** Retrieves employee data from SuccessFactors.


## Integration Flow Initiation

The integration flows can be started in three ways:

1. **ProcessDirect (Start from ProcessDirect):** This flow initiates via a ProcessDirect sender adapter.  It's triggered externally through a ProcessDirect call and sets several header properties before calling the Main Process.

2. **Scheduler:** This flow is triggered by a timer-based event ("Start Timer 1").  It sets header properties using constant and expression values before calling the Main Process.

3. **Manually (Implied):** While not explicitly defined by a sender adapter in the provided XML, other processes (e.g., those triggered by events outside the scope of this definition) could potentially initiate subprocesses within this flow.


## Detailed Process Description


### 1. XML to CSV

This process transforms XML data into CSV format.  It consists of several parallel branches:

* **Start 4:** The start event of the process.
* **Set empty body:** Sets the message body to empty, likely as a pre-processing step.
* **XML to CSV Employee:** Converts the employee section of the XML to CSV.
* **XML to CSV Delegate:** Converts the delegate section of the XML to CSV.  This step is skipped if the `SkipDelegates` property is set to `true`.
* **XML to CSV Header:** Converts the header section of the XML to CSV.
* **Gather 1:** Gathers the results from the three CSV conversion steps into a single message.
* **End 4:** The end event of the process.

A router ("Router 2") determines whether to process the Delegate section based on the `SkipDelegates` header property.


### 2. Start from ProcessDirect

This process is initiated via a ProcessDirect sender adapter named "ProcessDirect".

* **Start 5:** Start event triggered by ProcessDirect.
* **Config Flow from Headers:**  This Content Modifier step creates several header properties.  See detailed table below.
* **Main Process:** Calls the main process, passing the configured message.
* **End:** End event of the process.


**Content Modifier Property Details (Config Flow from Headers):**

| Name                     | Action | Type    | Value                 | Default | Datatype |
|--------------------------|--------|---------|-----------------------|---------|-----------|
| Concur_SFTP_Directory    | Create | header  | Concur_SFTP_Directory |         |           |
| SkipDelegates            | Create | header  | SkipDelegates         |         |           |
| CashAdvanceAccountCode   | Create | header  | CashAdvanceAccountCode  |         |           |
| TestUsersSFSF            | Create | header  | TestUsersSFSF          |         |           |
| CompanyListSFSF          | Create | header  | CompanyListSFSF        |         |           |
| InactiveDaysSF           | Create | header  | InactiveDaysSF         |         |           |
| skipEncryption           | Create | header  | skipEncryption         |         |           |
| local_log                | Create | header  | local_log              |         |           |
| LogicalSystem            | Create | header  | LogicalSystem          |         |           |


### 3. Scheduler

This process is initiated by a timer.

* **Start Timer 1:** Timer-based start event.
* **Config Flow:**  This Content Modifier step creates several header properties. The values are defined using constants and expressions. Note that the constants likely reference values configured outside this XML definition (e.g., from system properties).
* **Main Process:** Calls the main process.
* **End 5:** End event of the process.


### 4. Get Vendors from SAP ERP

This process retrieves vendor data.

* **Start 3:** The start event.
* **Remove XML Declaration:** Removes the XML declaration from the input message.
* **Generate Request GetVendorId:** Generates the request message for retrieving vendor IDs.
* **Save XML:** Saves the input XML to a property named `employeeXML`.
* **GetVendors:** Calls SAP ERP SOAP API to retrieve vendor data. This calls a 3rd party system (SAP_ERP).  The API credentials are defined by the `{{ECC_CRED}}` expression, likely resolving to a system property or configuration value.
* **GetVendorMock:** Calls a Mock API to retrieve vendor data (for testing purposes).  This likely represents a testing alternative to avoid calls to SAP ERP.
* **Combine Vendors & Employees:** Combines the vendor and employee data.
* **Merge Vendors & Employees:** Merges the data from both vendor sources (SAP ERP and Mock) if necessary.
* **Filter:** Filters the combined data.
* **End 3:** The end event.


### 5. Main Process

This is the central orchestration process.

* **Start 6:** The start event.
* **Get Employees from SFSF:** Calls the "Get Employees from SFSF" process to retrieve employee data from SuccessFactors. This calls a 3rd party system (SFSF_SOAP & SFSF_Test).  The system called depends on the flow.
* **Remove nodes without Person:** Removes nodes from the XML that lack a Person element.
* **Remove incomplete Employees:** Removes incomplete employee records.
* **Remove employees with company code C628 and country code not Norway:** Filters out specific employee records.
* **Remove unpaid leave employees:** Removes employee records marked as unpaid leave.
* **Remove employment_information and job_information with lower fte:** Removes employment and job information where the FTE is below a certain threshold.
* **Keep newest employment:** Keeps only the newest employment records for each employee.
* **Set Highest Direct Reports:** Sets the highest direct report for each employee.
* **Filter current nodes:** Filters the remaining employee nodes.
* **Employee SF to Concur:** Transforms employee data for Concur.
* **Get Vendors from SAP ERP:** Calls the "Get Vendors from SAP ERP" process.
* **XML to CSV:** Calls the "XML to CSV" process.
* **AddBOM:** Adds a Bill of Materials (likely a supplementary data element).
* **Router 1:** Routes the message based on the `skipEncryption` property.
* **PGPEncryptor 1:** Encrypts the data using PGP (if `skipEncryption` is false).
* **Skip encryption for testing:** Skips encryption (if `skipEncryption` is true).
* **Log Msg:** Logs a message.
* **Concur SFTP:** Transfers the processed data to Concur via SFTP. This calls a 3rd party system (Concur_SFTP).
* **End 6:** The end event.


### 6. Enrich from SFSF

This process enriches employee data from SuccessFactors.

* **Start 2:** Start event.
* **Enrich FOCompany:** Calls a SuccessFactors API to enrich company information.  This calls a 3rd party system.
* **Enrich person_id_ext for relation:** Calls a SuccessFactors API to enrich additional person ID information.  This calls a 3rd party system.
* **End 2:** End event.


### 7. Get Employees from SFSF

This process retrieves employee data from SuccessFactors.

* **Start 1:** Start event.
* **GetEmployees:** Calls SuccessFactors SOAP API to retrieve employee data. This calls a 3rd party system (SFSF_SOAP).
* **Test Employees:** Calls a SuccessFactors API for testing purposes. This calls a 3rd party system (SFSF_Test).
* **End 1:** End event.


This detailed specification provides a comprehensive understanding of the integration flow's logic and functionality.  Remember to consult the external configurations (e.g., API credentials, system properties) to fully understand the complete system behavior.
