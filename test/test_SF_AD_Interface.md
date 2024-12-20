```markdown
# Integration Flow Technical Specification

This document provides a detailed technical specification for the integration flow developed in SAP Cloud Integration. It outlines the different processes involved, their initiation points, and step-by-step descriptions of each process. This document aims to provide a clear understanding of the integration flow's logic for new team members.

## Process Overview

The integration flow consists of multiple interconnected processes, each designed for a specific purpose. These processes work together to achieve the overall integration goal. Below is a simplified view of the main process and its sub-processes:

```
                                   +--------------------------+
                                   |     Integration Process   |
                                   +-----------+--------------+
                                               |
                  +---------------------------v----------------------------+
                  |                 Router - all / selected users           |
                  +-----------------------+--------+------------------------+
                                          |        |
                 +----------+             |        |           +-----------------------+
                 |   All    |             |        |           |    Selected Users     |
                 | Employees|             |        |           | (Local Integration) |
                 +----------+      +------v---+    +----------v-------+
                       |          | Local Looping |         |   Request Reply - SF    |
                       |          | Integration  |         |      +----------+       |
                       |          |   Process    |         |      |   Filter   |      |
                       +----------v-------+      +----------v-------+       |
                       |  Request Reply  |       | Save Body  |      | Save Body |
                       |   SF All users   |       +-----+---+      +-----v----+
                       | +--------------+    +----v-----+   |          |
                       | | Filter        |    |  Set Body    |          |
                       | +--------------+    |    +---+    |     +-----v----+
                       +-----v------------+    +----v-----+    | Process Call - Process Employees  |
                               |                  |            +-----v----+
                               +------------------+------------------+
                                                |  Router - Empty Result  |
                                                +--------+--------+
                                                         |        |
                                                         |        |  
                                        +----------------v---+     +--------------+
                                        |   Valid result       |     | Empty result |
                                        +--------------------+     +--------------+
                                                        |                                  
                                                        |   
                                       +-----------------v-------------------+
                                       |       Router - Log Invalid Entries    |
                                       +-----------------+-------------------+
                                                       |            |
                         +------------------------v---------+   +---------------v--------+
                         |        Invalid Entries        |   |    No Invalid Entries      |
                         +-------------------------------+   +-----------------------------+
                                      |                              |
                                      v                              v
                    +-------------------+           +------------------------------+
                    |     Invalid Log   |           |          Write LastRun        |
                    +-------------------+           +------------------------------+
