```markdown
## Integration Flow Technical Specification

This document provides a technical specification for the integration flows described in the provided XML file. It details each process, its steps, and configurations to aid new team members in understanding the integration logic.

### Process Overview

This section provides a high-level overview of each integration process defined in the XML file.

| Process Name                       | Description                                                                 |
|------------------------------------|-----------------------------------------------------------------------------|
| Invalid Log                        | Handles logging of invalid entries encountered during processing.              |
| Local Integration Process Selected Users | A local integration process to handle selected users.                      |
| Integration Process                | The main integration process orchestrating various sub-processes.             |
| Read LastRun                       | Reads the timestamp of the last successful run of the integration flow.      |
| Local Looping Integration Process All Users | A local integration process to loop through and process all users.         |
| Process Employee                   | Processes individual employee data.                                        |
| Extract Error Process              | Extracts error details for logging and handling.                             |
| Write LastRun                      | Writes the timestamp of the current run as the last successful run.          |
| Log                                | A utility process for logging information, potentially via email and SFTP. |

### Integration Flow Initiation

This section describes how each integration flow is initiated.

*   **Integration Process**:
    *   **Start Timer**: This integration flow is initiated by a timer event, meaning it is scheduled to run automatically at predefined intervals.

*   **Local Integration Process Selected Users**:
    *   **Start**: This integration flow is initiated by an incoming message. It is designed to be called by another integration flow to process selected users.

*   **Invalid Log**:
    *   **Start**: This integration flow is initiated by an incoming message. It is designed to be called by another integration flow to log invalid entries.

*   **Read LastRun**:
    *   **Start**: This integration flow is initiated by an incoming message. It is designed to be called by another integration flow to retrieve the last run timestamp.

*   **Local Looping Integration Process All Users**:
    *   **Start**: This integration flow is initiated by an incoming message. It is designed to be called by another integration flow to process all users in a looping manner.

*   **Process Employee**:
    *   **Start**: This integration flow is initiated by an incoming message. It is designed to be called by another integration flow to process individual employee data.

*   **Extract Error Process**:
    *   **Start**: This integration flow is initiated by an incoming message. It is designed to be called by another integration flow to handle and log errors.

*   **Write LastRun**:
    *   **Start**: This integration flow is initiated by an incoming message. It is designed to be called by another integration flow to write the last run timestamp.

*   **Log**:
    *   **Start**: This integration flow is initiated by an incoming message. It is a utility process designed for logging purposes, triggered by other flows needing to log information.

### Detailed Process Description

This section provides a detailed description of each process and its steps.

#### 1. Invalid Log Process

**Process Name:** Invalid Log

**Process Overview:** This process is responsible for logging invalid data entries. It receives data, formats it using an XSLT mapping, and then calls a generic logging process.

**Process Flow:**

```
Start --> Content Modifier --> XSLT Mapping Invalid entries --> Process Call - Log --> End
```

**Step-by-step Description:**

1.  **Start**:
    *   **Type**: StartEvent
    *   **Description**: The process begins here upon receiving an incoming message containing invalid data.

2.  **Content Modifier**:
    *   **Type**: ContentModifier
    *   **Description**: This step sets a property named `filename` to define the name of the log file.
    *   **Properties Table:**

        | Action | Type       | Name     | Value                        | Datatype | Default |
        |--------|------------|----------|------------------------------|----------|---------|
        | Create | expression | filename | `invalid_${date:now:yyyyMMdd}.xml` |          |         |

3.  **XSLT Mapping Invalid entries**:
    *   **Type**: XSLTMapping
    *   **Description**: This step transforms the incoming message payload using the XSLT script resource `xmlinvalidlogging`. This mapping likely formats the invalid data into a standardized XML structure suitable for logging.
    *   **XSLT Script Resource**: `xmlinvalidlogging`

4.  **Process Call - Log**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Log" (Process_75333029). It passes the transformed XML message to the "Log" process for actual logging.
    *   **Called Process**: Log

5.  **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends here after logging the invalid entry.

#### 2. Local Integration Process Selected Users

**Process Name:** Local Integration Process Selected Users

**Process Overview:** This process retrieves user data from SuccessFactors (SF) for selected users, filters the data, and calls another process to process the employees.

**Process Flow:**

```
Start --> Request Reply - SF --> Router - Empty Result --> Process Call - Process Employees --> Filter --> Save Body --> Set Body --> End
                                  |
                                  +--> End
