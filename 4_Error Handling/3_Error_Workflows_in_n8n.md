# Chapter 3: Error Workflows in n8n

## Learning Objectives

By the end of this chapter, you should be able to:

- Explain what an error workflow is and why it is useful.
- Describe the roles of the **Error Trigger** and **Stop and Error** nodes.
- Create a basic error-notification workflow.
- Attach an error workflow to another workflow.
- Notify workflow owners and teams about failures.
- Classify errors by priority and log them for later investigation.

---

## 1. What Is an Error Workflow?

An **error workflow** is a dedicated workflow that runs when another configured workflow encounters an error during a production execution. It can collect details about the failure, write them to a log, and notify the people responsible for fixing the issue.

Without deliberate error handling, a workflow can sometimes appear successful even though important data was ignored or did not reach its destination. An error workflow helps make failures visible and actionable.

Typical purposes include:

1. **Capture error details** — record the workflow name, execution ID, execution URL, error message, and the node where the failure occurred.
2. **Notify responsible people** — send an email, Slack message, or another alert.
3. **Create an error log** — maintain a record of failures for investigation and reporting.
4. **Prioritize incidents** — route low- and high-priority errors differently.

## 2. The Two Key Nodes

### 2.1 Error Trigger

The **Error Trigger** node starts an error workflow when a configured workflow fails. It provides error-related information that can be used in later nodes, such as:

- Workflow name and ID
- Execution ID and execution URL
- Error message and other error details
- The last executed node or node associated with the error
- Additional execution information, which may include an HTTP status code or stack trace

The exact properties available depend on the error event. Inspect the actual Error Trigger output before writing expressions that reference its fields.

The Error Trigger is designed for handling errors from other workflows. Its output becomes the input to your notification, logging, and routing steps.

### 2.2 Stop and Error

The **Stop and Error** node deliberately stops a workflow and raises an error. It is useful when a condition that must be satisfied is not met.

For example, if a form submission must contain an email address, you can validate the field first. If it is missing, route the item to Stop and Error instead of silently ignoring it.

The node can raise an error using a custom message or, where supported by the node configuration, a structured error object. A custom message might be:

```text
Registration rejected: no email address was provided.
```

A structured error object can carry additional context into the error workflow, making it easier to investigate the failure.

### How the two nodes work together

```text
Main workflow
     |
     v
Validate required data
     |
     +---- Valid ----> Continue normal processing
     |
     +---- Invalid --> Stop and Error
                           |
                           v
                 Configured error workflow
                           |
                 +---------+---------+
                 v                   v
             Log error          Notify owner/team
```

**Important distinction:** Stop and Error raises a deliberate failure in the main workflow. Error Trigger receives the failure event in the separately configured error workflow.

## 3. Build a Basic Error Workflow

A simple error workflow can send an email whenever another configured workflow fails.

### Step 1: Create a new workflow

Create a new workflow and give it a clear name, such as `Generic Error Handler`.

### Step 2: Add an Error Trigger node

1. Add the **Error Trigger** node as the first node.
2. Use its test-event or mock-data option, if available, to inspect an example event.
3. Review the available workflow, execution, and error fields.

The trigger generally does not require the same kind of input configuration as a normal application trigger; it receives details when a configured workflow reports an error.

### Step 3: Add a notification node

Connect a notification node, such as Gmail, after the Error Trigger.

A useful alert should contain:

- The workflow that failed
- The error message
- The node associated with the error, if available
- A direct link to the failed execution
- A timestamp or execution identifier, where available

A sample email layout:

```text
Subject: Workflow failure — {{ workflow name }}

A workflow execution has failed.

Workflow: {{ workflow name }}
Failed node: {{ node name }}
Error: {{ error message }}

Open the execution details:
{{ execution URL }}
```

The placeholders above describe the information to include. Use expressions that match the actual Error Trigger output in your n8n instance.

### Step 4: Test the notification

Use test data to check the notification node and confirm that the message is formatted as expected. A mock event can help with building the workflow, but it does not prove that the complete production error flow is configured correctly.

---

## 4. Attach the Error Workflow to a Main Workflow

Creating an error workflow is not enough on its own. Each main workflow that should use it must be configured to reference it.

Typical setup:

1. Open the main workflow.
2. Open its workflow settings.
3. Find the **Error workflow** setting.
4. Select the error workflow you created.
5. Save the main workflow.
6. Activate the main workflow when it is intended to run in production.

In the setup described here, error workflows are triggered by production workflow failures. Manual test runs may not behave the same way as active production executions, so verify the behavior supported by your n8n version.

Repeat the configuration for each workflow that should send errors to the shared handler.

## 5. Make Alerts More Useful

A generic alert is a good start, but larger projects benefit from routing each incident to the right owner.

### Maintain a workflow-owner mapping

Keep a small table in an accessible source, such as a Google Sheet or database, with fields like:

| Workflow name or ID | Owner name | Owner email | Slack user ID |
|---|---|---|---|
| Customer registration | Riya | riya@example.com | `U123EXAMPLE` |
| Attachment processing | Dev team | dev-team@example.com | `U456EXAMPLE` |

These are illustrative records. Use your own real workflow identifiers and owner details.

When an error arrives, use the workflow name or ID from the Error Trigger to look up the corresponding owner. A stable workflow ID can be preferable if names may change.

### Notify the owner and the team

A useful notification strategy can include:

- A direct message to the workflow owner
- An email to a monitored team inbox
- A message in a shared incident or operations channel
- A link to the failed execution so the recipient can investigate quickly

