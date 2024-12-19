# Technical Specification: SAP Cloud Integration Flow

This document details the technical specifications for the SAP Cloud Integration flow, encompassing four processes: `VA_DK_SalesFactSet OData query`, `VD_DK_CUSTSet OData Query`, `VD_DK_ItemSet OData query`, and `Main process`.

## Process Overview

The integration flow orchestrates data retrieval from various sources and ultimately updates a target system (implied, not explicitly stated in the provided XML).  It leverages OData for initial data requests and HTTP calls to a third-party API (`https://api.teamplace.finneg.com` and `https://api.finneg.com`).  The `Main process` acts as a central orchestrator, handling authentication, data transformation, and routing based on incoming data.

Three separate processes initiate the overall flow via OData calls targeting different entity sets:

* **`VA_DK_SalesFactSet OData query`**: Initiates the flow to retrieve data from the `VA_DK_SalesFactSet` entity.
* **`VD_DK_CUSTSet OData Query`**: Initiates the flow to retrieve data from the `VD_DK_CUSTSet` entity.
* **`VD_DK_ItemSet OData query`**: Initiates the flow to retrieve data from the `VD_DK_ItemSet` entity.

A fourth process, `Integration Process`, provides a scheduled start mechanism (using a Timer).  All three OData query processes and the timer-based process ultimately invoke the `Main process`.


## Integration Flow Initiation

The integration flow can be initiated in four ways, each using a different `SenderAdapter`:

1. **OData (VA_DK_SalesFactSet):**  Triggered by an incoming OData request to the `VA_DK_SalesFactSet` entity set within the `/edmx/Biobest_Finneg.edmx` EDMX file. Uses basic authentication.
2. **OData (VD_DK_CUSTSet):** Triggered by an incoming OData request to the `VD_DK_CUSTSet` entity set within the `/edmx/Biobest_Finneg.edmx` EDMX file. Uses basic authentication.
3. **OData (VD_DK_ItemSet):** Triggered by an incoming OData request to the `VD_DK_ItemSet` entity set within the `/edmx/Biobest_Finneg.edmx` EDMX file. Uses basic authentication.
4. **Timer:** The `Integration Process` starts periodically based on a configured timer schedule. This is a scheduled, internal trigger.


## Detailed Process Description

### `Main process`

This process is the core of the integration flow, called by all other processes.

**Steps:**

1. **Start 1:**  The starting point of the `Main process`.

2. **Set Secure param:**
    * **Type:** `ContentModifier`
    * **Incoming:** `SequenceFlow_7`
    * **Outgoing:** `SequenceFlow_95`
    * **Content Modifier Properties:**
        | Property Name | Action | Type     | Value                 | Default | Datatype |
        |---------------|--------|----------|-----------------------|---------|----------|
        | SecureParam   | Create  | constant | BioBest_ClientSecret |         |          |

3. **Get secure parameter:** Retrieves the secure parameter (`BioBest_ClientSecret`, likely a Client Secret) – This step's functionality is not explicitly defined but is implied by its position in the flow.

4. **Get token:**
    * **Type:** `API_CALL` (HTTP Adapter)
    * **Incoming:** `SequenceFlow_19`
    * **Outgoing:** `SequenceFlow_13`
    * **API Endpoint:** `https://api.teamplace.finneg.com/api/oauth/token`
    * **API Endpoint Query:** `grant_type=client_credentials&client_id=033169c62d4e4bc6b7356fbdbbdbede8&client_secret=${property.P_Password}`  This is a 3rd party HTTP call to obtain an access token using client credentials. The Client Secret is likely obtained in step 2.  `P_Password`  is assumed to be another secure parameter that is being passed to the step.

5. **Store access token:**
    * **Type:** `ContentModifier`
    * **Incoming:** `SequenceFlow_13`
    * **Outgoing:** `SequenceFlow_17`
    * **Content Modifier Properties:**
        | Property Name | Action | Type    | Value             | Default | Datatype |
        |---------------|--------|---------|--------------------|---------|----------|
        | AccessToken   | Create  | expression | `${in.body}`      |         | String   |

6. **Get Data:**
    * **Type:** `API_CALL` (HTTP Adapter)
    * **Incoming:** `SequenceFlow_17`
    * **Outgoing:** `SequenceFlow_97`
    * **API Endpoint:** `https://api.finneg.com/api/reports/ANALISISVENTASBBAPI`
    * **API Endpoint Query:** `ACCESS_TOKEN=${property.AccessToken}&PARAMWEBREPORT_FechaDesde=2023-01-01&PARAMWEBREPORT_FechaHasta=2024-12-30&PARAMWEBREPORT_Moneda=DOL&PARAMWEBREPORT_Empresa=BIOBEST36` This is a 3rd party HTTP call to retrieve data; the `AccessToken` obtained in step 5 is used for authentication.

7. **Test body:** Performs an unspecified test on the data received from the external API call.

8. **Format JSON:**  Formats the data received from the external call into JSON.

9. **Is Which entity:**
    * **Type:** `Router`
    * **Incoming:** `SequenceFlow_110`
    * **Outgoing:**  The router routes the message based on the value of `${property.SAP_ODataEntityName}`:
        * `VD_DK_CUSTSet`: Routes to `Set VD_DK_CUSTSet body`
        * `VA_DK_SalesFactSet`: Routes to `Set VA_DK_SalesFact body`
        * `VD_DK_ItemSet`: Routes to `Set VD_DK_Item body`
        * Default: Routes to `End 3`

10. **Set VD_DK_CUSTSet body**, **Set VA_DK_SalesFact body**, **Set VD_DK_Item body**: These steps prepare the data for the implied output/update step.  The exact transformation is undefined.

11. **End 1**, **End 3**: Termination points of the `Main process`.


### Other Processes (`VA_DK_SalesFactSet OData query`, `VD_DK_CUSTSet OData Query`, `VD_DK_ItemSet OData query`, `Integration Process`)

These processes are simpler, serving primarily as entry points that invoke the `Main process`.  Their internal logic is minimal: they receive an OData request or timer trigger and subsequently call the `Main process`.  No transformations or external calls happen within these processes.


## Conclusion

This integration flow uses a central orchestrator (`Main process`) to manage the retrieval and transformation of data from various sources. The use of external API calls requires careful attention to authentication and error handling. The unspecified transformations within the steps require further clarification for complete implementation.  The lack of information regarding the target system prevents a complete specification of the overall data flow.
