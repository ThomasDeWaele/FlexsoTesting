# SAP Cloud Integration Technical Specification: Biobest Data Ingestion Flow

This document details the technical specifications for an SAP Cloud Integration (SCI) flow designed to ingest data from an Azure Blob Storage.  The flow retrieves files, decrypts them, and saves the output.  A third-party system, Azure Blob Storage, is accessed.


## 1. Flow Overview

The integration flow processes files from an Azure Blob Storage based on a timer trigger. It sorts filenames, retrieves a specific file, decrypts it using PGP, and then saves the decrypted output.

## 2. Steps and Detailed Explanation

The flow consists of the following steps, executed sequentially:

**2.1 Start Timer 1:**

* **Description:** This step initiates the flow at a predefined interval (not specified in the input XML).
* **Outgoing:** SequenceFlow_5 - This flow connects to the next step: "Set version".
* **Third-party interaction:** None.

**2.2 Set version:**

* **Description:**  This step sets several properties which are used to configure the connection to Azure Blob Storage and define the file retrieval criteria.  These properties are created using `ContentModifier`.
    * **Property: `container` (Type: constant, Value: `bbmabbmt`):** Specifies the container name in the Azure Blob Storage.
    * **Property: `folderPath` (Type: constant, Value: `.`):** Specifies the folder path within the container. A "." indicates the root folder.
    * **Property: `accountName` (Type: constant, Value: `datasphereblob`):**  Specifies the Azure storage account name.
    * **Property: `accountKeyAlias` (Type: constant, Value: `Biobest_Blob`):** Specifies the alias for the Azure storage account key, which is defined in the SCI's KeyStore.
    * **Property: `lastpoll` (Type: persisted variables, Value: `Fri, 01 Jan 2021 00:00:00 GMT`, Default: `Fri, 01 Jan 2021 00:00:00 GMT`):** This stores the timestamp of the last successful poll. This is a persisted variable, meaning its value is maintained across flow executions.  This is crucial for incremental processing; only files modified after this timestamp are processed.
    * **Property: `filenamePrefix` (Type: constant, Value: `*Biobest_SAGE100*`):** Specifies a prefix to filter files in Azure Blob Storage. This uses a wildcard (*) to match files starting with "Biobest_SAGE100".

* **Outgoing:** SequenceFlow_224 - This flow connects to the next step: "Set Azure Headers".
* **Third-party interaction:** None (sets properties; no external calls yet).

**2.3 Set Azure Headers:**

* **Description:**  This step likely sets HTTP headers required for authentication with Azure Blob Storage. The exact headers are not specified in the input XML but would include things like account key or SAS token.
* **Outgoing:** SequenceFlow_226 - This flow connects to the next step: "get list of files".
* **Incoming:** SequenceFlow_224.
* **Third-party interaction:** Indirectly prepares for interaction with Azure Blob Storage.

**2.4 get list of files:**

* **Description:** This step retrieves a list of files from the Azure Blob Storage using HTTP.  The filters defined in the "Set version" step are used here.
* **Outgoing:** SequenceFlow_229 - This flow connects to the next step: "sort filesnames in ascending order".
* **Incoming:** SequenceFlow_226.
* **Third-party interaction:** YES - Calls the Azure Blob Storage REST API (HTTP) to list files.
* **Adapter:** HTTP
* **Receiver:** Azure_BlobFileShare (This is the identifier for the Azure Blob Storage connection).

**2.5 sort filesnames in ascending order:**

* **Description:** This step sorts the list of files obtained in the previous step in ascending order based on filenames. The exact sorting algorithm is not specified.
* **Outgoing:** SequenceFlow_230 - This flow connects to the next step: "Set Filename".
* **Incoming:** SequenceFlow_229.
* **Third-party interaction:** None.

**2.6 Set Filename:**

* **Description:** This step sets the filename of the file to be retrieved.  In this case, a hardcoded filename is used.  This might be improved to dynamically select the next file from the sorted list.
* **Outgoing:** SequenceFlow_242 - This flow connects to the next step: "Set Azure Headers to get File".
* **Incoming:** SequenceFlow_230.
* **Third-party interaction:** None (sets properties; no external calls yet).
  * **Property: `fileName` (Type: constant, Value: `Biobest_SAGE100_09082024_1526.csv.pgp`):** This property specifies the name of the file to be downloaded from Azure Blob Storage.

**2.7 Set Azure Headers to get File:**

* **Description:** This step likely sets HTTP headers for the file download request to Azure Blob Storage.  Similar to step 2.3, authentication headers would be required.
* **Outgoing:** SequenceFlow_232 - This flow connects to the next step: "Get file".
* **Incoming:** SequenceFlow_242.
* **Third-party interaction:** Indirectly prepares for interaction with Azure Blob Storage.


**2.8 Get file:**

* **Description:** This step downloads the specified file from Azure Blob Storage.
* **Outgoing:** SequenceFlow_235 - This flow connects to the next step: "PGPDecryptor 1".
* **Incoming:** SequenceFlow_232.
* **Third-party interaction:** YES - Calls the Azure Blob Storage REST API (HTTP) to download the file.
* **Adapter:** HTTP
* **Receiver:** Azure_BlobFileShare (This is the identifier for the Azure Blob Storage connection).

**2.9 PGPDecryptor 1:**

* **Description:** This step decrypts the downloaded file using PGP decryption. This requires a PGP private key to be accessible to the SCI environment.  The specifics of the decryption process are not detailed here.
* **Outgoing:** SequenceFlow_244 - This flow connects to the next step: "Save Output".
* **Incoming:** SequenceFlow_235.
* **Third-party interaction:**  Potentially yes, depending on the PGP decryption implementation used (if an external library is called).

**2.10 Save Output:**

* **Description:** This step saves the decrypted output. The location is not explicitly defined in the input but needs to be configured.
* **Outgoing:** SequenceFlow_240 - This flow connects to the next step: "End".
* **Incoming:** SequenceFlow_244.
* **Third-party interaction:**  Potentially yes, depending on the save location (e.g., another storage service).
  * **Property: `test` (Type: constant, Value: `test`):**  The purpose of this property is unclear from the provided information. It might be a placeholder or a flag for logging purposes.


**2.11 End:**

* **Description:**  The end of the integration flow.
* **Incoming:** SequenceFlow_240.
* **Third-party interaction:** None.


## 3.  Missing Information and Recommendations

* **Error Handling:** The specification lacks details on error handling. Mechanisms for handling exceptions during file retrieval, decryption, and saving should be incorporated.
* **Retry Logic:**  Retry mechanisms for transient errors (e.g., network issues) during Azure Blob Storage calls are essential.
* **Logging:**  Detailed logging should be implemented to track the flow's execution and identify potential issues.
* **Security:**  Securely managing the Azure storage account key and PGP private key is crucial. Consider using secrets management features provided by SCI or an external secrets management solution.
* **Dynamic Filename Selection:** The current design uses a hardcoded filename.  It's highly recommended to modify the flow to dynamically select the next file to process from the sorted list, ensuring that only the newest, unprocessed files are handled.
* **Clarification on `test` property:** The purpose and usage of the `test` property in the "Save Output" step should be clarified.


This detailed specification provides a comprehensive overview of the Biobest data ingestion flow in SAP Cloud Integration.  Addressing the missing information and recommendations will enhance the robustness and maintainability of the solution.
