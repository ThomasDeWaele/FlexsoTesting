# SAP Cloud Integration Flow Technical Specification

## Process Overview

This document details the technical specifications for an SAP Cloud Integration flow designed to extract employee data from SuccessFactors (SFSF), enrich it with data from SAP ERP, and finally, transfer the processed data to Concur via SFTP.  The flow is composed of several interconnected processes:

1. **Start from ProcessDirect:** Initiates the flow via a ProcessDirect adapter.  This allows external systems to trigger the integration process.
2. **Scheduler:**  Provides a scheduled execution mechanism for the flow using a timer.
3. **Get Employees from SFSF:** Retrieves employee data from SuccessFactors using SOAP and SuccessFactors adapters. Includes a test call to validate the connection.
4. **Enrich from SFSF:** Enriches the employee data retrieved from SFSF by calling SFSF's OData V2 API.
5. **Get Vendors from SAP ERP:** Retrieves vendor data from SAP ERP using a SOAP adapter.  Includes a mock API call for testing.
6. **Main Process:**  The central orchestration process which performs data transformations, filtering, encryption (optional), and the final SFTP transfer to Concur.
7. **XML to CSV:** Converts XML data to CSV format.

The overall flow can be visualized as follows:

```
[Start from ProcessDirect] --> [Config Flow from Headers] --> [Main Process] --> [End]
                                 ^
                                 |
                 [Scheduler] --> [Config Flow] --> [Main Process] --> [End 5]
                                         |
                                         V
                      [Get Employees from SFSF] --> [Enrich from SFSF] --> ... --> [Concur SFTP]
                                                       |
                                                       V
                                                [Get Vendors from SAP ERP]
                                                        |
                                                        V
                                                    [XML to CSV] 
```

## Integration Flow Initiation

The integration flow can be started in two ways:

1. **ProcessDirect:** This adapter allows for external systems to trigger the flow directly. The `Start from ProcessDirect` process uses this adapter to start the flow.

2. **Timer:** The `Scheduler` process employs a timer event to trigger the flow at predetermined intervals.


## Detailed Process Description

### 1. Start from ProcessDirect

* **Process Name:** Start from ProcessDirect
* **Description:** This process acts as the entry point for externally triggered integrations.  It receives the initial request from a ProcessDirect adapter and triggers the `Config Flow from Headers` step.
* **Sender Adapter:** ProcessDirect
* **Steps:**
    * **Start 5:**  Start event initiated by the ProcessDirect adapter.
    * **Config Flow from Headers:** Sets several header properties using a Content Modifier (see detailed property table below).  These properties control the flow's behavior, such as skipping delegate processing or enabling/disabling logging.
    * **Main Process:** Calls the central `Main Process` (Process_82344329).
    * **End:** End event marking the completion of this initiation process.

### 2. Scheduler

* **Process Name:** Scheduler
* **Description:** This process starts the integration flow periodically based on a configured timer schedule.
* **Steps:**
    * **Start Timer 1:** A timer start event triggers the process.
    * **Config Flow:**  Sets several header properties using a Content Modifier (see detailed property table below). These properties are set using constant or expression values.
    * **Main Process:** Calls the central `Main Process` (Process_82344329).
    * **End 5:** End event marking the completion of this scheduled process.

### 3. Get Employees from SFSF

* **Process Name:** Get Employees from SFSF
* **Description:** This process retrieves employee data from SuccessFactors.  It includes both a main call and a test call.
* **Steps:**
    * **Start 1:** Start event.
    * **GetEmployees:** Calls the SuccessFactors (SFSF_SOAP) API to retrieve employee data.  Uses SuccessFactors adapter.
    * **Test Employees:** Calls a test SuccessFactors API (SFSF_Test) to validate the connection.  Uses SuccessFactors adapter.
    * **End 1:** End event.

### 4. Enrich from SFSF

