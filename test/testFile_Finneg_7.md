# Technical Specification: SAP Cloud Integration Flow

This document details the technical specification for the integration flow in SAP Cloud Integration, encompassing all processes and their individual steps.


## Process Overview

The integration flow consists of five main processes:

1. **VA_DK_SalesFactSet OData query:**  Initiates an OData query against the `VA_DK_SalesFactSet` entity and calls the `Main process` for further processing.
2. **VD_DK_CUSTSet OData Query:** Initiates an OData query against the `VD_DK_CUSTSet` entity and calls the `Main process` for further processing.
3. **VD_DK_ItemSet OData query:** Initiates an OData query against the `VD_DK_ItemSet` entity and calls the `Main process` for further processing.
4. **Integration Process:** A timer-triggered process that initiates the `Main process`.
5. **Main process:** The central process responsible for retrieving an access token, fetching data from an external API, and routing the data based on the OData entity name before updating corresponding OData entities.


## Integration Flow Initiation

The integration flow can be initiated in three ways, using different Sender Adapters:

1. **OData Sender (VA_DK_SalesFactSet OData query):**  Triggers the `Main process` upon receiving an OData request for the `VA_DK_SalesFactSet` entity.  This uses basic authentication. The  `edmxPath` points to `/edmx/Biobest_Finneg.edmx`. The `entitySet` is `VA_DK_SalesFactSet`.

2. **OData Sender (VD_DK_CUSTSet OData Query):** Triggers the `Main process` upon receiving an OData request for the `VD_DK_CUSTSet` entity. This uses basic authentication. The `edmxPath` points to `/edmx/Biobest_Finneg.edmx`. The `entitySet` is `VD_DK_CUSTSet`.

3. **OData Sender (VD_DK_ItemSet OData query):** Triggers the `Main process` upon receiving an OData request for the `VD_DK_ItemSet` entity. This uses basic authentication. The `edmxPath` points to `/edmx/Biobest_Finneg.edmx`. The `entitySet` is `VD_DK_ItemSet`.

4. **Timer (Integration Process):** Triggers the `Main process` based on a scheduled timer.  This provides a mechanism for periodic execution of the overall process.


## Detailed Process Description

### Main process

This process is the core of the integration flow, handling token retrieval, external API calls, and data routing.

**Steps:**

1. **Start 1:** The initial start event of the `Main process`.

2. **Set Secure param:** This `ContentModifier` step creates a property named `SecureParam` with a constant value of `BioBest_ClientSecret`.  This is presumably a client secret used for authentication.

| Property Name | Action | Type | Value | Default | Datatype |
|---|---|---|---|---|---|
| SecureParam | Create | constant | BioBest_ClientSecret |  |  |


3. **Get token:** This step makes an HTTP call to a third-party API to obtain an access token.

    * **Adapter:** HTTP
    * **API Endpoint:** `https://api.teamplace.finneg.com/api/oauth/token`
    * **API Endpoint Query:** `grant_type=client_credentials&client_id=033169c62d4e4bc6b7356fbdbbdbede8&client_secret=${property.P_Password}`  (Note:  `P_Password` is assumed to be a secure parameter containing the client secret).

4. **Store access token:** This `ContentModifier` step stores the response body (access token) from the previous step into a property named `AccessToken`.

| Property Name | Action | Type | Value | Default | Datatype |
|---|---|---|---|---|---|
| AccessToken | Create | expression | `${in.body}` |  | String |


5. **Get Data:** This step makes an HTTP call to another third-party API to retrieve data.

    * **Adapter:** HTTP
    * **API Endpoint:** `https://api.finneg.com/api/reports/ANALISISVENTASBBAPI`
    * **API Endpoint Query:** `ACCESS_TOKEN=${property.AccessToken}&PARAMWEBREPORT_FechaDesde=2023-01-01&PARAMWEBREPORT_FechaHasta=2024-12-30&PARAMWEBREPORT_Moneda=DOL&PARAMWEBREPORT_Empresa=BIOBEST36`  (The access token obtained in the previous step is used here).

6. **Test body:** This step likely performs some validation or transformation on the data received from the external API.

7. **Format JSON:** This step formats the data into a JSON structure (the exact format is not specified).

8. **Is Which entity:** This router step determines which OData entity to update based on the value of the  `SAP_ODataEntityName` property.  It has four outgoing conditions:

    * **VD_DK_CUST:**  If `SAP_ODataEntityName` is `VD_DK_CUSTSet`, routes to `Set VD_DK_CUSTSet body`.
    * **is VA_DK_SalesFactSet:** If `SAP_ODataEntityName` is `VA_DK_SalesFactSet`, routes to `Set VA_DK_SalesFact body`.
    * **Is VD_DK_ItemSet:** If `SAP_ODataEntityName` is `VD_DK_ItemSet`, routes to `Set VD_DK_Item body`.
    * **No:** Default route to `End 3`.

9. **Set VD_DK_CUSTSet body:** Prepares the message body for updating the `VD_DK_CUSTSet` OData entity (details not specified).

10. **Set VA_DK_SalesFact body:** Prepares the message body for updating the `VA_DK_SalesFactSet` OData entity (details not specified).

11. **Set VD_DK_Item body:** Prepares the message body for updating the `VD_DK_ItemSet` OData entity (details not specified).


12. **End 1:** The final end event for the `Main process`.
13. **End 3:** An additional end event used if none of the router conditions are met in step 8.



### Other Processes

The other processes (`VA_DK_SalesFactSet OData query`, `VD_DK_CUSTSet OData Query`, `VD_DK_ItemSet OData query`, `Integration Process`) are relatively simple and consist of a start event, a call to the `Main process`, and an end event.  Their functionality is solely to trigger the central `Main process`.


This specification provides a comprehensive overview of the integration flow.  Further detail on the internal processing steps (e.g.,  `Set VD_DK_CUSTSet body`)  would require access to the internal mappings and transformations within those steps.