```

**Step-by-step Description:**

1.  **Start**:
    *   **Type**: StartEvent
    *   **Description**: The process starts upon receiving an incoming message, likely containing criteria for selecting users.

2.  **Request Reply - SF**:
    *   **Type**: API_CALL
    *   **Description**: This step calls the SuccessFactors (SF) system to retrieve user data. It uses the adapter `SuccessFactorsODataSingleUser` and communicates with the receiver `SF`. This is a **3rd party system call** to SuccessFactors.
    *   **Adapter**: `SuccessFactorsODataSingleUser`
    *   **Receiver**: `SF`

3.  **Router - Empty Result**:
    *   **Type**: router
    *   **Description**: This step checks if the response from SuccessFactors contains user data.
    *   **Conditions**:
        *   **Empty result**: Condition `not(//User/User/node())`. If the SF response does not contain any User nodes, the flow proceeds to the "End" step, indicating no users were found based on the selection criteria.
        *   **Valid result**: Default condition. If the SF response contains user data, the flow proceeds to "Process Call - Process Employees".

4.  **Process Call - Process Employees**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Process Employee" (Process_75333156) to process the retrieved user data.
    *   **Called Process**: Process Employee

5.  **Filter**:
    *   **Type**: *Unknown - incomplete step definition in XML, assuming a filter based on context*.
    *   **Description**:  *Step definition is incomplete in XML. Assuming this step is meant to filter the data received from "Process Employees" based on some criteria.*

6.  **Save Body**:
    *   **Type**: ContentModifier
    *   **Description**: This step appends the current message body to a property named `savedBody`. This might be used to accumulate data across multiple iterations, although the context in this specific flow is unclear without looping.
    *   **Properties Table:**

        | Action | Type       | Name      | Value                  | Datatype        | Default |
        |--------|------------|-----------|------------------------|-----------------|---------|
        | Create | expression | savedBody | `${property.savedBody}${body}` | `java.lang.String` |         |

7.  **Set Body**:
    *   **Type**: *Unknown - incomplete step definition in XML, assuming setting/modifying body*.
    *   **Description**: *Step definition is incomplete in XML. Assuming this step is meant to set or modify the message body based on the processing done in previous steps.*

8.  **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends here, either after processing users or if no users were found.

#### 3. Integration Process

**Process Name:** Integration Process

**Process Overview:** This is the main integration flow. It reads the last run timestamp, configures flow properties, processes users (both all and selected users), handles invalid logs, and writes the current run timestamp as the last run.

**Process Flow:**

```
Start Timer --> Read LastRun --> Configure Flow --> Router - all / selected users --> Looping Process Call - All Users --> Router - Log Invalid Entries --> Write LastRun --> End
                                                     |                                       ^
                                                     +--> Process Call - Selected Users --------+
                                                     |
                                                     +--> Invalid Log
```

**Step-by-step Description:**

1.  **Start Timer**:
    *   **Type**: StartEvent
    *   **Description**: The process starts based on a timer schedule. This is the entry point of the main integration flow.

2.  **Read LastRun**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Read LastRun" (Process_75333694) to retrieve the timestamp of the last successful execution.
    *   **Called Process**: Read LastRun

3.  **Configure Flow**:
    *   **Type**: ContentModifier
    *   **Description**: This step sets various properties used throughout the integration flow using constants defined as placeholders (e.g., `{{Roles_To_Exclude}}`, `{{SF_Query_usernames}}`). These placeholders are expected to be replaced with actual values during deployment or configuration.
    *   **Properties Table:**

        | Action | Type     | Name                  | Value                      | Datatype | Default |
        |--------|----------|-----------------------|----------------------------|----------|---------|
        | Create | constant | RolesToExclude        | `{{Roles_To_Exclude}}`       |          |         |
        | Create | constant | queryusernames        | `{{SF_Query_usernames}}`       |          |         |
        | Create | constant | processDirectUrl      | `{{ProcessDirect_Url}}`      |          |         |
        | Create | constant | debuggingEnabled      | `{{Debugging_Enabled}}`      |          |         |
        | Create | constant | emailFrom             | `{{Email_From}}`             |          |         |
        | Create | constant | EmailNotificationsEnabled | `{{Email_Notifications_Enabled}}` |          |         |
        | Create | constant | SFTPLoggingEnabled      | `{{SFTP_Logging_Enabled}}`     |          |         |
        | Create | constant | emailRecipients       | `{{Email_Recipients}}`       |          |         |
        | Create | constant | processAllEmployees   | `{{All_Employees}}`          |          |         |

4.  **Router - all / selected users**:
    *   **Type**: router
    *   **Description**: This step routes the flow based on whether to process all employees or only selected users.
    *   **Conditions**:
        *   **All Employees**: Condition `${property.processAllEmployees} = 'true'`. If the property `processAllEmployees` is set to 'true', the flow proceeds to "Looping Process Call - All Users".
        *   **Selected Employees**: Default condition. If `processAllEmployees` is not 'true', the flow proceeds to "Process Call - Selected Users".

5.  **Looping Process Call - All Users**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Local Looping Integration Process All Users" (Process_75333180) to process all users.
    *   **Called Process**: Local Looping Integration Process All Users

6.  **Process Call - Selected Users**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Local Integration Process Selected Users" (Process_75333176) to process selected users.
    *   **Called Process**: Local Integration Process Selected Users

7.  **Router - Log Invalid Entries**:
    *   **Type**: router
    *   **Description**: This step checks for invalid entries in the message payload.
    *   **Conditions**:
        *   **Invalid Entries**: Condition `count(/root/error) > 0`. If the XML payload contains one or more `<error>` elements under the root, the flow proceeds to "Invalid Log".
        *   **No Invalid Entries**: Default condition. If no invalid entries are found, the flow proceeds to "Write LastRun".

8.  **Invalid Log**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Invalid Log" (Process_75333867) to log any invalid entries found.
    *   **Called Process**: Invalid Log

9.  **Write LastRun**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Write LastRun" (Process_75333701) to record the current run as the last successful run.
    *   **Called Process**: Write LastRun

10. **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends here after completing all steps, including processing users, logging invalid entries, and updating the last run timestamp.

#### 4. Read LastRun Process

**Process Name:** Read LastRun

**Process Overview:** This process is designed to read and return the last successful run timestamp. The implementation details are minimal in the provided XML, suggesting it might be reading from a persistent store not defined in this snippet.

**Process Flow:**

```
Start --> Content Modifier --> End
```

**Step-by-step Description:**

1.  **Start**:
    *   **Type**: StartEvent
    *   **Description**: The process starts upon receiving an incoming message, triggered by the "Integration Process".

2.  **Content Modifier**:
    *   **Type**: ContentModifier
    *   **Description**: *Step definition is incomplete in XML. This step is likely intended to read the last run timestamp from a persistent store and set it as a property or in the message body.  However, no properties are defined in the XML.*

3.  **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends after attempting to read the last run timestamp.

#### 5. Local Looping Integration Process All Users

**Process Name:** Local Looping Integration Process All Users

**Process Overview:** This process retrieves all users from SuccessFactors in a looping manner, likely to handle large datasets, filters the data, and processes each employee.

**Process Flow:**

```
Start --> Request Reply - SF All users --> Router - Empty Result --> Process Call - Process Employees --> Filter --> Save Body --> Set Body --> End
                                          |
                                          +--> End
```

**Step-by-step Description:**

1.  **Start**:
    *   **Type**: StartEvent
    *   **Description**: The process starts upon receiving an incoming message, triggered by the "Integration Process".

2.  **Request Reply - SF All users**:
    *   **Type**: API_CALL
    *   **Description**: This step calls the SuccessFactors (SF) system to retrieve data for all users. It uses the adapter `SuccessFactorsODataAllUsers` and communicates with the receiver `SF`. This is a **3rd party system call** to SuccessFactors.
    *   **Adapter**: `SuccessFactorsODataAllUsers`
    *   **Receiver**: `SF`

3.  **Router - Empty Result**:
    *   **Type**: router
    *   **Description**: This step checks if the response from SuccessFactors contains user data.
    *   **Conditions**:
        *   **Empty result**: Condition `not(//User/User/node())`. If the SF response does not contain any User nodes, the flow proceeds to the "End" step, indicating no users were found.
        *   **Valid result**: Default condition. If the SF response contains user data, the flow proceeds to "Process Call - Process Employees".

4.  **Process Call - Process Employees**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Process Employee" (Process_75333156) to process the retrieved user data.
    *   **Called Process**: Process Employee

5.  **Filter**:
    *   **Type**: *Unknown - incomplete step definition in XML, assuming a filter based on context*.
    *   **Description**: *Step definition is incomplete in XML. Assuming this step is meant to filter the data received from "Process Employees" based on some criteria.*

6.  **Save Body**:
    *   **Type**: ContentModifier
    *   **Description**: This step appends the current message body to a property named `savedBody`. This is likely used for accumulating responses across multiple pages of user data if SuccessFactors returns data in pages.
    *   **Properties Table:**

        | Action | Type       | Name      | Value                  | Datatype        | Default |
        |--------|------------|-----------|------------------------|-----------------|---------|
        | Create | expression | savedBody | `${property.savedBody}${body}` | `java.lang.String` |         |

7.  **Set Body**:
    *   **Type**: *Unknown - incomplete step definition in XML, assuming setting/modifying body*.
    *   **Description**: *Step definition is incomplete in XML. Assuming this step is meant to set or modify the message body based on the processing done in previous steps.*

8.  **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends here, either after processing users or if no users were found.

#### 6. Process Employee

**Process Name:** Process Employee

**Process Overview:** This process handles the processing of individual employee data. It performs message and XSLT mappings to filter and transform the data, calls another integration flow to further process the data, and includes debugging and logging capabilities.

**Process Flow:**

```
Start --> Message Mapping - filter dates --> XSLT Mapping - exclude filter_roles --> XSLT Mapping - append roles --> Splitter --> Content Modifier - username, firstName, lastName, statute --> Request Reply --> Gather --> Router  debug --> Send --> End
                                                                                                                                     |
                                                                                                                                     +--> End
```

**Step-by-step Description:**

1.  **Start**:
    *   **Type**: StartEvent
    *   **Description**: The process starts upon receiving an incoming message containing employee data, triggered by the "Local Looping Integration Process All Users" or "Local Integration Process Selected Users".

2.  **Message Mapping - filter dates**:
    *   **Type**: MessageMapping
    *   **Description**: This step performs a message mapping using the resource `MM_SFSF_Filter_Nodes_Dates`. This mapping likely filters or transforms date-related fields in the employee data.
    *   **Mapping Resource**: `MM_SFSF_Filter_Nodes_Dates`

3.  **XSLT Mapping - exclude filter_roles**:
    *   **Type**: XSLTMapping
    *   **Description**: This step transforms the message payload using the XSLT script resource `excludefilter_roles`. This mapping likely excludes roles based on predefined filter criteria.
    *   **XSLT Script Resource**: `excludefilter_roles`

4.  **XSLT Mapping - append roles**:
    *   **Type**: XSLTMapping
    *   **Description**: This step transforms the message payload using the XSLT script resource `append_roles`. This mapping likely appends or adds roles to the employee data.
    *   **XSLT Script Resource**: `append_roles`

5.  **Splitter**:
    *   **Type**: *Unknown - incomplete step definition in XML, assuming a splitter*.
    *   **Description**: *Step definition is incomplete in XML. Assuming this step splits the incoming message into multiple messages, possibly to process each employee record individually if the input contains a list of employees.*

6.  **Content Modifier - username, firstName, lastName, statute**:
    *   **Type**: ContentModifier
    *   **Description**: *Step definition is incomplete in XML. This step is likely intended to extract or set properties related to username, first name, last name, and statute. However, no properties are defined in the XML.*

7.  **Request Reply**:
    *   **Type**: API_CALL
    *   **Description**: This step calls another system using the `ProcessDirect-EDS-AzureAD` adapter and receiver `Receiver-ProcessDirect-EDS-AzureAD`. This is a **3rd party system call** to a system identified as "ProcessDirect-EDS-AzureAD", likely to further process or validate the employee data against Azure Active Directory.
    *   **Adapter**: `ProcessDirect-EDS-AzureAD`
    *   **Receiver**: `Receiver-ProcessDirect-EDS-AzureAD`

8.  **XSLT Mapping - append filter_username, filter_firstName, filter_lastName, filter_statute**:
    *   **Type**: XSLTMapping
    *   **Description**: This step transforms the message payload using the XSLT script resource `appendfilter_username`. This mapping likely appends filter criteria related to username, first name, last name, and statute to the message.
    *   **XSLT Script Resource**: `appendfilter_username`

9.  **Gather**:
    *   **Type**: *Unknown - incomplete step definition in XML, assuming a gather step*.
    *   **Description**: *Step definition is incomplete in XML. Assuming this step gathers or aggregates messages, likely to combine results after a splitter step.*

10. **Router  debug**:
    *   **Type**: router
    *   **Description**: This step checks if debugging is enabled.
    *   **Conditions**:
        *   **Yes**: Condition `${property.debuggingEnabled} = 'true'`. If the property `debuggingEnabled` is 'true', the flow proceeds to "Send" to log debug information.
        *   **No**: Default condition. If debugging is not enabled, the flow proceeds directly to "End".

11. **Send**:
    *   **Type**: API_CALL
    *   **Description**: This step sends data to an SFTP server for logging invalid AD users. It uses the adapter `SFTP-Process-Employee-InvalidADUsers` and receiver `SFTP-Log`. This is a **3rd party system call** to an SFTP server.
    *   **Adapter**: `SFTP-Process-Employee-InvalidADUsers`
    *   **Receiver**: `SFTP-Log`

12. **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends here after processing the employee data and potentially logging debug information.

#### 7. Extract Error Process

**Process Name:** Extract Error Process

**Process Overview:** This process extracts error details from exceptions, formats them using XSLT mapping, and then calls the generic logging process.

**Process Flow:**

```
Start --> CM - filename / stacktrace --> XSLT Mapping Exception --> Process Call - Log --> End
```

**Step-by-step Description:**

1.  **Start**:
    *   **Type**: StartEvent
    *   **Description**: The process starts upon encountering an error in another integration flow, designed to handle exceptions.

2.  **CM - filename / stacktrace**:
    *   **Type**: ContentModifier
    *   **Description**: This step sets properties to extract exception message, create a filename for the error log, and extract the stacktrace.
    *   **Properties Table:**

        | Action | Type       | Name                     | Value                        | Datatype        | Default |
        |--------|------------|--------------------------|------------------------------|-----------------|---------|
        | Create | expression | exception_message_to_escape | `exception.message`            | `java.lang.String` |         |
        | Create | expression | filename                 | `general_error_${date:now:yyyyMMdd}.xml` |          |         |
        | Create | expression | exception_to_escape      | `exception.stacktrace`         | `java.lang.String` |         |

3.  **XSLT Mapping Exception**:
    *   **Type**: XSLTMapping
    *   **Description**: This step transforms the extracted exception details using the XSLT script resource `Exception`. This mapping likely formats the error information for logging.
    *   **XSLT Script Resource**: `Exception`

4.  **Process Call - Log**:
    *   **Type**: CallLocalProcess
    *   **Description**: This step calls the local integration process named "Log" (Process_75333029) to perform the actual logging of the error details.
    *   **Called Process**: Log

5.  **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends here after logging the error details.

#### 8. Write LastRun Process

**Process Name:** Write LastRun

**Process Overview:** This process writes the current timestamp as the last successful run timestamp.

**Process Flow:**

```
Start --> Content Modifier --> Write LastSync --> End
```

**Step-by-step Description:**

1.  **Start**:
    *   **Type**: StartEvent
    *   **Description**: The process starts upon being called by the "Integration Process" after successful completion of user processing.

2.  **Content Modifier**:
    *   **Type**: ContentModifier
    *   **Description**: This step sets a property named `thisRun` to the current timestamp in ISO 8601 format.
    *   **Properties Table:**

        | Action | Type       | Name    | Value                                 | Datatype | Default |
        |--------|------------|---------|---------------------------------------|----------|---------|
        | Create | expression | thisRun | `${date:now:YYYY-MM-dd'T'HH:mm:ss.SSSXXX}` |          |         |

3.  **Write LastSync**:
    *   **Type**: *Unknown - incomplete step definition in XML, assuming writing to persistent store*.
    *   **Description**: *Step definition is incomplete in XML. Assuming this step writes the `thisRun` property value to a persistent storage (e.g., Data Store, external system) to record the last successful run timestamp.*

4.  **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends after writing the last run timestamp.

#### 9. Log Process

**Process Name:** Log

**Process Overview:** This is a utility process for logging information. It can send logs via email and/or SFTP based on configuration properties.

**Process Flow:**

```
Start --> Content Modifier --> Mail --> SFTP --> Send Mail --> End
                                |         |
                                +-------->+--> End
                                |         |
                                +-------->+--> Send SFTP --> End
```

**Step-by-step Description:**

1.  **Start**:
    *   **Type**: StartEvent
    *   **Description**: The process starts upon receiving a message containing log information from other processes like "Invalid Log" or "Extract Error Process".

2.  **Content Modifier**:
    *   **Type**: ContentModifier
    *   **Description**: This step extracts properties from the message payload, assuming the payload is in a format like `<mailOutput>`. It extracts `mail_from`, `mail_title`, `mail_to`, and `mail_body` using XPath expressions.
    *   **Properties Table:**

        | Action | Type  | Name      | Value                      | Datatype        | Default |
        |--------|-------|-----------|----------------------------|-----------------|---------|
        | Create | xpath | mail_from | `/mailOutput/from/text()`   | `java.lang.String` |         |
        | Create | xpath | mail_title| `/mailOutput/title/text()`  | `java.lang.String` |         |
        | Create | xpath | mail_to   | `/mailOutput/recipients/text()` | `java.lang.String` |         |
        | Create | xpath | mail_body | `/mailOutput/body/text()`   | `java.lang.String` |         |

3.  **Mail**:
    *   **Type**: router
    *   **Description**: This step checks if email notifications are enabled.
    *   **Conditions**:
        *   **Yes**: Condition `${property.EmailNotificationsEnabled} = 'true'`. If email notifications are enabled, the flow proceeds to "Send Mail".
        *   **No**: Default condition. If email notifications are not enabled, the flow proceeds directly to "SFTP".

4.  **SFTP**:
    *   **Type**: router
    *   **Description**: This step checks if SFTP logging is enabled.
    *   **Conditions**:
        *   **Yes**: Condition `${property.SFTPLoggingEnabled} = 'true'`. If SFTP logging is enabled, the flow proceeds to "Send SFTP".
        *   **No**: Default condition. If SFTP logging is not enabled, the flow proceeds to "End".

5.  **Send Mail**:
    *   **Type**: API_CALL
    *   **Description**: This step sends an email with the log information. It uses the adapter `Mail-Exceptions` and receiver `Receiver-Email`. This is a **3rd party system call** to an email service.
    *   **Adapter**: `Mail-Exceptions`
    *   **Receiver**: `Receiver-Email`

6.  **Send SFTP**:
    *   **Type**: API_CALL
    *   **Description**: This step sends the log information to an SFTP server. It uses the adapter `SFTP-Error` and receiver `SFTP-Log`. This is a **3rd party system call** to an SFTP server.
    *   **Adapter**: `SFTP-Error`
    *   **Receiver**: `SFTP-Log`

7.  **End**:
    *   **Type**: endEvent
    *   **Description**: The process ends here after attempting to log via email and/or SFTP based on the configuration.
```