* **Process Name:** Enrich from SFSF
* **Description:** This process enriches the employee data obtained from SuccessFactors using OData V2 API calls.
* **Steps:**
    * **Start 2:** Start event.
    * **Enrich FOCompany:**  OData V2 API call to enrich with FOCompany data. Uses OData V2 protocol. Uses SF_CRED for authentication and SF_URL for the endpoint.
    * **Enrich person_id_ext for relation:** OData V2 API call to enrich data with  `person_id_ext` for relationship data. Uses OData V2 protocol. Uses SF_CRED for authentication and SF_URL for the endpoint.
    * **End 2:** End event.

### 5. Get Vendors from SAP ERP

* **Process Name:** Get Vendors from SAP ERP
* **Description:** This process retrieves vendor data from SAP ERP and also includes a Mock API for testing purposes.
* **Steps:**
    * **Start 3:** Start event.
    * **Remove XML Declaration:** Removes the XML declaration from the input XML.
    * **Save XML:** Saves the XML payload as a property called `employeeXML`.
    * **Generate Request GetVendorId:** Generates the request to retrieve vendor IDs.
    * **GetVendors:** Calls the SAP ERP API (SOAP adapter) to retrieve vendor data. Uses ECC_CRED for authentication.  The Receiver is identified as SAP_ERP
    * **GetVendorMock:** Calls a Mock API to retrieve vendor data (for testing).  Uses SOAP adapter. The Receiver is identified as Mock.
    * **Combine Vendors & Employees:** Combines the vendor data with the existing employee data.
    * **Merge Vendors & Employees:** Merges the vendor and employee data.
    * **Filter:** Filters the combined data.
    * **Content Modifier 1:** Performs additional data modifications.
    * **End 3:** End event.

### 6. Main Process

* **Process Name:** Main Process
* **Description:** This is the central process, performing various data transformations, filtering, and encryption (optional) before sending the data to Concur via SFTP.
* **Steps:**
    * **Start 6:** Start event.
    * **Get Employees from SFSF:** Calls the `Get Employees from SFSF` process (Process_9431).
    * **Enrich from SFSF:** Calls the `Enrich from SFSF` process (Process_9448).
    * **Get Vendors from SAP ERP:** Calls the `Get Vendors from SAP ERP` process (Process_9472).
    * **Set Highest Direct Reports:** Sets the highest direct reports.
    * **Employee SF to Concur:** Prepares the employee data for transfer to Concur.
    * **XML to CSV:** Calls the `XML to CSV` process (Process_82344268) to convert the data to CSV format.
    * **AddBOM:** Adds a Bill of Materials (BOM).
    * **Router 1:** Routes the flow based on the `skipEncryption` property.
    * **PGPEncryptor 1:** Encrypts the data using PGP encryption.
    * **Skip encryption for testing:** Skips encryption if `skipEncryption` is true.
    * **Log Msg:** Logs a message.
    * **Remove employment_information and job_information with lower fte:** Removes employment and job information with lower FTE.
    * **Remove unpaid leave employees:** Removes employees on unpaid leave.
    * **Remove employees with company code C628 and country code not Norway:** Removes specific employees based on criteria.
    * **Keep newest employment:** Keeps only the newest employment record.
    * **Remove nodes without Person:** Removes nodes without Person information.
    * **Remove incomplete Employees:** Removes incomplete employee records.
    * **Filter current nodes:** Filters the current nodes.
    * **Concur SFTP:** Sends the processed data to Concur via SFTP. Uses SFTP adapter. The Receiver is identified as Concur_SFTP.
    * **End 6:** End event.

### 7. XML to CSV

