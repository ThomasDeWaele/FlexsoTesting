# Technical Specification: Integration Flow in SAP Cloud Integration

This document details the technical specifications for the integration flow developed in SAP Cloud Integration, encompassing several interconnected processes.

## Process Overview

The integration flow orchestrates data from SuccessFactors (SFSF), SAP ERP, and potentially a mock system, transforming and enriching it before transferring it to Concur via SFTP.  The flow consists of several subprocesses:

* **XML to CSV:** Transforms XML data into CSV format.
* **Start from ProcessDirect:** Initiates the main process via ProcessDirect.
* **Scheduler:** Initiates the main process via a timer.
* **Get Vendors from SAP ERP:** Retrieves vendor data from SAP ERP.
* **Main Process:** The core process encompassing data retrieval, transformation, enrichment, encryption (optional), and SFTP transfer.
* **Enrich from SFSF:** Enriches employee data from SFSF.
* **Get Employees from SFSF:** Retrieves employee data from SFSF.

## Integration Flow Initiation

The integration flow can be initiated in two ways:

1. **ProcessDirect:** The `Start from ProcessDirect` process uses a `ProcessDirect` sender adapter named "ProcessDirect" to trigger the main process. This allows external systems to initiate the integration flow directly.

2. **Timer:** The `Scheduler` process uses a timer event (`Start Timer 1`) to schedule periodic execution of the main process. This enables automated, recurring data synchronization.


## Detailed Process Description

### 1. Get Employees from SFSF (`Get Employees from SFSF` Process)

This process retrieves employee data from SuccessFactors.

* **Start 1:** The process starts with a timer event.
* **GetEmployees:** A SuccessFactors API call is made to the `SFSF_SOAP` receiver to retrieve employee data.  This step uses the SuccessFactors adapter.
* **Test Employees:** A SuccessFactors API call is made to the `SFSF_Test` receiver. The purpose of this call is not explicitly stated in the XML but likely for testing purposes. This step also uses the SuccessFactors adapter.
* **End 1:** The process ends after successful execution of API calls.

### 2. Enrich from SFSF (`Enrich from SFSF` Process)

This process enriches the employee data retrieved from SuccessFactors.

* **Start 2:** The process starts with a start event.
* **Enrich FOCompany:** An API call (details not specified in XML) is made to enrich the data with `FOCompany` information. The adapter type is not specified.
* **Enrich person_id_ext for relation:** An API call is made to enrich the data with  `person_id_ext` information for establishing relationships. The adapter type is not specified.
* **End 2:** The process ends after successful enrichment.

### 3. Get Vendors from SAP ERP (`Get Vendors from SAP ERP` Process)

This process retrieves vendor data from SAP ERP.

* **Start 3:** The process begins with a start event.
* **Remove XML Declaration:** Removes the XML declaration from the input XML.  This is a pre-processing step.
* **Generate Request GetVendorId:** Generates a request to retrieve Vendor IDs. This step likely involves transforming the data to a format suitable for the SAP ERP API call.
* **Save XML:**  Saves the input XML to a variable named `employeeXML`. This is a content modifier step.
  * **ContentModifier Properties:**
    | Name          | Action | Type     | Value             | Default | Datatype |
    |-----------------|--------|----------|--------------------|---------|-----------|
    | employeeXML    | Create  | expression | `${in.body}`       |         |           |


* **GetVendors:** A SOAP API call is made to the `SAP_ERP` receiver to retrieve vendor data.  This step uses the SOAP adapter. Uses API credential `{{ECC_CRED}}`.
* **GetVendorMock:** A SOAP API call is made to a `Mock` receiver – likely for testing purposes. This step uses the SOAP adapter.
* **Combine Vendors & Employees:** Combines the retrieved vendor data with the employee data.  This step likely involves joining data based on a common key.
* **Filter:** Filters the combined data based on unspecified criteria.
* **Merge Vendors & Employees:** Merges the employee and vendor data.  The specific merging logic isn't defined in the XML.
* **End 3:** The process ends after successful retrieval and merging of data.


### 4. XML to CSV (`XML to CSV` Process)

This process transforms XML data into CSV format.

* **Start 4:** The process starts with a start event.
* **XML to CSV Header:** Transforms the header section of the XML data into CSV format. This involves extracting header information and structuring it appropriately for CSV.
* **XML to CSV Employee:** Transforms the employee data section of the XML into CSV format.  This likely involves field mapping and data transformation.
* **XML to CSV Delegate:** Transforms the delegate data section of the XML into CSV format. This likely involves field mapping and data transformation.
* **Set empty body:** Sets the message body to empty. The purpose is not explicitly stated in the XML but likely for preparing the message before the next step.
* **Gather 1:** Gathers the transformed CSV data from the previous steps (`XML to CSV Header`, `XML to CSV Employee`, and `XML to CSV Delegate`) into a single message.
* **End 4:** The process ends after successful CSV transformation.


### 5. Main Process (`Main Process` Process)

This process is the core of the integration flow.

