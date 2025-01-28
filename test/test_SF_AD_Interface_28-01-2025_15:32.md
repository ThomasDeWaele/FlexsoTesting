# Integration Flow Technical Specification

This document provides a detailed technical specification for the SAP Cloud Integration flow, outlining the processes, their initiation, and step-by-step descriptions.

## Process Overview

This integration flow consists of several interconnected processes that orchestrate the retrieval of user data from SuccessFactors, processing it, and logging any errors. The main processes are:

1.  **Integration Process:** The main process that orchestrates the entire integration flow.
2.  **Local Integration Process Selected Users:** A sub-process that retrieves and processes a specific set of users.
3.  **Local Looping Integration Process All Users:** A sub-process that retrieves and processes all users.
4.  **Process Employee:** A sub-process responsible for individual employee data transformation and processing.
5. **Invalid Log**: A sub-process responsible for logging the invalid user entries.
6.  **Read LastRun:** A sub-process that reads the last execution timestamp of the integration flow.
7.  **Write LastRun:** A sub-process that writes the last execution timestamp of the integration flow.
8.  **Extract Error Process**: A sub-process that extracts and logs errors.
9.  **Log**: A sub-process that handles logging via mail and sftp.

## Integration Flow Initiation

The main integration flow, "Integration Process", is initiated by a **Timer Start Event**. This indicates the integration flow is scheduled to run periodically based on the configuration of the timer.

## Detailed Process Description

This section provides a detailed breakdown of each process and its steps.

### Integration Process

This is the main process that drives the entire integration flow. It is responsible for reading the last run time, configuring the flow, calling subprocesses to retrieve and process employee data, and logging any invalid entries.

