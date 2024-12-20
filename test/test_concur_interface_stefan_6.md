# SAP Cloud Integration Flow Technical Specification

This document details the technical specifications for the integration flows described in the provided XML file.  The flows are designed for data processing and exchange between SAP Cloud Integration and various third-party systems.

## 1. Process Overview

The integration process consists of several interconnected flows:

* **Start from ProcessDirect:** This flow initiates the overall process via a ProcessDirect sender adapter. It configures headers and then calls the Main Process.
* **Scheduler:** This flow triggers the Main Process periodically using a timer.  It also configures headers.
* **Main Process:** This is the central orchestration flow. It retrieves employee data from SuccessFactors (SFSF), enriches it with additional data from SFSF, retrieves vendor data from SAP ERP, performs data transformations and filtering, encrypts the data (optionally), and finally sends the processed data to Concur via SFTP.
* **XML to CSV:** This sub-flow transforms XML data into CSV format.
* **Get Vendors from SAP ERP:** This sub-flow retrieves vendor data from SAP ERP.
* **Enrich from SFSF:** This sub-flow enriches employee data obtained from SuccessFactors.
* **Get Employees from SFSF:** This sub-flow retrieves employee data from SuccessFactors.


**Diagrammatic Representation:**

```
+-----------------+      +-----------------+      +-----------------+      +-----------------+
| ProcessDirect    |---->| Config Flow     |---->| Main Process     |---->| Concur SFTP     |
+-----------------+      +-----------------+      +-----------------+      +-----------------+
       ^                                                                       |
       |                                                                       v
       +----------------------------------------------------------------------+-----------------+
                                                                               | End 6           |
                                                                               +-----------------+
       |                                                                               ^
       |                                                                               |
       +----------------------------------------------------------------------+-----------------+
                                                                               | XML to CSV      |
                                                                               +-----------------+
       |                                                                               ^
       |                                                                               |
       +----------------------------------------------------------------------+-----------------+
                                                                               | Get Vendors from|
                                                                               | SAP ERP         |
                                                                               +-----------------+
       |                                                                               ^
       |                                                                               |
       +----------------------------------------------------------------------+-----------------+
                                                                               | Enrich from SFSF|
                                                                               +-----------------+
       |                                                                               ^
       |                                                                               |
       +----------------------------------------------------------------------+-----------------+
                                                                               | Get Employees   |
                                                                               | from SFSF       |
                                                                               +-----------------+
       |
       +-----------------+      +-----------------+
       |     Scheduler    |---->| Config Flow     |
       +-----------------+      +-----------------+
```


## 2. Integration Flow Initiation

The integration flow can be initiated in two ways:

1. **ProcessDirect:** The `Start from ProcessDirect` process uses a ProcessDirect sender adapter named "ProcessDirect" to initiate the flow. This allows external systems to trigger the process directly.

2. **Scheduler:** The `Scheduler` process uses a timer start event ("Start Timer 1") to initiate the flow periodically.  This allows for automated, scheduled execution of the process.


## 3. Detailed Process Description

### 3.1 Start from ProcessDirect

* **Start 5:** The process begins with a Start Event.
* **Config Flow from Headers:** This `ContentModifier` step creates several header properties.  See the detailed property table below.  These properties are used to control the subsequent steps of the process.
* **Main Process:** Calls the `Main Process` (Process_82344329).
* **End:** The process terminates with an End Event.

### 3.2 Scheduler

* **Start Timer 1:** The process begins with a Timer Start Event, initiating the process based on a defined schedule.
* **Config Flow:** This `ContentModifier` step creates several constant and expression-based header properties. See the detailed property table below.  These properties are used to control the subsequent steps of the process.
* **Main Process:** Calls the `Main Process` (Process_82344329).
* **End 5:** The process terminates with an End Event.

### 3.3 Main Process

