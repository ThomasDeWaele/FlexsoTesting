# Technical Specification: SAP Cloud Integration Flows

This document details the technical specifications for the integration flows developed within SAP Cloud Integration, based on the provided XML file.  The specifications describe the flow logic, external system calls, and configuration details to enable a new team member to understand and maintain the system.

## Process Overview

The integration flow comprises several interconnected processes working together to extract employee data from SuccessFactors (SFSF), enrich it with data from SAP ERP, and finally transfer the processed data to Concur via SFTP.  The main processes are:

* **XML to CSV:** This process transforms XML data into CSV format.
* **Start from ProcessDirect:** This process initiates the main flow via a ProcessDirect trigger.
* **Scheduler:** This process triggers the main flow periodically using a timer.
* **Get Vendors from SAP ERP:** This process retrieves vendor information from SAP ERP.
* **Main Process:** This process orchestrates the entire data flow, including data transformation, enrichment, encryption, and SFTP transfer to Concur.
* **Enrich from SFSF:** This process enriches employee data retrieved from SFSF with additional information.
* **Get Employees from SFSF:** This process retrieves employee data from SFSF.


## Integration Flow Initiation

The integration flows can be initiated in two ways:

1. **ProcessDirect:** The `Start from ProcessDirect` process uses a `ProcessDirect` sender adapter named "ProcessDirect". This allows external systems to trigger the integration flow directly by sending a message to this adapter.

2. **Timer:** The `Scheduler` process uses a timer trigger (`Start Timer 1` step) to initiate the integration flow at scheduled intervals. This enables automated, recurring execution of the data processing and transfer.


## Detailed Process Description

### 1. Get Employees from SFSF (`Get Employees from SFSF` Process)

This process retrieves employee data from SuccessFactors.

* **Start 1:** The process starts with a timer event.
* **GetEmployees:**  This step makes a SOAP API call to the SuccessFactors system (`SFSF_SOAP`) to fetch employee data.  The API credentials are not explicitly mentioned in the provided XML, but assumed to be configured within the iFlow.
* **Test Employees:** This step makes an API call to SuccessFactors (`SFSF_Test`) for testing purposes; the purpose of this call is not detailed in the provided XML.
* **End 1:** The process ends after successfully retrieving employee data from either of the SuccessFactors endpoints.

### 2. Enrich from SFSF (`Enrich from SFSF` Process)

This process enriches the employee data received from SFSF.

* **Start 2:** The process begins with a start event.
* **Enrich FOCompany:** This step makes an API call (the specific API is not named) to enrich the data with FOCompany information.
* **Enrich person_id_ext for relation:** This step makes an API call (the specific API is not named) to enrich the data with `person_id_ext` for relationship purposes.
* **End 2:** The process ends after the enrichment process is complete.

### 3. Get Vendors from SAP ERP (`Get Vendors from SAP ERP` Process)

This process retrieves vendor data from SAP ERP.

* **Start 3:** The process starts with a start event.
* **Remove XML Declaration:** This step removes the XML declaration from the input XML.
* **Generate Request GetVendorId:** This step generates a request to get vendor IDs.  The details of the request generation are not specified.
* **GetVendors:** This step makes a SOAP API call to the SAP ERP system (`SAP_ERP`) using credentials defined as `{{ECC_CRED}}`.
* **GetVendorMock:**  This step provides a mock response to `GetVendors` presumably for testing purposes. The mock endpoint uses a SOAP adapter.
* **Filter:** This step filters the retrieved vendor data; criteria are not defined.
* **Combine Vendors & Employees:** This step combines the vendor data with the employee data.  The logic for combination isn't detailed.
* **Merge Vendors & Employees:** Merges vendors and employee data after filtering (the filtering criteria for this is not provided).
* **Content Modifier 1:** This step modifies the content using an unspecified expression.
* **Save XML:** This step saves the XML body into a property called `employeeXML`. The exact content of this property isn't detailed.
* **End 3:** The process ends after successfully retrieving and processing vendor data.


### 4. XML to CSV (`XML to CSV` Process)

This process converts XML data to CSV format.  The details of the conversion logic are not provided in the XML.

* **Start 4:** The process starts with a start event.
* **Set empty body:** This step sets an empty message body.  The reason for this is not detailed.
* **XML to CSV Header:** This step processes the header part of the XML data for CSV conversion.
* **XML to CSV Employee:** This step processes the employee part of the XML data for CSV conversion.
* **XML to CSV Delegate:** This step processes the delegate part of the XML data for CSV conversion.
* **Gather 1:** This step gathers the processed header, employee, and delegate parts of the XML data.
* **Router 2:** This router determines the processing path based on the `property.SkipDelegates` header property. If true, the flow skips the delegate processing. Otherwise, it processes the delegate data.
* **End 4:** The process ends after the CSV conversion.

### 5. Main Process (`Main Process` Process)

This process orchestrates the entire data flow.

