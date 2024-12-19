# SAP Cloud Integration Flow Technical Specification

## Process Overview

This document details the technical specifications for an integration flow in SAP Cloud Integration. The flow consists of multiple processes, orchestrating data retrieval and processing from various sources, including external APIs and SAP OData services.  The core process, "Main process," acts as a central hub, coordinating the data flow and interactions with external systems.  Other processes initiate OData queries for specific entities.


## Integration Flow Initiation

The integration flow can be initiated via three different sender adapters:

1. **`VA_DK_SalesFactSet OData query` Process:** Initiated by an OData sender listening for requests to the `VA_DK_SalesFactSet` entity set within the `/edmx/Biobest_Finneg.edmx` service.  Authentication uses basic authentication.

2. **`VD_DK_CUSTSet OData query` Process:** Initiated by an OData sender listening for requests to the `VD_DK_CUSTSet` entity set within the `/edmx/Biobest_Finneg.edmx` service. Authentication uses basic authentication.

3. **`VD_DK_ItemSet OData query` Process:** Initiated by an OData sender listening for requests to the `VD_DK_ItemSet` entity set within the `/edmx/Biobest_Finneg.edmx` service. Authentication uses basic authentication.

4. **`Integration Process` Process:** Initiated by a Timer Start Event, triggering the process at a predefined schedule.


## Detailed Process Description

This section describes each process and its steps in detail.

### 1. `VA_DK_SalesFactSet OData query` Process

This process simply triggers the "Main process" after receiving an OData request.

* **Start 3:** OData sender initiating the process.  Retrieves data from the `VA_DK_SalesFactSet` entity set. Uses Basic Authentication.
* **Call main process:** Calls the "Main process" to process the retrieved data.
* **End 4:** Terminates the process.

### 2. `VD_DK_CUSTSet OData query` Process

This process also triggers the "Main process" after receiving an OData request.

* **Start 2:** OData sender initiating the process.  Retrieves data from the `VD_DK_CUSTSet` entity set. Uses Basic Authentication.
* **Call local process:** Calls the "Main process" to process the retrieved data.
* **End 2:** Terminates the process.

### 3. `VD_DK_ItemSet OData query` Process

This process triggers the "Main process" after receiving an OData request.

* **Start 3:** OData sender initiating the process.  Retrieves data from the `VD_DK_ItemSet` entity set. Uses Basic Authentication.
* **Call main process:** Calls the "Main process" to process the retrieved data.
* **End 4:** Terminates the process.

### 4. `Integration Process` Process

This process uses a timer to initiate the "Main process" periodically.

* **Start Timer 1:** Timer event triggering the process.
* **Call local process:** Calls the "Main process."
* **End:** Terminates the process.


### 5. `Main process` Process

This is the central process orchestrating data retrieval and transformation.

* **Start 1:** Start event initiating the process.
* **Set Secure param:**  `ContentModifier` step.
    * **Property Table:**
        | Property Name | Action | Type     | Value                 | Default | Datatype |
        |---------------|--------|----------|-----------------------|---------|----------|
        | SecureParam   | Create  | constant | BioBest_ClientSecret |         |          |

* **Get secure parameter:** Retrieves a secure parameter (likely the client secret).  The source of this secure parameter is not explicitly defined in the provided XML.

* **Get token:**  An external HTTP call (API_CALL) to retrieve an access token.
    * **Adapter:** HTTP
    * **Receiver:** Receiver
    * **API Endpoint:** `https://api.teamplace.finneg.com/api/oauth/token`
    * **API Endpoint Query:** `grant_type=client_credentials&client_id=033169c62d4e4bc6b7356fbdbbdbede8&client_secret=${property.P_Password}`  This uses a property `P_Password` containing the client secret.

* **Store access token:** `ContentModifier` step. Stores the access token received from the previous step.
    * **Property Table:**
        | Property Name | Action | Type    | Value           | Default | Datatype |
        |---------------|--------|---------|-----------------|---------|----------|
        | AccessToken   | Create  | expression | ${in.body}     |         | String   |

* **Get Data:** An external HTTP call (API_CALL) to fetch data.
    * **Adapter:** HTTP
    * **Receiver:** Receiver1
    * **API Endpoint:** `https://api.finneg.com/api/reports/ANALISISVENTASBBAPI`
    * **API Endpoint Query:** `ACCESS_TOKEN=${property.AccessToken}&PARAMWEBREPORT_FechaDesde=2023-01-01&PARAMWEBREPORT_FechaHasta=2024-12-30&PARAMWEBREPORT_Moneda=DOL&PARAMWEBREPORT_Empresa=BIOBEST36`  This uses the AccessToken stored earlier.

* **Test body:** A processing step (type not specified) to potentially validate or transform the data received from `Get Data`.

* **Format JSON:** A processing step (type not specified) to format the received data as JSON.

* **Set VA_DK_SalesFact body:** A processing step (type not specified) that likely prepares the data for the `VA_DK_SalesFactSet` OData update.

* **Set VD_DK_CUSTSet body:** A processing step (type not specified) that likely prepares the data for the `VD_DK_CUSTSet` OData update.

* **Set VD_DK_Item body:** A processing step (type not specified) that likely prepares the data for the `VD_DK_ItemSet` OData update.

* **End 1:** Terminates the process.  This step has multiple incoming sequences, suggesting parallel processing of different data sets before termination.


This specification provides a comprehensive overview of the integration flow.  Further details about the unspecified processing steps ("Set VA_DK_SalesFact body," "Set VD_DK_CUSTSet body," "Set VD_DK_Item body,"  "Test body," "Format JSON") would require examination of the actual implementation within SAP Cloud Integration.  Additionally, the source of `P_Password` needs to be clarified for security and maintainability purposes.