* **Start 6:** The process begins with a Start Event.
* **Get Employees from SFSF:** Calls the `Get Employees from SFSF` (Process_9431) process to retrieve employee data from SuccessFactors using a SuccessFactors adapter.  Two API calls exist, one to retrieve test employees (`Test Employees`) and one to retrieve all employees (`GetEmployees`).
* **Remove nodes without Person:** Removes XML nodes lacking a "Person" element.
* **Remove incomplete Employees:** Removes employee records with incomplete data.
* **Remove employees with company code C628 and country code not Norway:** Filters out employees based on specified criteria.
* **Remove unpaid leave employees:** Removes employees on unpaid leave.
* **Keep newest employment:**  Keeps only the most recent employment record for each employee.
* **Remove employment_information and job_information with lower fte:** Removes employment and job information where the FTE (full-time equivalent) is below a certain threshold.
* **Set Highest Direct Reports:** Sets the highest direct report for each employee.
* **Filter current nodes:** Filters the employee data.
* **Enrich from SFSF:** Calls the `Enrich from SFSF` (Process_9448) process to enrich the employee data with additional information from SuccessFactors via OData V2 calls to `FOCompany` and `EmpEmployment`.  Third-party system call using OData V2 protocol.
* **Get Vendors from SAP ERP:** Calls the `Get Vendors from SAP ERP` (Process_9472) process to retrieve vendor data from SAP ERP via a SOAP call. Third-party system call using SOAP protocol.
* **Employee SF to Concur:** Prepares the employee data for the next step.
* **XML to CSV:** Calls the `XML to CSV` (Process_82344268) process to convert the XML data to CSV format.
* **AddBOM:** Adds a Bill of Materials (BOM).
* **Router 1:** Routes the flow based on the `skipEncryption` property. If `true`, it skips encryption; otherwise, it proceeds with encryption.
* **Skip encryption for testing:** This step is executed if `skipEncryption` is true.
* **PGPEncryptor 1:** Encrypts the data using PGP encryption if `skipEncryption` is false.
* **Log Msg:** Logs a message.
* **Concur SFTP:** Sends the processed data (CSV) to Concur via SFTP. Third-party system call using SFTP protocol.
* **End 6:** The process terminates with an End Event.


### 3.4 XML to CSV

* **Start 4:** The process begins with a Start Event.
* **XML to CSV Header:** Processes the header section of the XML data.
* **XML to CSV Employee:** Processes the employee section of the XML data.
* **XML to CSV Delegate:** Processes the delegate section of the XML data. This step is conditionally skipped based on the 'SkipDelegates' property.
* **Set empty body:** Sets an empty message body.  This likely resets the payload for the subsequent 'Gather' step.
* **Gather 1:** Aggregates the processed header, employee, and delegate data.
* **Router 2:** Routes the flow based on the `SkipDelegates` property. If `true`, skips the delegate processing; otherwise, processes it.
* **End 4:** The process terminates with an End Event.

### 3.5 Get Vendors from SAP ERP

* **Start 3:** The process begins with a Start Event.
* **Remove XML Declaration:** Removes the XML declaration from the input.
* **Generate Request GetVendorId:** Generates a request for vendor IDs.
* **Save XML:** Saves XML data into a property called `employeeXML`.
* **GetVendors:** Retrieves vendor data from SAP ERP using a SOAP adapter. This is a third-party system call.
* **GetVendorMock:** Retrieves mock vendor data (likely for testing purposes) using a SOAP adapter.
* **Combine Vendors & Employees:** Combines the retrieved vendor data with employee data.
* **Content Modifier 1:** Modifies the combined data.
* **Merge Vendors & Employees:** Merges the vendor and employee data.
* **Filter:** Filters the merged data.
* **End 3:** The process terminates with an End Event.

### 3.6 Enrich from SFSF

