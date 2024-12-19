# Technical Specification: SAP Cloud Integration Flow - Azure Blob File Processing

This document details the technical specification for an integration flow in SAP Cloud Integration (SCI) designed to process files from an Azure Blob storage.  The flow retrieves files sequentially, decrypts them (assuming PGP encryption), and saves the decrypted output.

**1. Overview**

The integration flow orchestrates the retrieval, decryption, and saving of files from an Azure Blob storage. It utilizes several iFlows steps including timers, HTTP receivers for Azure interaction, and custom processing steps.  The flow operates sequentially, processing one file at a time.

**2. Flow Diagram**

```
[Start Timer 1] --> [Set Version] --> [Get List of Files (HTTP - Azure)] --> [Sort Filenames] --> [Set Filename] --> [Set Azure Headers to get File] --> [Get File (HTTP - Azure)] --> [PGPDecryptor] --> [Save Output] --> [End]
```

**3. Detailed Step-by-Step Specification**

Each step in the integration flow is detailed below:


| Step Name                     | Incoming Sequence Flow | Outgoing Sequence Flow | Description                                                                                                                                                             | 3rd Party System Call | Properties Set                                      | Explanation of Properties                                                                          |
|---------------------------------|-------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------|----------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| **Start Timer 1**              | -                        | SequenceFlow_5          | Initiates the flow using a timer.  This could be configured to run at specific intervals (e.g., every 5 minutes).                                                        | No                       | -                                                  | Triggers the start of the process based on a defined schedule.                                      |
| **Set Version**                | SequenceFlow_5           | SequenceFlow_224         | Sets a version property (likely for tracking purposes).  The exact nature of the version property needs further clarification.                                          | No                       | `version` property (Value needs clarification)         |  Stores a version number, potentially tied to the processed files or the flow execution itself.    |
| **Get List of Files**          | SequenceFlow_224         | SequenceFlow_229         | Retrieves a list of files from the Azure Blob storage using an HTTP adapter.                                                                                             | **Yes (Azure)**       |  Authentication Headers (Azure Storage Account Key/SAS), container name, potential file filter | Sends a request to Azure to fetch the list of files in a specified container; Requires proper authentication. |
| **Sort Filenames**             | SequenceFlow_229         | SequenceFlow_230         | Sorts the list of filenames received from Azure in ascending order.  This step likely involves a Groovy script or similar transformation within SCI.                         | No                       | -                                                  | Ensures files are processed in a defined order (e.g., chronological).                               |
| **Set Filename**                | SequenceFlow_230         | SequenceFlow_242         | Sets the filename property for subsequent steps, using the sorted list. This prepares the flow for the retrieval of the first file in the sorted list.                               | No                       | `filename` property (taken from sorted list)          |  Selects the next file from the sorted list for processing.                                        |
| **Set Azure Headers to get File** | SequenceFlow_242         | SequenceFlow_232         | Sets the necessary HTTP headers for retrieving a specific file from Azure Blob storage, using the filename set in the previous step.                                  | No                       |  Authentication Headers (Azure Storage Account Key/SAS),  `x-ms-range` (potentially), `filename` header,  |  Configures the HTTP request to download a particular file from the Azure container; Includes authentication and potentially range headers for partial downloads. |
| **Get File**                   | SequenceFlow_232         | SequenceFlow_235         | Retrieves a single file from Azure Blob storage using an HTTP adapter based on the set headers.                                                                                  | **Yes (Azure)**       | -                                                  | Downloads a specific file from Azure based on the provided filename and headers.                      |
| **PGPDecryptor 1**             | SequenceFlow_235         | SequenceFlow_244         | Decrypts the file using a PGP decryption library/service. This likely requires a custom extension or an external service call.                                                | **Yes (PGP Decryption Service)** | -                                                  |  Performs PGP decryption on the retrieved file.  The service details need further clarification.      |
| **Save Output**                 | SequenceFlow_244         | SequenceFlow_240         | Saves the decrypted file to a target location (needs further clarification - potentially another storage location or database).                                                | No                       | -                                                  |  Persists the decrypted file to a specified destination.                                          |
| **End**                         | SequenceFlow_240         | -                        | Terminates the flow.                                                                                                                                                      | No                       | -                                                  |  Signals the completion of the process.                                                          |


**4. External System Integration Details**

* **Azure Blob Storage:**  Access is achieved using HTTP calls, requiring proper authentication (Storage Account Key or Shared Access Signature - SAS).  Details of the storage account, container name, and any file filtering need to be configured within the iFlow.
* **PGP Decryption Service:** Requires integration with a PGP decryption service.  Details regarding the service endpoint, authentication, and any required parameters need further clarification.

**5.  Assumptions**

* The PGP decryption step utilizes a suitable library or service. The integration method needs to be defined.
*  The 'Set Version' step sets a relevant version number; the exact implementation needs clarification.
* The 'Save Output' step requires specification of the target location.

**6. Open Issues**

* Clarification is needed on the specific versioning scheme.
* Details on the PGP decryption service and integration mechanism are required.
* The target location for the saved output needs to be defined.
* Error handling and exception management need to be implemented.


This detailed specification provides a foundation for developing and maintaining the SAP Cloud Integration flow.  Further clarification on open issues is necessary before full implementation can commence.
