# Technical Specification: SAP Cloud Integration Flows

This document details the technical specifications for the SAP Cloud Integration flows described in the provided XML file.  The flows are designed to integrate data from various sources, primarily using OData calls and external HTTP APIs.


## 1. Process Overview

The integration solution consists of four main processes:

* **VA_DK_SalesFactSet OData query:** This process retrieves data from the `VA_DK_SalesFactSet` entity using an OData call.
* **VD_DK_CUSTSet OData Query:** This process retrieves data from the `VD_DK_CUSTSet` entity using an OData call.
* **VD_DK_ItemSet OData query:** This process retrieves data from the `VD_DK_ItemSet` entity using an OData call.
* **Main process:** This is the central orchestration process, responsible for handling authentication, data retrieval from an external HTTP API, data transformation, and routing based on the data type. This process then calls the other processes to handle OData queries for different entities.


## 2. Integration Flow Initiation

The integration flows are initiated via OData Sender Adapters in three different processes:

* **VA_DK_SalesFactSet OData query:** Initiated by an OData request to the `VA_DK_SalesFactSet` entity set in the `/edmx/Biobest_Finneg.edmx` EDMX file. Uses basic authentication.
* **VD_DK_CUSTSet OData Query:** Initiated by an OData request to the `VD_DK_CUSTSet` entity set in the `/edmx/Biobest_Finneg.edmx` EDMX file. Uses basic authentication.
* **VD_DK_ItemSet OData query:** Initiated by an OData request to the `VD_DK_ItemSet` entity set in the `/edmx/Biobest_Finneg.edmx` EDMX file. Uses basic authentication.
* **Integration Process:** Initiated by a timer event.


## 3. Detailed Process Description

This section describes the steps within each process in detail.

### 3.1 Main process

This process acts as the central integration point, orchestrating the retrieval of data from an external API and subsequent OData calls.

**Steps:**

1. **Start 1:** The process starts.

2. **Set Secure param:**  This `ContentModifier` step creates a property named `SecureParam` with a constant value of `BioBest_ClientSecret`.  This is likely a client secret for authentication.

| Property Name | Action | Type | Value | Default | Datatype |
|---|---|---|---|---|---|
| SecureParam | Create | constant | BioBest_ClientSecret |  |  |

3. **Get secure parameter:** This step retrieves the secure parameter (likely from an SAP Cloud Platform KeyStore). The step's output feeds into "Get token".

4. **Get token:** This step makes an external HTTP call to retrieve an access token.

   * **Adapter:** HTTP
   * **Receiver:** Receiver
   * **API Endpoint:** `https://api.teamplace.finneg.com/api/oauth/token`
   * **API Endpoint Query:** `grant_type=client_credentials&client_id=033169c62d4e4bc6b7356fbdbbdbede8&client_secret=${property.P_Password}`  (Note: `P_Password` is presumably a secure parameter containing the client secret.)


5. **Store access token:** This `ContentModifier` step stores the access token received from the previous step in a message property.

| Property Name | Action | Type | Value | Default | Datatype |
|---|---|---|---|---|---|
| AccessToken | Create | expression | `${in.body}` |  | String |

6. **Get Data:** This step makes an external HTTP call to retrieve data.

   * **Adapter:** HTTP
   * **Receiver:** Receiver1
   * **API Endpoint:** `https://api.finneg.com/api/reports/ANALISISVENTASBBAPI`
   * **API Endpoint Query:** `ACCESS_TOKEN=${property.AccessToken}&PARAMWEBREPORT_FechaDesde=2023-01-01&PARAMWEBREPORT_FechaHasta=2024-12-30&PARAMWEBREPORT_Moneda=DOL&PARAMWEBREPORT_Empresa=BIOBEST36`

7. **Test body:** This step likely performs some validation or transformation on the data received from the external API. This step's output feeds into "Is Which entity".


8. **Format JSON:** This step formats the data received from "Test body" into JSON format. The step's output feeds into "Is Which entity".

9. **Is Which entity:** This router step determines which OData query process to call next based on the value of the `SAP_ODataEntityName` property.  It has four branches:

    * **VD_DK_CUST:** Calls the `VD_DK_CUSTSet OData Query` process if `SAP_ODataEntityName` is 'VD_DK_CUSTSet'.
    * **is VA_DK_SalesFactSet:** Calls the `VA_DK_SalesFactSet OData query` process if `SAP_ODataEntityName` is 'VA_DK_SalesFactSet'.
    * **Is VD_DK_ItemSet:** Calls the `VD_DK_ItemSet OData query` process if `SAP_ODataEntityName` is 'VD_DK_ItemSet'.
    * **No:**  If none of the above conditions are met, the flow goes to "End 3".

10. **Set VD_DK_CUSTSet body:** This step sets the request body for the VD_DK_CUSTSet OData call (after the call from Is Which entity). The step's output feeds into "End 1".

11. **Set VA_DK_SalesFact body:** This step sets the request body for the VA_DK_SalesFactSet OData call (after the call from Is Which entity). The step's output feeds into "End 1".

12. **Set VD_DK_Item body:** This step sets the request body for the VD_DK_ItemSet OData call (after the call from Is Which entity). The step's output feeds into "End 1".

13. **End 1:** The main process ends.
14. **End 3:**  The main process ends if the router step does not find a match.



### 3.2 VA_DK_SalesFactSet OData query

1. **Start 3:** OData request initiated.
2. **Call main process:** Calls the `Main process`.
3. **End 4:** Process ends.

### 3.3 VD_DK_CUSTSet OData Query

1. **Start 2:** OData request initiated.
2. **Call local process:** Calls the `Main process`.
3. **End 2:** Process ends.


### 3.4 VD_DK_ItemSet OData query

1. **Start 3:** OData request initiated.
2. **Call main process:** Calls the `Main process`.
3. **End 4:** Process ends.


### 3.5 Integration Process

1. **Start Timer 1:**  Process starts based on a timer schedule.
2. **Call local process:** Calls the `Main process`.
3. **End:** Process ends.


This detailed specification provides a comprehensive understanding of the integration flow's logic and functionality.  Further details on specific message structures and error handling would require additional information.