* **Start 2:** The process begins with a Start Event.
* **Enrich FOCompany:** Enriches the data with information from the `FOCompany` resource in SuccessFactors using an OData V2 API call.  Third-party system call.
* **Enrich person_id_ext for relation:** Enriches the data with information from the `EmpEmployment` resource in SuccessFactors using an OData V2 API call. Third-party system call.
* **End 2:** The process terminates with an End Event.

### 3.7 Get Employees from SFSF

* **Start 1:** The process begins with a Start Event.
* **GetEmployees:** Retrieves employee data from SuccessFactors using a SuccessFactors adapter.  Third-party system call.
* **Test Employees:** Retrieves test employee data from SuccessFactors using a SuccessFactors adapter.  Third-party system call.
* **End 1:** The process terminates with an End Event.


## 4. Content Modifier Property Table

The following tables detail the properties set within the `ContentModifier` steps:

**4.1 Config Flow from Headers (Start from ProcessDirect)**

| Name                   | Action | Type    | Value             | Default | Datatype | Description                                      |
|------------------------|--------|---------|--------------------|---------|-----------|--------------------------------------------------|
| Concur_SFTP_Directory  | Create | header  | Concur_SFTP_Directory |          |           | SFTP directory for Concur                       |
| SkipDelegates          | Create | header  | SkipDelegates       |          |           | Flag to skip delegate processing                 |
| CashAdvanceAccountCode | Create | header  | CashAdvanceAccountCode|          |           | Cash advance account code                        |
| TestUsersSFSF          | Create | header  | TestUsersSFSF        |          |           | Flag for testing users in SuccessFactors        |
| CompanyListSFSF        | Create | header  | CompanyListSFSF      |          |           | List of companies in SuccessFactors              |
| InactiveDaysSF         | Create | header  | InactiveDaysSF       |          |           | Number of inactive days in SuccessFactors        |
| skipEncryption         | Create | header  | skipEncryption       |          |           | Flag to skip encryption                          |
| local_log              | Create | header  | local_log            |          |           | Flag to enable local logging                    |
| LogicalSystem          | Create | header  | LogicalSystem        |          |           | Logical system identifier                        |


**4.2 Config Flow (Scheduler)**

| Name                   | Action | Type      | Value                        | Default | Datatype | Description                                      |
|------------------------|--------|-----------|-----------------------------|---------|-----------|--------------------------------------------------|
| Concur_SFTP_Directory  | Create | constant  | {{Concur_SFTP_Directory}}    |          |           | SFTP directory for Concur                       |
| SkipDelegates          | Create | constant  | {{SkipDelegates}}            |          |           | Flag to skip delegate processing                 |
| CashAdvanceAccountCode | Create | constant  | {{CashAdvanceAccountCode}}   |          |           | Cash advance account code                        |
| TestUsersSFSF          | Create | expression | {{TestUsersSFSF}}           |          |           | Flag for testing users in SuccessFactors        |
| CompanyListSFSF        | Create | constant  | {{CompanyListSFSF}}          |          |           | List of companies in SuccessFactors              |
| InactiveDaysSF         | Create | constant  | {{InactiveDaysSF}}           |          |           | Number of inactive days in SuccessFactors        |
| skipEncryption         | Create | constant  | {{skip_encryption}}          |          |           | Flag to skip encryption                          |
| local_log              | Create | constant  | {{enable_log}}               |          |           | Flag to enable local logging                    |
| LogicalSystem          | Create | constant  | {{LogicalSystem}}            |          |           | Logical system identifier                        |


**4.3 Save XML (Get Vendors from SAP ERP)**

| Name          | Action | Type      | Value      | Default | Datatype | Description                               |
|---------------|--------|-----------|------------|---------|-----------|-------------------------------------------|
| employeeXML   | Create | expression | ${in.body} |          |           | Stores the input XML body                 |


This specification provides a comprehensive overview of the integration flows, enabling a new team member to understand the logic and functionality.  Remember to replace placeholder values like `{{Concur_SFTP_Directory}}` with their actual values.
