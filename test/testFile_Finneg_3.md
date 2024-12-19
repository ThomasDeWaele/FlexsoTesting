# SAP Cloud Integration Technical Specification: Data Integration Flow

This document details the technical specifications for the data integration flow implemented in SAP Cloud Integration.  The flow consists of several interconnected processes, each with specific steps.  A third-party system is called via HTTP API calls.

## Process Overview

The integration flow orchestrates data retrieval from an OData service and subsequent processing.  The main flow is triggered by three independent OData queries and a timer. All OData queries eventually call a central "Main process" for token management and data formatting.

**Processes:**

1. **VA_DK_SalesFactSet OData query:** Retrieves data from the `VA_DK_SalesFactSet` entity set.
2. **VD_DK_CUSTSet OData Query:** Retrieves data from the `VD_DK_CUSTSet` entity set.
3. **VD_DK_ItemSet OData query:** Retrieves data from the `VD_DK_ItemSet` entity set.
4. **Integration Process:**  A timer-triggered process that calls the Main process.
5. **Main process:** The central process responsible for token retrieval, data processing, and formatting.


## Detailed Process Descriptions

### 1. VA_DK_SalesFactSet OData query

* **Sender Adapter:** OData Sender
    * **Name:** OData
    * **Entity Set:** `VA_DK_SalesFactSet`
    * **EDMX Path:** `/edmx/Biobest_Finneg.edmx`
    * **Authentication:** Basic Authentication
* **Process Steps:**
    * **Start 3:** Initiates the process via an OData request.
    * **Call main process:** Calls the `Main process` to perform data processing and formatting.
    * **End 4:** Terminates the process.

### 2. VD_DK_CUSTSet OData Query

* **Sender Adapter:** OData Sender
    * **Name:** OData
    * **Entity Set:** `VD_DK_CUSTSet`
    * **EDMX Path:** `/edmx/Biobest_Finneg.edmx`
    * **Authentication:** Basic Authentication
* **Process Steps:**
    * **Start 2:** Initiates the process via an OData request.
    * **Call local process:** Calls the `Main process` to perform data processing and formatting.
    * **End 2:** Terminates the process.

### 3. VD_DK_ItemSet OData query

* **Sender Adapter:** OData Sender
    * **Name:** OData
    * **Entity Set:** `VD_DK_ItemSet`
    * **EDMX Path:** `/edmx/Biobest_Finneg.edmx`
    * **Authentication:** Basic Authentication
* **Process Steps:**
    * **Start 3:** Initiates the process via an OData request.
    * **Call main process:** Calls the `Main process` to perform data processing and formatting.
    * **End 4:** Terminates the process.

### 4. Integration Process

* **Sender Adapter:** Timer. This process is triggered periodically by a timer.
* **Process Steps:**
    * **Start Timer 1:** Starts the process based on a timer schedule.
    * **Call local process:** Calls the `Main process`.
    * **End:** Terminates the process.


### 5. Main process

This process is the core of the integration, handling token retrieval and data manipulation.

* **Process Steps:**
    * **Start 1:**  Initiates the main process.
    * **Set Secure param:**  Sets a secure parameter named "SecureParam" with a constant value "BioBest_ClientSecret". This is likely used for authentication.
        * **ContentModifier Properties:**
            | Property Name | Action | Type     | Value                  | Default | Datatype |
            |--------------|--------|----------|-----------------------|---------|----------|
            | SecureParam   | Create  | constant | BioBest_ClientSecret   |         |          |

    * **Get token:** Calls an HTTP API to retrieve an access token.
        * **Adapter:** HTTP
        * **Receiver:** Receiver  (This likely represents the API endpoint for token retrieval.)

    * **Store access token:** Stores the received access token in the message body using an expression.
        * **ContentModifier Properties:**
            | Property Name | Action | Type    | Value                  | Default | Datatype |
            |--------------|--------|---------|-----------------------|---------|----------|
            | AccessToken   | Create  | expression | `${in.body}`          |         | String   |


    * **Get Data:** Calls an HTTP API to retrieve data (likely using the previously obtained access token).
        * **Adapter:** HTTP
        * **Receiver:** Receiver1 (This likely represents the API endpoint for data retrieval.)


    * **Set VA_DK_SalesFact body:**  Prepares the message body with data received from `VA_DK_SalesFactSet` OData query.

    * **Set VD_DK_Item body:**  Prepares the message body with data received from `VD_DK_ItemSet` OData query.

    * **Set VD_DK_CUSTSet body:**  Prepares the message body with data received from `VD_DK_CUSTSet` OData query.

    * **Format JSON:** Formats the data into a JSON structure.

    * **Test body:**  Performs a test on the formatted JSON body (purpose not explicitly stated).

    * **End 1:** Terminates the process.


## Integration Flow Initiation

The integration flow can be started in four ways:

1. **VA_DK_SalesFactSet OData query:** An OData request to the `VA_DK_SalesFactSet` entity set.
2. **VD_DK_CUSTSet OData Query:** An OData request to the `VD_DK_CUSTSet` entity set.
3. **VD_DK_ItemSet OData query:** An OData request to the `VD_DK_ItemSet` entity set.
4. **Integration Process:** A scheduled timer event.

## Third-Party System Calls

Two third-party system calls are made within the `Main process`:

1. **Get token:** Retrieves an access token via an HTTP call.
2. **Get Data:** Retrieves data via an HTTP call (this likely requires the access token).


This specification provides a detailed overview of the integration flow.  Further details may be required for specific implementation aspects.  Missing information includes the exact structure of the JSON being formatted, the purpose of the "Test body" step, and the specifics of the HTTP API calls (endpoints, request/response structure, error handling).