* **Start 6:** The process begins with a start event.
* **Get Employees from SFSF:** Calls the `Get Employees from SFSF` subprocess to retrieve employee data from SFSF.
* **Remove nodes without Person:** Removes nodes from the XML that do not contain "Person" data.
* **Filter current nodes:** Filters the remaining XML nodes based on unspecified criteria.
* **Keep newest employment:** Selects only the newest employment record for each employee.
* **Remove employees with company code C628 and country code not Norway:** Filters out employees based on company code and country code.
* **Remove incomplete Employees:** Removes employees with incomplete data.
* **Remove unpaid leave employees:** Removes employees on unpaid leave.
* **Remove employment_information and job_information with lower fte:** Removes employment and job information for employees with lower FTE (full-time equivalent).
* **Set Highest Direct Reports:** Sets the highest direct reports for each employee.
* **Employee SF to Concur:** Prepares the employee data for sending to Concur. This step involves data transformation and potentially formatting.
* **Get Vendors from SAP ERP:** Calls the `Get Vendors from SAP ERP` subprocess to retrieve vendor data from SAP ERP.
* **XML to CSV:** Calls the `XML to CSV` subprocess to transform the prepared data into CSV format.
* **AddBOM:** Adds a Bill of Materials (BOM) - the purpose is not explicitly stated but likely for enriching the data.
* **Router 1:** Routes the message based on the value of the `skipEncryption` property.
    * If `${property.skipEncryption} = 'true'`, it routes to `Skip encryption for testing`.
    * Otherwise, it routes to `PGPEncryptor 1`.
* **PGPEncryptor 1:** Encrypts the message using PGP encryption.
* **Skip encryption for testing:** Skips encryption (for testing purposes).
* **Log Msg:** Logs a message.  The specific log message is not detailed.
* **Concur SFTP:** Transfers the processed data to Concur via SFTP. This step utilizes the SFTP adapter and targets the `Concur_SFTP` receiver.
* **End 6:** The process ends after successful data transfer.


### 6. Start from ProcessDirect (`Start from ProcessDirect` Process)

This process initiates the Main Process via ProcessDirect.

* **Start 5:** The process begins with a start event triggered by the ProcessDirect adapter.
* **Config Flow from Headers:** Sets several header properties using a content modifier.
    * **ContentModifier Properties:**
      | Name                    | Action | Type   | Value                 | Default | Datatype |
      |-------------------------|--------|--------|-----------------------|---------|-----------|
      | Concur_SFTP_Directory   | Create  | header | Concur_SFTP_Directory |         |           |
      | SkipDelegates           | Create  | header | SkipDelegates           |         |           |
      | CashAdvanceAccountCode  | Create  | header | CashAdvanceAccountCode  |         |           |
      | TestUsersSFSF           | Create  | header | TestUsersSFSF           |         |           |
      | CompanyListSFSF         | Create  | header | CompanyListSFSF         |         |           |
      | InactiveDaysSF          | Create  | header | InactiveDaysSF          |         |           |
      | skipEncryption          | Create  | header | skipEncryption          |         |           |
      | local_log               | Create  | header | local_log               |         |           |
      | LogicalSystem           | Create  | header | LogicalSystem           |         |           |

* **Main Process:** Calls the `Main Process` subprocess.
* **End:** The process ends after the successful completion of the `Main Process`.


### 7. Scheduler (`Scheduler` Process)

This process initiates the Main Process via a timer.

* **Start Timer 1:** The process starts with a timer event.  The specific timer configuration (frequency, etc.) is not specified.
* **Config Flow:** This step sets header properties using a Content Modifier.  The values are defined as constants or expressions. Note that some constants use double curly braces `{{...}}`, indicating potential usage of mapping variables or expressions.
    * **ContentModifier Properties:**
      | Name                    | Action | Type      | Value                       | Default | Datatype |
      |-------------------------|--------|-----------|----------------------------|---------|-----------|
      | Concur_SFTP_Directory   | Create  | constant  | {{Concur_SFTP_Directory}}   |         |           |
      | SkipDelegates           | Create  | constant  | {{SkipDelegates}}           |         |           |
      | CashAdvanceAccountCode  | Create  | constant  | {{CashAdvanceAccountCode}}  |         |           |
      | TestUsersSFSF           | Create  | expression | {{TestUsersSFSF}}           |         |           |
      | CompanyListSFSF         | Create  | constant  | {{CompanyListSFSF}}         |         |           |
      | InactiveDaysSF          | Create  | constant  | {{InactiveDaysSF}}          |         |           |
      | skipEncryption          | Create  | constant  | {{skip_encryption}}          |         |           |
      | local_log               | Create  | constant  | {{enable_log}}              |         |           |
      | LogicalSystem           | Create  | constant  | {{LogicalSystem}}           |         |           |
* **Main Process:** Calls the `Main Process` subprocess.
* **End 5:** The process ends after the successful completion of the `Main Process`.


This detailed specification provides a comprehensive overview of the integration flow, allowing a new team member to understand the logic and functionality of each process and step.  Further clarification might be needed regarding the exact details of data transformations within certain steps, which are not fully explicit in the provided XML.
