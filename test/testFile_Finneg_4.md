# Technical Specification: SAP Cloud Integration Flow

This document details the technical specifications for the integration flow implemented in SAP Cloud Integration.  The flow comprises several processes working together to retrieve and process data.

## Process Overview

The integration flow consists of five main processes:

1. **VA_DK_SalesFactSet OData query:**  This process initiates an OData query against the `VA_DK_SalesFactSet` entity in the Biobest_Finneg.edmx service.
2. **VD_DK_CUSTSet OData Query:** This process initiates an OData query against the `VD_DK_CUSTSet` entity in the Biobest_Finneg.edmx service.
3. **VD_DK_ItemSet OData query:** This process initiates an OData query against the `VD_DK_ItemSet` entity in the Biobest_Finneg.edmx service.
4. **Integration Process:** This process acts as a scheduled trigger, initiating the main process.
5. **Main process:** This process orchestrates the retrieval of data from external systems,  data transformation, and potentially storage,  combining data from the other OData queries.


## Integration Flow Initiation

The integration flow can be initiated in three ways, each using a different `SenderAdapter`:

1. **OData Sender (VA_DK_SalesFactSet OData query):**  Initiated by an OData request targeting the `VA_DK_SalesFactSet` entity set within the `Biobest_Finneg.edmx` service.  Authentication is done via Basic Authentication.
2. **OData Sender (VD_DK_CUSTSet OData Query):** Initiated by an OData request targeting the `VD_DK_CUSTSet` entity set within the `Biobest_Finneg.edmx` service. Authentication is done via Basic Authentication.
3. **OData Sender (VD_DK_ItemSet OData query):** Initiated by an OData request targeting the `VD_DK_ItemSet` entity set within the `Biobest_Finneg.edmx` service. Authentication is done via Basic Authentication.
4. **Timer (Integration Process):** The `Integration Process` is triggered by a timer, initiating the `Main process` at scheduled intervals.


## Detailed Process Description

### Main process (`Main process`)

This process is the central orchestrator, called by the other three processes.

**Steps:**

1. **Start 1:** The process starts.
2. **Set Secure param:**  This `ContentModifier` step creates a property named `SecureParam` with a constant value of `BioBest_ClientSecret`. This is presumably used for authentication in subsequent steps.

   | Property Name | Action | Type     | Value                 | Default | Datatype |
   |--------------|--------|----------|----------------------|---------|----------|
   | SecureParam  | Create  | constant | BioBest_ClientSecret |         |          |


3. **Get secure parameter:** This step retrieves the secure parameter set in the previous step.  The exact implementation is not detailed here but likely involves accessing a secure storage mechanism within SAP CPI.
4. **Get token:** This step makes an HTTP call to an external system (`API_CALL` with adapter `HTTP`) to retrieve an access token.  The URL and specific request details are not provided in the XML. The `SecureParam` is likely used for authentication within this HTTP call.  The response (access token) is stored in the message body.
5. **Store access token:** This `ContentModifier` step extracts the access token from the message body and stores it in a message property named `AccessToken`.

   | Property Name | Action | Type     | Value                 | Default | Datatype |
   |--------------|--------|----------|----------------------|---------|----------|
   | AccessToken  | Create  | expression | ${in.body}            |         | String   |


6. **Get Data:** This step performs another HTTP call (`API_CALL` with adapter `HTTP`) to a different external system (`Receiver1`). The `AccessToken` is presumably used for authentication. The response body likely contains the data needed for subsequent processing.
7. **Test body:** This step likely performs validation or transformation of the data received from `Get Data`. The exact logic is not specified.
8. **Format JSON:**  This step formats the data into JSON format. The exact transformation is not detailed.
9. **Set VA_DK_SalesFact body:** This step prepares the data for the `VA_DK_SalesFactSet` OData call. This involves mapping and potentially transforming data from the previous step to match the structure of the target entity.
10. **Set VD_DK_CUSTSet body:** This step prepares the data for the `VD_DK_CUSTSet` OData call. This involves mapping and potentially transforming data from the previous step to match the structure of the target entity.
11. **Set VD_DK_Item body:** This step prepares the data for the `VD_DK_ItemSet` OData call. This involves mapping and potentially transforming data from the previous step to match the structure of the target entity.
12. **End 1:** The process ends.


### VA_DK_SalesFactSet OData query

This process initiates an OData query to fetch data from the `VA_DK_SalesFactSet` entity set. It then calls the `Main process`.

**Steps:**

1. **Start 3:** Initiated via OData request to `VA_DK_SalesFactSet`.
2. **Call main process:** Calls the `Main process` to process the data.
3. **End 4:** The process ends.


### VD_DK_CUSTSet OData Query

This process initiates an OData query to fetch data from the `VD_DK_CUSTSet` entity set. It then calls the `Main process`.

**Steps:**

1. **Start 2:** Initiated via OData request to `VD_DK_CUSTSet`.
2. **Call local process:** Calls the `Main process` to process the data.
3. **End 2:** The process ends.


### VD_DK_ItemSet OData query

This process initiates an OData query to fetch data from the `VD_DK_ItemSet` entity set. It then calls the `Main process`.

**Steps:**

1. **Start 3:** Initiated via OData request to `VD_DK_ItemSet`.
2. **Call main process:** Calls the `Main process` to process the data.
3. **End 4:** The process ends.


### Integration Process

This process is triggered by a timer and simply calls the `Main process`.

**Steps:**

1. **Start Timer 1:** Initiated by a timer.
2. **Call local process:** Calls the `Main process`.
3. **End:** The process ends.


**Note:**  The XML provided lacks detail on specific data transformations and mappings within the `Main process`.  Further documentation is needed to fully understand the data flow and logic.  Also, the exact URLs and request/response structures for the HTTP calls are missing.  This specification provides a high-level overview and needs to be supplemented with more detailed information.