* **Process Name:** XML to CSV
* **Description:** This process converts XML data to CSV format.  It is broken down into three sub-processes for header, employee, and delegate information.  These are not individually identified in the provided XML, but the logic suggests this structured approach.
* **Steps:**
    * **Start 4:** Start event.
    * **Set empty body:** Sets an empty body to initiate the process.
    * **XML to CSV Header:** Processes the header section of the XML data for CSV conversion.
    * **XML to CSV Employee:** Processes the employee section of the XML data for CSV conversion.
    * **XML to CSV Delegate:** Processes the delegate section of the XML data for CSV conversion (potentially skipped based on a routing decision).
    * **Gather 1:** Gathers the processed CSV data from header, employee, and delegate sections.
    * **Router 2:** Routes the flow based on the `SkipDelegates` property.  If true, the delegate processing is skipped.
    * **End 4:** End event.


## Content Modifier Property Table

The following table details the properties set within the `ContentModifier` steps:

| Process Name             | Property Name             | Action | Type      | Value                      | Default | Datatype | Description                                                              |
|--------------------------|--------------------------|--------|-----------|-----------------------------|---------|----------|--------------------------------------------------------------------------|
| Start from ProcessDirect | Concur_SFTP_Directory    | Create | header    | Concur_SFTP_Directory       |         |          | Directory for Concur SFTP                                                |
| Start from ProcessDirect | SkipDelegates            | Create | header    | SkipDelegates                |         |          | Flag to skip delegate processing                                          |
| Start from ProcessDirect | CashAdvanceAccountCode   | Create | header    | CashAdvanceAccountCode       |         |          | Cash advance account code                                                |
| Start from ProcessDirect | TestUsersSFSF            | Create | header    | TestUsersSFSF               |         |          | SuccessFactors test users                                              |
| Start from ProcessDirect | CompanyListSFSF          | Create | header    | CompanyListSFSF             |         |          | SuccessFactors company list                                             |
| Start from ProcessDirect | InactiveDaysSF           | Create | header    | InactiveDaysSF              |         |          | Number of inactive days in SuccessFactors                               |
| Start from ProcessDirect | skipEncryption           | Create | header    | skipEncryption              |         |          | Flag to skip encryption                                                 |
| Start from ProcessDirect | local_log                | Create | header    | local_log                   |         |          | Flag to enable local logging                                           |
| Start from ProcessDirect | LogicalSystem            | Create | header    | LogicalSystem               |         |          | Logical system identifier                                                |
| Scheduler                | Concur_SFTP_Directory    | Create | constant  | {{Concur_SFTP_Directory}}   |         |          | Directory for Concur SFTP                                                |
| Scheduler                | SkipDelegates            | Create | constant  | {{SkipDelegates}}            |         |          | Flag to skip delegate processing                                          |
| Scheduler                | CashAdvanceAccountCode   | Create | constant  | {{CashAdvanceAccountCode}}   |         |          | Cash advance account code                                                |
| Scheduler                | TestUsersSFSF            | Create | expression | {{TestUsersSFSF}}           |         |          | SuccessFactors test users                                              |
| Scheduler                | CompanyListSFSF          | Create | constant  | {{CompanyListSFSF}}          |         |          | SuccessFactors company list                                             |
| Scheduler                | InactiveDaysSF           | Create | constant  | {{InactiveDaysSF}}           |         |          | Number of inactive days in SuccessFactors                               |
| Scheduler                | skipEncryption           | Create | constant  | {{skip_encryption}}         |         |          | Flag to skip encryption                                                 |
| Scheduler                | local_log                | Create | constant  | {{enable_log}}              |         |          | Flag to enable local logging                                           |
| Scheduler                | LogicalSystem            | Create | constant  | {{LogicalSystem}}           |         |          | Logical system identifier                                                |
| Get Vendors from SAP ERP | employeeXML              | Create | expression | ${in.body}                   |         |          | XML payload containing employee data                                     |


**Note:**  The `{{...}}` notation in the Scheduler process indicates the use of predefined variables or expressions.  The actual values will be substituted during runtime.


This detailed specification provides a comprehensive overview of the integration flow, enabling a new team member to quickly understand the logic and functionality.  Further documentation regarding specific API calls, error handling, and security aspects may be necessary for complete implementation.
