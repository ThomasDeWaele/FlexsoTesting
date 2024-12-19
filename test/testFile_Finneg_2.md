# Technical Specification: SAP Cloud Integration Flow

This document details the technical specifications for an SAP Cloud Integration (SCI) integration flow, based on the provided XML file. The flow orchestrates several OData calls to different entities within the `Biobest_Finneg.edmx` EDMX file, leveraging a main process (`Process_4`) to manage authentication and data processing.

## Overview

The integration flow consists of multiple subprocesses:

* **`Process_1` (Integration Process):** The main trigger process, initiated by a timer.  It calls the `Process_4` (Main Process).
* **`Process_4` (Main Process):** This process is the core logic, handling authentication, data retrieval from multiple OData services, and data transformation.
* **`Process_113` (VD_DK_CUSTSet OData Query):** Retrieves data from the `VD_DK_CUSTSet` entity.
* **`Process_129` (VA_DK_SalesFactSet OData query):** Retrieves data from the `VA_DK_SalesFactSet` entity.
* **`Process_140` (VD_DK_ItemSet OData query):** Retrieves data from the `VD_DK_ItemSet` entity.

Each subprocess except `Process_1` initiates an OData call to fetch data from the `Biobest_Finneg.edmx` EDMX file using basic authentication. `Process_1` is triggered by a timer. All subprocesses call the `Process_4` to handle the common logic.

## Detailed Process Description

### Process_1 (Integration Process)

1. **Start Timer 1:**  A timer event triggers the entire flow.
2. **Call local process:** Calls `Process_4` to execute the core logic.
3. **End:** The flow terminates after `Process_4` completes.


### Process_4 (Main Process)

This process is central to the integration and handles authentication and data retrieval from multiple OData services.

1. **Start 1:** The process starts.
2. **Set Secure param:** A `ContentModifier` step sets a secure parameter named "SecureParam" with a constant value "BioBest_ClientSecret". This parameter likely holds a client secret for authentication.  This is a crucial step for security, securely handling the client secret within SCI.
    * **Content Modifier Properties:**

    | Property Name | Action  | Type      | Value                     | Default | Datatype |
    |---------------|---------|-----------|--------------------------|---------|----------|
    | SecureParam   | Create  | constant  | BioBest_ClientSecret      |         |          |


3. **Get secure parameter:** This step retrieves the "BioBest_ClientSecret" parameter, presumably to use it for the token request.
4. **Get token:** This step makes a call to a third-party system (an OAuth2 token endpoint, inferred from the context) using the HTTP adapter. The "Receiver" parameter isn't fully defined but indicates a target endpoint for the token request.  The response (access token) is passed on.
    * **Third-party system call:** HTTP call to an OAuth2 token endpoint.
    * **Adapter:** HTTP
    * **Receiver:** Receiver (Endpoint needs to be defined separately)
5. **Store access token:**  A `ContentModifier` step stores the received access token in the message payload.
    * **Content Modifier Properties:**

    | Property Name | Action  | Type      | Value                     | Default | Datatype |
    |---------------|---------|-----------|--------------------------|---------|----------|
    | AccessToken   | Create  | expression | ${in.body}                |         | String   |


6. **Get Data:** This step makes another call to a third-party system (Likely an OData service based on the context) using the HTTP adapter and the access token obtained in the previous step.  "Receiver1" indicates a different target endpoint than the token endpoint.
    * **Third-party system call:** HTTP call to an OData service, likely needing authentication.
    * **Adapter:** HTTP
    * **Receiver:** Receiver1 (Endpoint needs to be defined separately)
7. **Test body:** This step likely performs a validation or transformation on the data received from "Get Data".
8. **Format JSON:** This step formats the data received from "Test body" into a JSON format.
9. **Set VD_DK_CUSTSet body:** This step prepares the message body for the `VD_DK_CUSTSet` OData call. This step's specifics aren't detailed in the XML.
10. **Set VA_DK_SalesFact body:** This step prepares the message body for the `VA_DK_SalesFactSet` OData call.  This step's specifics aren't detailed in the XML.
11. **Set VD_DK_Item body:** This step prepares the message body for the `VD_DK_ItemSet` OData call. This step's specifics aren't detailed in the XML.
12. **End 1:** The process terminates after completing data processing and preparation.


### Process_113 (VD_DK_CUSTSet OData Query)

1. **Start 2:**  The process starts.
2. **Call local process:** Calls `Process_4` to execute the core logic.  This implies the OData call to `VD_DK_CUSTSet` is handled within `Process_4`.
3. **End 2:** The process terminates.


### Process_129 (VA_DK_SalesFactSet OData query)

1. **Start 3:** The process starts.
2. **Call local process:** Calls `Process_4` to execute the core logic. This implies the OData call to `VA_DK_SalesFactSet` is handled within `Process_4`.
3. **End 4:** The process terminates.


### Process_140 (VD_DK_ItemSet OData query)

1. **Start 3:** The process starts.
2. **Call local process:** Calls `Process_4` to execute the core logic. This implies the OData call to `VD_DK_ItemSet` is handled within `Process_4`.
3. **End 4:** The process terminates.


## Missing Information

The provided XML lacks detailed information on how the message bodies are prepared (`Set VD_DK_CUSTSet body`, `Set VA_DK_SalesFact body`, `Set VD_DK_Item body`) and the specific endpoints for the HTTP adapter calls.  Further details are needed to complete the specification.  Also, error handling is not defined in this specification.


## Conclusion

This document provides a high-level technical specification of the SCI integration flow.  The core logic resides within `Process_4`, which handles authentication and data retrieval from multiple OData entities.  Further details on message mapping and error handling are required to complete the specification fully.