![Integration Process](https://i.imgur.com/y08K83k.png)

1.  **Start Timer**: The process begins with a Timer Start Event, initiating the flow based on a configured schedule.
2.  **Read LastRun**: Calls the 'Read LastRun' sub-process to get the timestamp of the last successful execution.
3.  **Configure Flow**: A Content Modifier step that sets various properties used throughout the integration flow. These properties are set using constant values.
    
    | Property Name         | Action | Type     | Value                          |
    | :-------------------- | :----- | :------- | :----------------------------- |
    | RolesToExclude        | Create | constant |  `{{Roles_To_Exclude}}`         |
    | queryusernames        | Create | constant | `{{SF_Query_usernames}}`        |
    | processDirectUrl      | Create | constant | `{{ProcessDirect_Url}}`        |
    | debuggingEnabled      | Create | constant | `{{Debugging_Enabled}}`        |
    | emailFrom             | Create | constant | `{{Email_From}}`               |
    | EmailNotificationsEnabled | Create | constant | `{{Email_Notifications_Enabled}}` |
    | SFTPLoggingEnabled    | Create | constant | `{{SFTP_Logging_Enabled}}`     |
    | emailRecipients       | Create | constant | `{{Email_Recipients}}`         |
    | processAllEmployees   | Create | constant | `{{All_Employees}}`             |
4.  **Router - all / selected users**: This Router step determines which user retrieval process should be initiated based on the 'processAllEmployees' property.
    *   **All Employees**: If the `processAllEmployees` property is set to 'true', the "Local Looping Integration Process All Users" is called.
    *   **Selected Employees**: If the `processAllEmployees` property is not 'true', the "Local Integration Process Selected Users" is called.
5.  **Looping Process Call - All Users**: This step calls the "Local Looping Integration Process All Users" to retrieve and process all users.
6.  **Process Call - Selected Users**: This step calls the "Local Integration Process Selected Users" to retrieve and process a specific selection of users.
7.  **Router - Log Invalid Entries**: This Router step checks if there are any invalid entries based on the count of error nodes in the XML payload.
    *   **Invalid Entries**: If there are invalid entries (count(/root/error) > 0) calls the 'Invalid Log' sub-process.
    *   **No Invalid Entries**: If there are no invalid entries, the process continues.
8.  **Invalid Log**: Calls the 'Invalid Log' subprocess, to log invalid entries to sftp.
9.  **Write LastRun**: Calls the 'Write LastRun' sub-process to write the timestamp of the current execution.
10. **End**: The process concludes successfully.

### Local Integration Process Selected Users

This process retrieves and processes a specific selection of users.

![Local Integration Process Selected Users](https://i.imgur.com/7H6599h.png)

1.  **Start**: The process begins with a Start Event.
2.  **Request Reply - SF**: Calls the SuccessFactors system using the 'SuccessFactorsODataSingleUser' adapter to retrieve user data, this is a call to a 3rd party system.
3.  **Router - Empty Result**: This router step checks if the result is empty.
    *   **Empty result**: If the retrieved user data is empty (not(//User/User/node())), the process ends.
    *   **Valid result**: If the retrieved user data is not empty, the process continues.
4.  **Process Call - Process Employees**: Calls the 'Process Employee' sub-process for each user in the received payload.
5.  **Filter**: Filter step to be used in combination with the looping.
6.  **Save Body**: This Content Modifier step saves the current body to a property named 'savedBody'.
    | Property Name | Action | Type       | Value                     | Datatype        |
    | :------------ | :----- | :--------- | :------------------------ | :-------------- |
    | savedBody     | Create | expression | `${property.savedBody}${body}` | `java.lang.String` |
7. **Set Body**: Sets the body, required by the looping construct.
8.  **End**: The process concludes successfully.

### Local Looping Integration Process All Users

This process retrieves and processes all users from SuccessFactors.

![Local Looping Integration Process All Users](https://i.imgur.com/XpX5r6a.png)

1.  **Start**: The process begins with a Start Event.
2.  **Request Reply - SF All users**: Calls the SuccessFactors system using the 'SuccessFactorsODataAllUsers' adapter to retrieve all user data, this is a call to a 3rd party system.
3.  **Router - Empty Result**: This router step checks if the result is empty.
    *   **Empty result**: If the retrieved user data is empty (not(//User/User/node())), the process ends.
    *   **Valid result**: If the retrieved user data is not empty, the process continues.
4.  **Process Call - Process Employees**: Calls the 'Process Employee' sub-process for each user in the received payload.
5.  **Filter**: Filter step to be used in combination with the looping.
6.  **Save Body**: This Content Modifier step saves the current body to a property named 'savedBody'.
    | Property Name | Action | Type       | Value                     | Datatype        |
    | :------------ | :----- | :--------- | :------------------------ | :-------------- |
    | savedBody     | Create | expression | `${property.savedBody}${body}` | `java.lang.String` |
7. **Set Body**: Sets the body, required by the looping construct.
8.  **End**: The process concludes successfully.

### Process Employee

This process is responsible for transforming and processing individual employee data.

![Process Employee](https://i.imgur.com/wJ0y5c6.png)

1.  **Start**: The process begins with a Start Event.
2.  **Message Mapping - filter dates**: Transforms the incoming payload to filter the dates, by using the message mapping 'MM_SFSF_Filter_Nodes_Dates'.
3.  **XSLT Mapping - exclude filter_roles**: Transforms the incoming payload to exclude roles by using the xslt mapping 'excludefilter_roles'.
4.  **XSLT Mapping - append roles**: Transforms the incoming payload to append roles by using the xslt mapping 'append_roles'.
5. **Splitter**: Splits the payload to enable processing each employee individually.
6. **Content Modifier - username, firstName, lastName, statute**: This step prepares the payload for the external API call, the actual processing is done in the ProcessDirect API call.
7. **Request Reply**: This step calls the ProcessDirect API to process the user, this is a call to a 3rd party system.
8.  **XSLT Mapping - append filter_username, filter_firstName, filter_lastName, filter_statute**: Transforms the incoming payload to append the user data by using the xslt mapping 'appendfilter_username'.
9. **Gather**: Gathers all the different parts of the payload back together.
10. **Router  debug**: This Router step determines if the logging should be done based on the 'debuggingEnabled' property.
    *   **Yes**: If the `debuggingEnabled` property is set to 'true', the "Send" step is called.
    *   **No**: If the `debuggingEnabled` property is not set to 'true', the process continues.
11. **Send**: This step sends the invalid users to the SFTP server, this is a call to a 3rd party system.
12. **End**: The process concludes successfully.

### Invalid Log

This process logs the invalid entries to sftp.

![Invalid Log](https://i.imgur.com/a1rI43T.png)

1.  **Start**: The process begins with a Start Event.
2. **Content Modifier**: Sets the filename for the invalid log.
    | Property Name | Action | Type       | Value                            | Datatype        |
    | :------------ | :----- | :--------- | :--------------------------------- | :-------------- |
    | filename     | Create | expression | `invalid_${date:now:yyyyMMdd}.xml` |     |
3.  **XSLT Mapping Invalid entries**: Transforms the payload to format the log using the 'xmlinvalidlogging' XSLT mapping.
4.  **Process Call - Log**: Calls the 'Log' sub-process to handle the logging.
5.  **End**: The process concludes successfully.

### Read LastRun

This process retrieves the last run timestamp.

![Read LastRun](https://i.imgur.com/4WvJjFv.png)

1.  **Start**: The process begins with a Start Event.
2. **Content Modifier**: Placeholder for future use, no properties are set.
3. **End**: The process concludes successfully.

### Extract Error Process

This process extracts and logs errors.

![Extract Error Process](https://i.imgur.com/7f4k7jU.png)

1.  **Start**: The process begins with a Start Event.
2.  **CM - filename / stacktrace**: This Content Modifier step sets the filename, exception message, and stacktrace for logging.
    
    | Property Name              | Action | Type       | Value                 | Datatype        |
    | :------------------------- | :----- | :--------- | :-------------------- | :-------------- |
    | exception_message_to_escape    | Create | expression | `exception.message`   | `java.lang.String` |
    | filename                  | Create | expression | `general_error_${date:now:yyyyMMdd}.xml` |       |
    | exception_to_escape        | Create | expression | `exception.stacktrace` |  `java.lang.String`|
3.  **XSLT Mapping Exception**: Transforms the payload to a suitable format using the 'Exception' XSLT mapping.
4.  **Process Call - Log**: Calls the 'Log' sub-process to handle logging the error.
5.  **End**: The process concludes successfully.

### Write LastRun

This process writes the current timestamp as the last run timestamp.

![Write LastRun](https://i.imgur.com/f8z1yU0.png)

1.  **Start**: The process begins with a Start Event.
2.  **Content Modifier**: This Content Modifier step sets the current timestamp in the 'thisRun' property.
    
    | Property Name | Action | Type       | Value                      | Datatype |
    | :------------ | :----- | :--------- | :------------------------- | :------- |
    | thisRun       | Create | expression | `${date:now:YYYY-MM-dd'T'HH:mm:ss.SSSXXX}` |          |
3.  **Write LastSync**: This step is a placeholder for writing the last run timestamp to a persistent store (e.g., Data Store).
4.  **End**: The process concludes successfully.

### Log

This process handles the logging of messages via email and SFTP.

![Log](https://i.imgur.com/G7QZ2Jg.png)

1.  **Start**: The process begins with a Start Event.
2. **Content Modifier**: This step extracts email properties from the XML payload using XPath expressions.
    
    | Property Name | Action | Type  | Value                      | Datatype        |
    | :------------ | :----- | :---- | :------------------------- | :-------------- |
    | mail_from    | Create | xpath | `/mailOutput/from/text()`    | `java.lang.String`|
    | mail_title   | Create | xpath | `/mailOutput/title/text()`  | `java.lang.String`|
    | mail_to      | Create | xpath | `/mailOutput/recipients/text()` | `java.lang.String`|
    | mail_body    | Create | xpath | `/mailOutput/body/text()`    | `java.lang.String`|
3.  **Mail**: This Router step checks if email notifications are enabled.
    *   **Yes**: If the `EmailNotificationsEnabled` property is set to 'true', the "Send Mail" step is called.
    *   **No**: If the `EmailNotificationsEnabled` property is not set to 'true', the process continues to SFTP router.
4.  **SFTP**: This Router step checks if SFTP logging is enabled.
    *   **Yes**: If the `SFTPLoggingEnabled` property is set to 'true', the "Send SFTP" step is called.
    *   **No**: If the `SFTPLoggingEnabled` property is not set to 'true', the process ends.
5.  **Send Mail**: This step sends an email notification using the 'Mail-Exceptions' adapter to the receiver 'Receiver-Email', this is a call to a 3rd party system.
6.  **Send SFTP**: This step sends the log file to the SFTP server using the 'SFTP-Error' adapter to the receiver 'SFTP-Log', this is a call to a 3rd party system.
7.  **End**: The process concludes successfully.