```

## Integration Flow Initiation

The main integration flow, named **Integration Process**, can be triggered in the following way:

*   **Start Timer:** The integration flow is scheduled to start automatically based on a configured timer. This mechanism allows for periodic execution of the integration.

## Detailed Process Description

This section provides a detailed explanation of each process and its steps, including content modifiers and external system calls.

### Process: Integration Process

This is the main process that orchestrates the entire integration flow.

1.  **Start Timer:** The process starts based on a configured timer.
2.  **Read LastRun:** This step calls the sub-process `Read LastRun` to retrieve the timestamp of the last successful run.
3.  **Configure Flow:** This step uses a `ContentModifier` to set various properties used throughout the flow.

    | Property Name           | Action | Type     | Value                     | Description                                                                  |
    | :---------------------- | :----- | :------- | :------------------------ | :--------------------------------------------------------------------------- |
    | RolesToExclude          | Create | constant | `{{Roles_To_Exclude}}`    | Comma-separated list of roles to exclude.                                   |
    | queryusernames          | Create | constant | `{{SF_Query_usernames}}`    |  Query for specific usernames.                                            |
    | processDirectUrl        | Create | constant | `{{ProcessDirect_Url}}`      | URL for Process Direct endpoint.                                        |
    | debuggingEnabled        | Create | constant | `{{Debugging_Enabled}}`   | Flag to enable debug logging.                                             |
    | emailFrom               | Create | constant | `{{Email_From}}`          | Sender email address for notifications.                                    |
    | EmailNotificationsEnabled| Create | constant | `{{Email_Notifications_Enabled}}` | Flag to enable email notifications.                                      |
    | SFTPLoggingEnabled      | Create | constant | `{{SFTP_Logging_Enabled}}`| Flag to enable SFTP logging.                                              |
    | emailRecipients         | Create | constant | `{{Email_Recipients}}`    | Comma-separated list of recipient email addresses for notifications.      |
    | processAllEmployees     | Create | constant | `{{All_Employees}}`   |  Flag to process all employees.          |
4.  **Router - all / selected users:**  This router step determines the flow path based on whether to process all employees or only selected ones, based on the property `processAllEmployees`.
    *   **All Employees:** If the `processAllEmployees` property is set to `true`, it calls the `Local Looping Integration Process All Users` subprocess to retrieve information for all users.
    *  **Selected Employees:** If the `processAllEmployees` property is not `true`, it calls the `Local Integration Process Selected Users` subprocess to retrieve information for only the selected users.
5.  **Router - Log Invalid Entries:** This router step checks if there are errors in the `root/error` node
    *  **Invalid Entries:** If there are invalid entries, it calls the `Invalid Log` subprocess to log the error.
    *  **No Invalid Entries:** If there are no invalid entries it will continue.
6.  **Write LastRun:** This step calls the `Write LastRun` subprocess to save the current run's timestamp.
7.  **End:** The process ends.

### Process: Invalid Log

This process is responsible for logging invalid entries.

1.  **Start:** The process starts when called by the `Integration Process`.
2.  **Content Modifier:** This step sets the filename for the invalid log file using an expression.

    | Property Name | Action | Type     | Value                     | Description                                   |
    | :------------ | :----- | :------- | :------------------------ | :-------------------------------------------- |
    | filename      | Create | expression | `invalid_${date:now:yyyyMMdd}.xml` | Generates a filename with the current date. |
3.  **XSLT Mapping Invalid entries:** This step transforms the input XML using the `xmlinvalidlogging` XSLT script.
4.  **Process Call - Log:** This step calls the `Log` process to handle the logging of the invalid data.
5.  **End:** The process ends.

### Process: Local Integration Process Selected Users

This process handles the logic for processing selected users.

1.  **Start:** The process starts.
2. **Request Reply - SF:** This step calls a 3rd party system, **SuccessFactors**, via the `SuccessFactorsODataSingleUser` adapter to retrieve data for selected users.
3. **Save Body:** This step adds the current body to the `savedBody` property.

    | Property Name | Action | Type     | Value                       | Description                               |
    | :------------ | :----- | :------- | :-------------------------- | :---------------------------------------- |
    | savedBody     | Create | expression | `${property.savedBody}${body}` | Appends the current message body to the savedBody property. |
4.  **Filter:** This step acts as a filter.
5.  **Process Call - Process Employees:** This step calls the `Process Employee` sub-process to process each individual user.
6.  **Router - Empty Result:** This router checks if the result of the API call is empty
    *   **Empty result:** If the result is empty, the process ends.
    *  **Valid result:** If there is a result, the process continues to the end.
7. **Set Body:** This step is used to set the body for the end of the flow
8.  **End:** The process ends.

### Process: Local Looping Integration Process All Users

This process handles the logic for processing all users.

1.  **Start:** The process starts.
2. **Request Reply - SF All users:** This step calls a 3rd party system, **SuccessFactors**, via the `SuccessFactorsODataAllUsers` adapter to retrieve data for all users.
3.  **Save Body:** This step adds the current body to the `savedBody` property.

    | Property Name | Action | Type     | Value                       | Description                               |
    | :------------ | :----- | :------- | :-------------------------- | :---------------------------------------- |
    | savedBody     | Create | expression | `${property.savedBody}${body}` | Appends the current message body to the savedBody property. |
4.  **Filter:** This step acts as a filter.
5.  **Process Call - Process Employees:** This step calls the `Process Employee` sub-process to process each individual user.
6. **Router - Empty Result:** This router checks if the result of the API call is empty
    *   **Empty result:** If the result is empty, the process ends.
    *  **Valid result:** If there is a result, the process continues to the end.
7. **Set Body:** This step is used to set the body for the end of the flow
8.  **End:** The process ends.

### Process: Process Employee

This process handles the individual processing of each employee.

1.  **Start:** The process starts.
2.  **Message Mapping - filter dates:** This step performs a message mapping to filter dates using the `MM_SFSF_Filter_Nodes_Dates` mapping.
3.  **XSLT Mapping - exclude filter_roles:** This step transforms the input XML by excluding roles based on the `excludefilter_roles` XSLT script.
4.  **XSLT Mapping - append roles:** This step transforms the input XML by appending roles based on the `append_roles` XSLT script.
5.  **Splitter:** Splits the message into separate messages for each user.
6.   **Content Modifier - username, firstName, lastName, statute:** This step sets the content modifier.
7.  **Request Reply:** This step calls a 3rd party system via the `ProcessDirect-EDS-AzureAD` adapter, sending user data for further processing.
8.  **XSLT Mapping - append filter_username, filter_firstName, filter_lastName, filter_statute:** This step transforms the input XML by appending filter data using the `appendfilter_username` XSLT script.
9.  **Gather:** This step gathers the messages.
10. **Router  debug:** This router checks if the debugging flag is enabled.
    *  **Yes:** If debugging is enabled, it continues with logging.
    *  **No:** If debugging is not enabled, it ends.
11. **Send:** This step calls a 3rd party system via the `SFTP-Process-Employee-InvalidADUsers` adapter to send the log for users that could not be created in Azure AD.
12. **End:** The process ends.

### Process: Extract Error Process

This process handles the extraction and logging of errors.

1.  **Start:** The process starts.
2.  **CM - filename / stacktrace:** This `ContentModifier` step creates properties for filename and exception information.

    | Property Name             | Action | Type     | Value                  | Description                                          |
    | :------------------------ | :----- | :------- | :--------------------- | :--------------------------------------------------- |
    | exception_message_to_escape | Create | expression | `exception.message`      | Captures the exception message.                      |
    | filename                  | Create | expression | `general_error_${date:now:yyyyMMdd}.xml`| Generates a filename with the current date.     |
    | exception_to_escape       | Create | expression | `exception.stacktrace`     | Captures the exception stacktrace.                  |
3.  **XSLT Mapping Exception:** This step transforms the input XML using the `Exception` XSLT script.
4.  **Process Call - Log:** This step calls the `Log` process to handle the error logging.
5.  **End:** The process ends.

### Process: Write LastRun

This process handles the writing of the last successful run timestamp.

1.  **Start:** The process starts.
2.  **Content Modifier:** This step creates a property to store the current timestamp.

    | Property Name | Action | Type     | Value                           | Description                               |
    | :------------ | :----- | :------- | :------------------------------ | :---------------------------------------- |
    | thisRun       | Create | expression | `${date:now:YYYY-MM-dd'T'HH:mm:ss.SSSXXX}` |  Generates the current timestamp in a specific format.  |
3.  **Write LastSync:** This step handles the writing of the timestamp (implementation not detailed in XML).
4.  **End:** The process ends.

### Process: Read LastRun

This process handles the reading of the last successful run timestamp (implementation not detailed in XML).

1.  **Start:** The process starts.
2.  **Content Modifier:** This step sets the content modifier (no properties to set).
3.  **End:** The process ends.

### Process: Log

This process handles the logging of messages.

1.  **Start:** The process starts.
2.  **Content Modifier:** This step extracts relevant information from the message body to properties.

    | Property Name | Action | Type | Value                      | Description |
    | :------------ | :----- | :--- | :------------------------- | :---------- |
    | mail_from     | Create | xpath| `/mailOutput/from/text()` | Extracts the 'from' email address from the mailOutput XML node. |
    | mail_title    | Create | xpath |`/mailOutput/title/text()`  | Extracts the email subject from the mailOutput XML node. |
    | mail_to     | Create | xpath | `/mailOutput/recipients/text()` | Extracts the recipient email address from the mailOutput XML node. |
    | mail_body    | Create | xpath | `/mailOutput/body/text()`| Extracts the email body from the mailOutput XML node. |
3.  **Mail:** This router step determines if an email notification is required based on the value of property `EmailNotificationsEnabled`
    * **Yes:** If email notifications are enabled, the process sends an email.
    * **No:** If email notifications are not enabled, the process ends.
4. **SFTP:** This router step determines if SFTP logging is required based on the value of property `SFTPLoggingEnabled`
    * **Yes:** If SFTP logging is enabled, the process will send the file to SFTP.
    * **No:** If SFTP logging is not enabled, the process ends.
5.  **Send Mail:** This step calls a 3rd party system via the `Mail-Exceptions` adapter to send an email notification.
6.  **Send SFTP:** This step calls a 3rd party system via the `SFTP-Error` adapter to send the log file to an SFTP server.
7.  **End:** The process ends.

This detailed specification should provide a clear understanding of the integration flow. Please let me know if you have any further questions.
```