* **Start 6:** The process starts with a start event.
* **Get Employees from SFSF:** This step calls the `Get Employees from SFSF` process.
* **Remove nodes without Person:** Removes nodes that lack the "Person" element.
* **Remove incomplete Employees:** Removes employee records considered incomplete; the criteria are not defined.
* **Remove unpaid leave employees:** Removes employees on unpaid leave.
* **Remove employees with company code C628 and country code not Norway:** Removes employees that meet this specific condition.
* **Keep newest employment:** Keeps only the most recent employment record for each employee.
* **Set Highest Direct Reports:** Sets the highest direct report for each employee. The method is not detailed.
* **Filter current nodes:** Filters the data; the criteria is not detailed.
* **Employee SF to Concur:** This step prepares the employee data for Concur.
* **Get Vendors from SAP ERP:** This step calls the `Get Vendors from SAP ERP` process.
* **XML to CSV:** This step calls the `XML to CSV` process.
* **AddBOM:** This step adds a bill of materials (BOM); specifics are not provided.
* **Router 1:** This router determines whether to skip encryption based on the `property.skipEncryption` property.
* **PGPEncryptor 1:** This step encrypts the data using PGP encryption if encryption is not skipped.
* **Skip encryption for testing:** This step is used for skipping the encryption.
* **Log Msg:** This step logs a message.
* **Concur SFTP:** This step sends the (potentially encrypted) data to Concur via SFTP. The API credentials are assumed to be configured within the iFlow.
* **End 6:** The process ends after the data is successfully transferred to Concur.

### 6. Start from ProcessDirect (`Start from ProcessDirect` Process)

This process initiates the main process via a ProcessDirect trigger.

* **Start 5:** The ProcessDirect trigger starts the process.
* **Config Flow from Headers:** This step uses a `ContentModifier` to create several header properties:  `Concur_SFTP_Directory`, `SkipDelegates`, `CashAdvanceAccountCode`, `TestUsersSFSF`, `CompanyListSFSF`, `InactiveDaysSF`, `skipEncryption`, `local_log`, and `LogicalSystem`.  The values of these properties are set as header values. This allows the use of dynamic data from header values to drive the iFlow execution.
* **Main Process:** This step calls the `Main Process`.
* **End:** The process ends after calling the main process.


### 7. Scheduler (`Scheduler` Process)

This process initiates the main process via a timer trigger.

* **Start Timer 1:** This step triggers the process based on the defined schedule.
* **Config Flow:** This step uses a `ContentModifier` to create header properties with constant or expression values: `Concur_SFTP_Directory`, `SkipDelegates`, `CashAdvanceAccountCode`, `TestUsersSFSF`, `CompanyListSFSF`, `InactiveDaysSF`, `skipEncryption`, `local_log`, and `LogicalSystem`.  The values are based on the variables `{{Concur_SFTP_Directory}}`, `{{SkipDelegates}}`, etc. This is another approach to use dynamic data, this time based on predefined variables.
* **Main Process:** This step calls the `Main Process`.
* **End 5:** The process ends after calling the main process.


## Content Modifier Details

The following table summarizes the properties set within the `ContentModifier` steps:

| Process Name             | Step Name                 | Property Name             | Action  | Type      | Value                               | Default | Datatype |
|--------------------------|---------------------------|--------------------------|---------|------------|------------------------------------|---------|-----------|
| Start from ProcessDirect | Config Flow from Headers | Concur_SFTP_Directory    | Create  | header    | Concur_SFTP_Directory            |         |           |
| Start from ProcessDirect | Config Flow from Headers | SkipDelegates             | Create  | header    | SkipDelegates                     |         |           |
| Start from ProcessDirect | Config Flow from Headers | CashAdvanceAccountCode   | Create  | header    | CashAdvanceAccountCode            |         |           |
| Start from ProcessDirect | Config Flow from Headers | TestUsersSFSF            | Create  | header    | TestUsersSFSF                     |         |           |
| Start from ProcessDirect | Config Flow from Headers | CompanyListSFSF          | Create  | header    | CompanyListSFSF                   |         |           |
| Start from ProcessDirect | Config Flow from Headers | InactiveDaysSF           | Create  | header    | InactiveDaysSF                    |         |           |
| Start from ProcessDirect | Config Flow from Headers | skipEncryption            | Create  | header    | skipEncryption                    |         |           |
| Start from ProcessDirect | Config Flow from Headers | local_log                 | Create  | header    | local_log                         |         |           |
| Start from ProcessDirect | Config Flow from Headers | LogicalSystem             | Create  | header    | LogicalSystem                      |         |           |
| Scheduler                 | Config Flow               | Concur_SFTP_Directory    | Create  | constant  | {{Concur_SFTP_Directory}}         |         |           |
| Scheduler                 | Config Flow               | SkipDelegates             | Create  | constant  | {{SkipDelegates}}                 |         |           |
| Scheduler                 | Config Flow               | CashAdvanceAccountCode   | Create  | constant  | {{CashAdvanceAccountCode}}         |         |           |
| Scheduler                 | Config Flow               | TestUsersSFSF            | Create  | expression| {{TestUsersSFSF}}                 |         |           |
| Scheduler                 | Config Flow               | CompanyListSFSF          | Create  | constant  | {{CompanyListSFSF}}                |         |           |
| Scheduler                 | Config Flow               | InactiveDaysSF           | Create  | constant  | {{InactiveDaysSF}}                |         |           |
| Scheduler                 | Config Flow               | skipEncryption            | Create  | constant  | {{skip_encryption}}               |         |           |
| Scheduler                 | Config Flow               | local_log                 | Create  | constant  | {{enable_log}}                    |         |           |
| Scheduler                 | Config Flow               | LogicalSystem             | Create  | constant  | {{LogicalSystem}}                 |         |           |
| Get Vendors from SAP ERP | Save XML                  | employeeXML               | Create  | expression| ${in.body}                         |         |           |


**Note:** Datatype is not specified in the XML input.  Default values are also not provided in the XML input file.


This detailed technical specification provides a comprehensive overview of the integration flows.  Further details on specific API calls, data transformations, and error handling would need to be gathered from the actual iFlow implementation within SAP Cloud Integration.
