```markdown
# Integration Flow Technical Specification

This document provides a detailed technical specification for the SAP Cloud Integration flow. It outlines the processes, their steps, and configurations, enabling a new team member to quickly understand the integration logic.

## Process Overview

The integration flow consists of several interconnected processes, which are visualized below. The main integration flow (`Integration Process`) orchestrates the execution of other processes based on configurations and data. The primary goal is to process user data from SuccessFactors and integrate it with other systems, handling errors and logging along the way.

![Process Overview](https://i.imgur.com/06737I7.png)

*   **Integration Process**: The main process which acts as the orchestrator.
*   **Local Integration Process Selected Users**: Fetches and processes data for a selected group of users.
*   **Local Looping Integration Process All Users**: Fetches and processes data for all users.
*   **Process Employee**: Transforms and sends user data to another system.
*   **Invalid Log**: Handles logging of invalid entries.
*   **Extract Error Process**: Handles exceptions and error logging.
*   **Write LastRun**: Writes the last execution time.
*   **Read LastRun**: Reads the last execution time.
*  **Log**: Handles Logging of data using Email and/or SFTP.

## Integration Flow Initiation

The `Integration Process` is initiated by a timer start event. This means the integration flow starts automatically based on the configured schedule, and does not use any inbound adapter to start the flow:

*   **Start Timer**: The process starts based on a timer schedule that has been configured in the Cloud Integration platform.

## Detailed Process Description

Each process and their individual steps are described below.

### 1. Integration Process

This is the main process that controls the overall flow of the integration.

1.  **Start Timer**:
    *   **Type**: `StartEvent`
    *   **Description**: The integration flow starts based on the configured timer schedule.
    *   **Outgoing**: `SequenceFlow_75333170`
2.  **Read LastRun**:
    *   **Type**: `CallLocalProcess`
    *   **Description**: Calls the `Read LastRun` process to get the last execution time.
    *   **Outgoing**: `SequenceFlow_75333692`
    *   **Incoming**: `SequenceFlow_75333170`
    *   **Called Process**: `Read LastRun`
3.  **Configure Flow**:
    *   **Type**: `ContentModifier`
    *   **Description**: Sets the configuration parameters for the flow.
    *   **Outgoing**: `SequenceFlow_75333171`
    *   **Incoming**: `SequenceFlow_75333692`

    **Content Modifier Properties:**
    | Property Name       | Action | Type     | Value                     | Data Type  |
    |---------------------|--------|----------|---------------------------|------------|
    | RolesToExclude      | Create | constant | {{Roles_To_Exclude}}      |            |
    | queryusernames      | Create | constant | {{SF_Query_usernames}}   |            |
    | processDirectUrl    | Create | constant | {{ProcessDirect_Url}}     |            |
    | debuggingEnabled    | Create | constant | {{Debugging_Enabled}}     |            |
    | emailFrom           | Create | constant | {{Email_From}}            |            |
    | EmailNotificationsEnabled | Create | constant | {{Email_Notifications_Enabled}} |  |
    | SFTPLoggingEnabled  | Create | constant | {{SFTP_Logging_Enabled}}  |            |
    | emailRecipients     | Create | constant | {{Email_Recipients}}      |            |
    | processAllEmployees | Create | constant | {{All_Employees}}       |            |

4.  **Router - all / selected users**:
    *   **Type**: `router`
    *   **Description**: Routes the flow based on the value of `processAllEmployees` property.
    *   **Outgoing**:
        *   **Selected Employees**: `SequenceFlow_75333173` (Default Expression)
        *   **All Employees**: `SequenceFlow_75333155` (Condition: `${property.processAllEmployees} = 'true'`)
    *   **Incoming**: `SequenceFlow_75333171`
5.  **Process Call - Selected Users**:
    *   **Type**: `CallLocalProcess`
    *   **Description**: Calls the `Local Integration Process Selected Users` process.
    *   **Outgoing**: `SequenceFlow_75333511`
    *   **Incoming**: `SequenceFlow_75333174`
    *   **Called Process**: `Local Integration Process Selected Users`
6.  **Looping Process Call - All Users**:
    *   **Type**: `CallLocalProcess`
    *   **Description**: Calls the `Local Looping Integration Process All Users` process.
    *   **Outgoing**: `SequenceFlow_75333510`
    *   **Incoming**: `SequenceFlow_75333172`
    *   **Called Process**: `Local Looping Integration Process All Users`
7.  **Router - Log Invalid Entries**:
    *   **Type**: `router`
    *   **Description**: Routes the flow based on the number of errors found in the message body.
    *    **Outgoing**:
        *   **Invalid Entries**: `SequenceFlow_75333254` (Condition: `count(/root/error) > 0`)
        *    **No Invalid Entries**: `SequenceFlow_75333710` (Default Expression)
    *   **Incoming**: `SequenceFlow_75333510, SequenceFlow_75333511`
8.  **Invalid Log**:
    *   **Type**: `CallLocalProcess`
    *   **Description**: Calls the `Invalid Log` process to log invalid entries.
    *   **Outgoing**: `SequenceFlow_75333255`
    *   **Incoming**: `SequenceFlow_75333254`
    *   **Called Process**: `Invalid Log`
9. **Write LastRun**:
    *  **Type**: `CallLocalProcess`
    *   **Description**: Calls the `Write LastRun` process to save the timestamp for this run.
    *   **Outgoing**: `SequenceFlow_75333711`
    *   **Incoming**: `SequenceFlow_75333255, SequenceFlow_75333252`
    *   **Called Process**: `Write LastRun`
10. **End**:
    *   **Type**: `endEvent`
    *   **Description**: Marks the end of the main integration process.
    *   **Incoming**: `SequenceFlow_75333711`

### 2. Local Integration Process Selected Users

This process handles user data for a selected group of users.

1.  **Start**:
    *   **Type**: `StartEvent`
    *   **Description**: Marks the start of the process.
    *   **Outgoing**: `SequenceFlow_75333514`
2.  **Request Reply - SF**:
    *   **Type**: `API_CALL`
    *   **Description**: Calls the SuccessFactors OData API to retrieve user data.
    *   **Outgoing**: `SequenceFlow_75333513`
    *   **Incoming**: `SequenceFlow_75333514`
    *   **Adapter**: `SuccessFactorsODataSingleUser`
    *   **Receiver**: `SF`
        *  **This step calls a 3rd party system: SuccessFactors**
3.  **Router - Empty Result**:
    *   **Type**: `router`
    *   **Description**: Routes the flow based on whether the result of the SF query contains data.
    *   **Outgoing**:
        *   **Valid result**: `SequenceFlow_75333193` (Default Expression)
        *   **Empty result**: `SequenceFlow_75333178` (Condition: `not(//User/User/node())`)
    *   **Incoming**: `SequenceFlow_75333513`
4.  **Process Call - Process Employees**:
     *   **Type**: `CallLocalProcess`
     *   **Description**: Calls the `Process Employee` process to transform the fetched user data.
     *   **Outgoing**: `SequenceFlow_75333623`
     *   **Incoming**: `SequenceFlow_75333186`
     *   **Called Process**: `Process Employee`
5. **Filter**:
   *   **Type**: `Filter`
   *   **Description**: Filters out users based on some internal logic
   *   **Outgoing**: `SequenceFlow_75333797`
   *  **Incoming**: `SequenceFlow_75333623`
6.  **Save Body**:
    *   **Type**: `ContentModifier`
    *   **Description**: Stores the body in a property, this is used for looping.
    *   **Outgoing**: `SequenceFlow_75333800`
    *   **Incoming**: `SequenceFlow_75333797`

    **Content Modifier Properties:**
    | Property Name | Action | Type     | Value                      | Data Type       |
    |---------------|--------|----------|----------------------------|-----------------|
    | savedBody    | Create | expression | `${property.savedBody}${body}`| `java.lang.String` |

7. **Set Body**:
    *   **Type**: `ContentModifier`
    *   **Description**: Sets the message body to the property `savedBody`
    *   **Outgoing**: `SequenceFlow_75333803`
    *   **Incoming**: `SequenceFlow_75333800`
8.  **End**:
    *   **Type**: `endEvent`
    *   **Description**: Marks the end of the process.
    *  **Incoming**: `SequenceFlow_75333273, SequenceFlow_75333803`

### 3. Local Looping Integration Process All Users

This process handles user data for all users, it also loops over all the results that are fetched from SuccessFactors.

1.  **Start**:
    *   **Type**: `StartEvent`
    *   **Description**: Marks the start of the process.
    *   **Outgoing**: `SequenceFlow_75333183`
2. **Request Reply - SF All users**:
    *   **Type**: `API_CALL`
    *   **Description**: Calls the SuccessFactors OData API to retrieve all user data.
    *   **Outgoing**: `SequenceFlow_75333184`
    *   **Incoming**: `SequenceFlow_75333183`
    *   **Adapter**: `SuccessFactorsODataAllUsers`
    *   **Receiver**: `SF`
        *  **This step calls a 3rd party system: SuccessFactors**
3.  **Router - Empty Result**:
    *   **Type**: `router`
    *   **Description**: Routes the flow based on whether the result of the SF query contains data.
    *   **Outgoing**:
        *   **Valid result**: `SequenceFlow_75333197` (Default Expression)
        *   **Empty result**: `SequenceFlow_75333182` (Condition: `not(//User/User/node())`)
    *   **Incoming**: `SequenceFlow_75333184`
4.   **Process Call - Process Employees**:
     *   **Type**: `CallLocalProcess`
     *   **Description**: Calls the `Process Employee` process to transform the fetched user data.
     *   **Outgoing**: `SequenceFlow_75333795`
     *   **Incoming**: `SequenceFlow_75333189`
     *   **Called Process**: `Process Employee`
5.   **Filter**:
      *   **Type**: `Filter`
      *   **Description**: Filters out users based on some internal logic.
      *   **Outgoing**: `SequenceFlow_75333794`
      *   **Incoming**: `SequenceFlow_75333795`
6.  **Save Body**:
    *   **Type**: `ContentModifier`
    *   **Description**: Stores the message body in a property, for looping.
    *   **Outgoing**: `SequenceFlow_75333792`
    *  **Incoming**: `SequenceFlow_75333794`
    **Content Modifier Properties:**
    | Property Name | Action | Type     | Value                      | Data Type       |
    |---------------|--------|----------|----------------------------|-----------------|
    | savedBody    | Create | expression | `${property.savedBody}${body}`| `java.lang.String` |
7. **Set Body**:
    *   **Type**: `ContentModifier`
    *   **Description**: Sets the message body to the property `savedBody`.
    *  **Outgoing**: `SequenceFlow_75333781`
    *   **Incoming**: `SequenceFlow_75333792`
8.  **End**:
    *   **Type**: `endEvent`
    *   **Description**: Marks the end of the process.
    *   **Incoming**: `SequenceFlow_75333200, SequenceFlow_75333781`

### 4. Process Employee

This process transforms individual user data and sends it to another system.

1.  **Start**:
    *   **Type**: `StartEvent`
    *   **Description**: Marks the start of the process.
    *   **Outgoing**: `SequenceFlow_75333202`
2.  **Message Mapping - filter dates**:
    *   **Type**: `MessageMapping`
    *   **Description**: Transforms the data, filtering dates using message mapping `MM_SFSF_Filter_Nodes_Dates`.
    *   **Outgoing**: `SequenceFlow_75333903`
    *   **Incoming**: `SequenceFlow_75333202`
    *   **Mapping Name**: `MM_SFSF_Filter_Nodes_Dates`
3.  **XSLT Mapping - exclude filter_roles**:
    *   **Type**: `XSLTMapping`
    *   **Description**: Excludes roles based on some internal logic, by using the XSLT script `excludefilter_roles`.
    *   **Outgoing**: `SequenceFlow_75333905`
    *   **Incoming**: `SequenceFlow_75333903`
    *   **XSLT Script**: `excludefilter_roles`
4.  **XSLT Mapping - append roles**:
    *   **Type**: `XSLTMapping`
    *   **Description**: Appends additional roles using the XSLT script `append_roles`.
    *   **Outgoing**: `SequenceFlow_75333906`
    *   **Incoming**: `SequenceFlow_75333905`
    *   **XSLT Script**: `append_roles`
5.  **Splitter**:
    *   **Type**: `Splitter`
    *   **Description**: Splits the input message in to multiple messages
    *   **Outgoing**: `SequenceFlow_75333758`
    *   **Incoming**: `SequenceFlow_75333906`
6.  **Content Modifier - username, firstName, lastName, statute**:
    *   **Type**: `ContentModifier`
    *   **Description**: sets properties for username, firstName, lastName, and statute.
    *   **Outgoing**: `SequenceFlow_75333725`
    *   **Incoming**: `SequenceFlow_75333758`
7.  **Request Reply**:
     *   **Type**: `API_CALL`
     *   **Description**: Calls the ProcessDirect API to send user information
     *   **Outgoing**: `SequenceFlow_75333765`
     *   **Incoming**: `SequenceFlow_75333725`
     *    **Adapter**: `ProcessDirect-EDS-AzureAD`
     *   **Receiver**: `Receiver-ProcessDirect-EDS-AzureAD`
         * **This step calls a 3rd party system using ProcessDirect**
8. **XSLT Mapping - append filter_username, filter_firstName, filter_lastName, filter_statute**:
    *   **Type**: `XSLTMapping`
    *   **Description**: Appends user info using the XSLT script `appendfilter_username`.
    *   **Outgoing**: `SequenceFlow_75333901`
    *   **Incoming**: `SequenceFlow_75333765`
    *   **XSLT Script**: `appendfilter_username`
9.  **Gather**:
    *   **Type**: `Gather`
    *   **Description**: Gathers the message bodies and combines them into 1 message.
    *   **Outgoing**: `SequenceFlow_75333728`
    *   **Incoming**: `SequenceFlow_75333901`
10. **Router  debug**:
     *   **Type**: `router`
     *   **Description**: Routes the flow based on the value of `debuggingEnabled` property.
     *   **Outgoing**:
         *   **Yes**: `SequenceFlow_75333721` (Condition: `${property.debuggingEnabled} = 'true'`)
         *   **No**: `SequenceFlow_75333559` (Default Expression)
     *    **Incoming**: `SequenceFlow_75333728`
11.  **Send**:
    *   **Type**: `API_CALL`
    *   **Description**: Sends invalid AD users to SFTP server for logging.
    *   **Outgoing**: `SequenceFlow_75333726`
    *    **Incoming**: `SequenceFlow_75333884`
    *   **Adapter**: `SFTP-Process-Employee-InvalidADUsers`
    *   **Receiver**: `SFTP-Log`
        *  **This step calls a 3rd party system using SFTP**
12. **End**:
     *   **Type**: `endEvent`
     *   **Description**: Marks the end of the process.
     *    **Incoming**: `SequenceFlow_75333726, SequenceFlow_75333886`

### 5. Invalid Log

This process handles the logging of invalid entries.

1.  **Start**:
    *   **Type**: `StartEvent`
    *   **Description**: Marks the start of the process.
    *   **Outgoing**: `SequenceFlow_75333870`
2.  **Content Modifier**:
    *   **Type**: `ContentModifier`
    *   **Description**: Sets the filename for the log.
    *   **Outgoing**: `SequenceFlow_75333872`
    *   **Incoming**: `SequenceFlow_75333870`

    **Content Modifier Properties:**
    | Property Name | Action | Type     | Value                     | Data Type  |
    |---------------|--------|----------|---------------------------|------------|
    | filename      | Create | expression | `invalid_${date:now:yyyyMMdd}.xml` |             |

3.  **XSLT Mapping Invalid entries**:
    *   **Type**: `XSLTMapping`
    *   **Description**: Transforms the input message using XSLT script `xmlinvalidlogging`.
    *   **Outgoing**: `SequenceFlow_75333874`
    *   **Incoming**: `SequenceFlow_75333872`
    *    **XSLT Script**: `xmlinvalidlogging`
4.  **Process Call - Log**:
    *   **Type**: `CallLocalProcess`
    *   **Description**: Calls the `Log` process to handle the actual logging.
    *   **Outgoing**: `SequenceFlow_75333880`
    *   **Incoming**: `SequenceFlow_75333874`
    *    **Called Process**: `Log`
5.  **End**:
    *   **Type**: `endEvent`
    *   **Description**: Marks the end of the process.
    *    **Incoming**: `SequenceFlow_75333880`

### 6. Extract Error Process

This process handles exceptions and error logging.

1.  **Start**:
    *   **Type**: `StartEvent`
    *   **Description**: Marks the start of the process.
    *   **Outgoing**: `SequenceFlow_75333875`
2.  **CM - filename / stacktrace**:
    *   **Type**: `ContentModifier`
    *   **Description**: Sets the filename, exception message, and stack trace for logging.
    *   **Outgoing**: `SequenceFlow_75333034`
    *   **Incoming**: `SequenceFlow_75333875`

    **Content Modifier Properties:**
    | Property Name             | Action | Type     | Value                       | Data Type       |
    |---------------------------|--------|----------|-----------------------------|-----------------|
    | exception_message_to_escape   | Create | expression | `exception.message`      | `java.lang.String` |
    | filename                  | Create | expression | `general_error_${date:now:yyyyMMdd}.xml`    |            |
    | exception_to_escape        | Create | expression | `exception.stacktrace`    | `java.lang.String` |

3.  **XSLT Mapping Exception**:
    *   **Type**: `XSLTMapping`
    *   **Description**: Transforms the error message using XSLT script `Exception`.
    *   **Outgoing**: `SequenceFlow_75333877`
    *   **Incoming**: `SequenceFlow_75333034`
    *   **XSLT Script**: `Exception`
4.  **Process Call - Log**:
    *   **Type**: `CallLocalProcess`
    *   **Description**: Calls the `Log` process to handle the actual logging.
    *   **Outgoing**: `SequenceFlow_75333878`
    *   **Incoming**: `SequenceFlow_75333877`
    *    **Called Process**: `Log`
5.  **End**:
    *   **Type**: `endEvent`
    *   **Description**: Marks the end of the process.
    *    **Incoming**: `SequenceFlow_75333878`

### 7. Write LastRun

This process writes the timestamp of the current run.

1.  **Start**:
    *   **Type**: `StartEvent`
    *   **Description**: Marks the start of the process.
    *   **Outgoing**: `SequenceFlow_75333704`
2.  **Content Modifier**:
    *   **Type**: `ContentModifier`
    *   **Description**: Sets the `thisRun` property to the current timestamp.
    *   **Outgoing**: `SequenceFlow_75333706`
    *   **Incoming**: `SequenceFlow_75333704`

    **Content Modifier Properties:**
    | Property Name | Action | Type     | Value                               | Data Type  |
    |---------------|--------|----------|-------------------------------------|------------|
    | thisRun      | Create | expression | `${date:now:YYYY-MM-dd'T'HH:mm:ss.SSSXXX}` |            |

3. **Write LastSync**:
    *   **Type**: `Write`
    *   **Description**: Writes the last sync to the datastore.
    *   **Outgoing**: `SequenceFlow_75333709`
    *   **Incoming**: `SequenceFlow_75333706`
4.  **End**:
    *   **Type**: `endEvent`
    *   **Description**: Marks the end of the process.
    *   **Incoming**: `SequenceFlow_75333709`

### 8. Read LastRun

This process reads the timestamp of the last run.

1.  **Start**:
    *   **Type**: `StartEvent`
    *   **Description**: Marks the start of the process.
    *   **Outgoing**: `SequenceFlow_75333697`
2.  **Content Modifier**:
    *  **Type**: `ContentModifier`
    *   **Description**: Modifies the content but performs no action.
    *   **Outgoing**: `SequenceFlow_75333699`
    *   **Incoming**: `SequenceFlow_75333697`
3.  **End**:
    *   **Type**: `endEvent`
    *   **Description**: Marks the end of the process.
    *   **Incoming**: `SequenceFlow_75333699`

### 9. Log

This process logs the processed messages.

1.  **Start**:
    *   **Type**: `StartEvent`
    *   **Description**: Marks the start of the process.
    *   **Outgoing**: `SequenceFlow_75333882`
2.  **Content Modifier**:
    *   **Type**: `ContentModifier`
    *   **Description**: Retrieves email information from the input message.
    *   **Outgoing**: `SequenceFlow_75333836`
    *   **Incoming**: `SequenceFlow_75333830`

    **Content Modifier Properties:**
    | Property Name | Action | Type | Value                     | Data Type       |
    |---------------|--------|------|---------------------------|-----------------|
    | mail_from     | Create | xpath| `/mailOutput/from/text()`| `java.lang.String` |
    | mail_title    | Create | xpath| `/mailOutput/title/text()`  | `java.lang.String` |
    | mail_to       | Create | xpath| `/mailOutput/recipients/text()`|`java.lang.String`|
    | mail_body     | Create | xpath| `/mailOutput/body/text()`   | `java.lang.String`|

3.  **Mail**:
    *   **Type**: `router`
    *   **Description**: Routes the flow based on the value of `EmailNotificationsEnabled` property.
    *    **Outgoing**:
        *   **Yes**: `SequenceFlow_75333829` (Condition: `${property.EmailNotificationsEnabled} = 'true'`)
        *   **No**: `SequenceFlow_75333031` (Default Expression)
    *   **Incoming**: `SequenceFlow_75333835`
4.  **SFTP**:
    *   **Type**: `router`
    *   **Description**: Routes the flow based on the value of `SFTPLoggingEnabled` property.
    *    **Outgoing**:
         *   **Yes**: `SequenceFlow_75333036` (Condition: `${property.SFTPLoggingEnabled} = 'true'`)
         *   **No**: `SequenceFlow_75333031` (Default Expression)
    *   **Incoming**: `SequenceFlow_75333832`
5.  **Send Mail**:
     *   **Type**: `API_CALL`
     *   **Description**: Sends email with logged message
     *   **Outgoing**: `SequenceFlow_75333895`
     *   **Incoming**: `SequenceFlow_75333836`
     *    **Adapter**: `Mail-Exceptions`
     *   **Receiver**: `Receiver-Email`
        *  **This step calls a 3rd party system using Email**
6. **Send SFTP**:
    *   **Type**: `API_CALL`
    *   **Description**: Sends logged message using SFTP protocol.
    *   **Outgoing**: `SequenceFlow_75333891`
    *   **Incoming**: `SequenceFlow_75333822`
    *   **Adapter**: `SFTP-Error`
    *   **Receiver**: `SFTP-Log`
       *   **This step calls a 3rd party system using SFTP**
7.  **End**:
    *   **Type**: `endEvent`
    *   **Description**: Marks the end of the process.
    *    **Incoming**: `SequenceFlow_75333895, SequenceFlow_75333894, SequenceFlow_75333893, SequenceFlow_75333891`

This document provides a comprehensive technical overview of the integration flow. By reviewing each process and its steps, a new team member should gain a clear understanding of the integration logic and its functionalities.