Sending both an individual alert and a team-level notification can reduce the risk that an incident goes unnoticed when the owner is unavailable.

## 6. Classify Errors by Priority

Not all errors need the same response. A Switch node or other conditional logic can route errors based on fields in the error event.

An illustrative classification:

| Priority | Example situations | Possible response |
|---|---|---|
| **Low** | A known validation failure, such as a missing email field | Log the event and send a routine notification |
| **Medium** | A bad request caused by invalid parameters or data | Notify the owner and investigate the input/configuration |
| **High** | Authentication or authorization failures, an expired subscription, or a business-critical failure | Send an immediate alert to the owner and the responsible team |

This is an example policy, not a universal mapping. An HTTP status code alone does not always tell you the true business impact. Classify errors using the error details and the importance of the affected workflow.

### Example routing logic

```text
Error Trigger
     |
     v
Look up workflow owner
     |
     v
Classify error
     |
     +---- Low ----> Log + routine email
     |
     +---- Medium -> Log + owner notification
     |
     +---- High ---> Log + owner + team alert
```

Make sure the routing logic includes a fallback for unclassified errors. Otherwise, a new error type may not reach any notification branch.

## 7. Maintain an Error Log

Notifications help people respond, while a persistent log helps teams understand recurring failures.

An error log can include columns such as:

- Timestamp
- Workflow name or ID
- Execution ID
- Error message
- Failed node
- Execution URL
- Priority
- Assigned owner
- Status or resolution notes

An append-row operation in a spreadsheet or database can record each event. Make sure the expressions map to the actual Error Trigger fields and the columns in your log.

### Logging considerations

- Store only the data needed for diagnosis.
- Avoid putting passwords, access tokens, or other secrets into logs.
- Limit access to logs that contain personal or business-sensitive information.
- Decide how long logs should be retained.
- Record whether an incident has been resolved, if you need to track follow-up work.

## 8. Example: Handle a Missing Email in a Form

Suppose a registration workflow receives form data through a webhook. The email field is required, but a submission arrives without it.

### Main workflow

```text
Webhook
   |
   v
IF: email exists and is acceptable
   |
   +---- True ----> Continue registration workflow
   |
   +---- False ---> Stop and Error
```

Set the condition to check the field that actually exists in the incoming data. Depending on your form and requirements, you may need to check both that the value is non-empty and that it has an acceptable format.

Configure Stop and Error with a clear message, for example:

```text
Registration rejected: no email found.
```

### Error workflow

The configured error workflow receives the failure and can:

1. Look up the workflow owner.
2. Classify the error as a validation issue.
3. Append the event to an error log.
4. Send a routine email notification.
5. Send an urgent team alert only if the priority rules require it.

This approach makes invalid submissions visible instead of quietly discarding them.

## 9. Troubleshooting Checklist

If an error workflow does not behave as expected, check the following:

1. **Is the Error Trigger the first node?** Confirm that the workflow begins with the appropriate error trigger.
2. **Is the main workflow configured to use it?** Check the main workflow's Error workflow setting.
3. **Is the main workflow active?** Confirm that it is running in the intended production mode.
4. **Does the error workflow execute successfully?** Inspect its own execution log; the handler can fail too.
5. **Are the expressions correct?** Inspect the actual Error Trigger output and verify every referenced field.
6. **Are credentials valid?** Test the Gmail, Slack, Sheets, or other notification/logging connections.
7. **Does every priority branch have a destination?** Add a fallback path for unexpected error types.
8. **Is the original error intentional?** Use Stop and Error only when the workflow should genuinely be marked as failed.
9. **Are alerts actionable?** Include a clear message and a link to the relevant execution.
10. **Are logs safe?** Avoid storing secrets or unnecessary personal information.

## 10. Practice Questions

1. What is an error workflow?
2. What is the role of the Error Trigger node?
3. How is Stop and Error different from Error Trigger?
4. Why should you inspect actual Error Trigger output before writing expressions?
5. What configuration connects a main workflow to an error workflow?
6. Why might an error workflow need both an owner lookup and a shared team alert?
7. Name four useful fields for a persistent error log.
8. Why should error-priority rules include a fallback?
9. Give an example of a validation failure that should raise an error.
10. Why should logs avoid storing secrets?

### Answer Key

1. A dedicated workflow that handles failures reported by configured workflows.
2. It receives the error event and exposes information about the failed workflow execution.
3. Stop and Error deliberately raises a failure in the main workflow; Error Trigger starts the separate handler when a configured workflow fails.
4. Available fields can vary by error event, so expressions must match the real output.
5. Select the error workflow in the main workflow's settings and save the configuration.
6. The owner can investigate directly, while the team alert provides shared visibility if the owner is unavailable.
7. Examples: workflow name, execution ID, error message, failed node, execution URL, priority, timestamp.
8. An unexpected error might otherwise be routed nowhere and remain unnoticed.
9. A required email field is missing from a registration form.
10. Logs may be broadly accessible or retained for a long time, so secrets in them could be exposed.

## Key Takeaways

- Use a dedicated error workflow to make failures visible and actionable.
- Error Trigger receives failure information; Stop and Error deliberately raises a failure when a required condition is not met.
- Configure the main workflow to use the appropriate error workflow.
- Build notifications with the workflow name, error details, and a direct execution link.
- Use owner mapping, priority routing, and persistent logs as your automation grows.
- Add a fallback path and verify the error handler itself works.
