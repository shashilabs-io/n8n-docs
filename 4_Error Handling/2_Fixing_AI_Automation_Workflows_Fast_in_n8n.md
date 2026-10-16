# Chapter 2: Fixing AI Automation Workflows Fast in n8n

## Learning Objectives

By the end of this chapter, you should be able to:

- Inspect failed production executions and locate the node that caused an error.
- Use **Debug in editor** to load the failed execution's data into an editable workflow.
- Retry a failed execution using either the currently saved workflow or the original workflow.
- Restore an earlier workflow version when recent changes cause problems.
- Validate incoming data before passing it to sensitive nodes.
- Route invalid data to a deliberate error path instead of allowing it to disappear silently.

---

## 1. Why Workflows Fail

An automation can fail even when its overall design is correct. Common situations include:

- **Incorrect field references:** a node expects `userEmail`, but the incoming item contains `email`.
- **Invalid or incomplete input:** a form submission has an empty or malformed email address.
- **External service problems:** an API or email service is temporarily unavailable.
- **Configuration problems:** a sender address is not verified, or a node has incorrect parameters.
- **Changes that introduce regressions:** a workflow that worked earlier starts producing errors after edits.

The goal is not only to make the red error indicator disappear. You should identify the cause, repair it, and decide how to handle similar failures in future executions.

## 2. Find the Failed Execution

When a production workflow fails, begin with its execution history.

1. Open the n8n **Overview** or the relevant workflow.
2. Open **Executions** (the execution log).
3. Find the run marked **Error**.
4. Open the execution and inspect its status, start time, duration, execution ID, node inputs and outputs, and error message.
5. Identify the node where execution stopped.

For example, an email node might report a bad request. Inspect the error and the data reaching that node before changing settings at random.

> **Tip:** The execution view is useful for investigation, but the historical execution itself may be read-only. Use the editor's debugging feature when you need to test a fix against the data from the failed run.

## 3. Debug in Editor

**Debug in editor** loads a failed execution's data into the workflow editor so that you can investigate and test changes using the same input that caused the failure. This avoids having to submit the original form again or recreate the data manually.

### Workflow

1. Open the failed execution from the execution log.
2. Review the node error and the data available around it.
3. Choose **Debug in editor**.
4. Inspect the loaded workflow and its node data.
5. Edit the node or expression responsible for the error.
6. Execute the relevant step to test the correction.
7. Save the corrected workflow when you are satisfied.

### Example: Wrong field name

Suppose the incoming item is:

```json
{
  "firstName": "Riya",
  "lastName": "Sharma",
  "email": "riya@example.com",
  "company": "Example Co",
  "role": "Developer"
}
```

A downstream expression refers to a field named `userEmail`, but the actual field is `email`. The expression is therefore reading a property that does not exist.

Incorrect reference:

```text
{{$json.userEmail}}
```

Correct reference for this input:

```text
{{$json.email}}
```

The key lesson is to compare the expression with the **actual incoming JSON**, not with what you assume the field should be called.

### A second error may appear

Fixing one problem can reveal another. For example, after correcting the field name, an email service might reject the sender address because it has not been verified. That is a separate configuration issue: use an approved, verified sender address and test again.

**Debugging habit:** fix one cause at a time, execute the affected step, and read the next error carefully.

## 4. Retry a Failed Execution

A failure caused by a temporary external service problem may not require a change to the workflow. If the original run contained important data that was never processed, you may want to retry that execution after the service is available again.

In the failed execution view, use **Retry execution**. The interface described here offers two choices:

| Retry option | What it does | When it is useful |
|---|---|---|
| **Retry with currently saved workflow** | Runs the failed execution using the workflow version currently saved, including fixes made since the failure. | Use this when you have corrected the workflow and want to try the failed data again. |
| **Retry with original workflow** | Runs the failed execution using the workflow as it was at the time of the failure. | Use this when you want to reproduce the original behavior or confirm the original failure. |

### Example

1. A workflow fails because an expression references the wrong field.
2. You use **Debug in editor** to correct the expression.
3. You save the corrected workflow.
4. Open the earlier failed execution and choose **Retry execution**.
5. Select **Retry with currently saved workflow** to test the failed data against the fix.

If you retry with the original workflow, the same error may occur again because the original configuration is unchanged.

### Before retrying

- Confirm that the external service is available.
- Check whether the workflow has side effects, such as sending emails, creating records, or charging payments.
- Consider whether retrying could duplicate an action that partially succeeded before the failure.
- Review the execution result after the retry.

Retry behavior and available choices can depend on the n8n version and deployment. Check your instance's interface if the labels differ.

## 5. Use Workflow Version History

Version history helps recover from changes that make a previously working workflow behave incorrectly.

### Typical steps

1. Open the workflow.
2. Find **Workflow history** near the workflow's save/status controls.
3. Review the available saved versions.
4. Open or inspect a version to identify the configuration you want.
5. If appropriate, choose **Restore** and confirm the replacement.

The version-history interface may also offer actions such as opening a version in a new tab, cloning it into a separate workflow, or downloading and importing it. Availability can vary by n8n version and plan.

### Important caution

Restoring a version replaces the current workflow configuration with the selected earlier version. Any fixes made after that version may be lost.

A safer approach when you are unsure is to inspect or clone the earlier version first, compare it with the current workflow, and restore only when you have confirmed which version you need.

### Version history and plan limits

The retention period for workflow history can depend on your n8n plan or deployment. Do not assume that every old version will remain available indefinitely. For important workflows, maintain an appropriate backup or export process.

## 6. Prevent Invalid Data from Reaching Sensitive Nodes

A workflow should validate data before passing it to nodes that require specific fields. For example, an email-sending node should not receive an empty recipient address.

### Example workflow

```text
Form/Webhook
     |
     v
 Validate email (IF)
     |
     +---- True: continue to the email/action nodes
     |
     +---- False: record or raise an error
```

Use an **IF** node to check whether the email field is present and non-empty before continuing. You can add additional checks appropriate to the application, such as checking that the value resembles an email address.

### What validation can prevent

- Empty required fields reaching a downstream node.
- Expressions reading properties that are missing from an input item.
- Avoidable failures caused by clearly malformed data.
- Unclear behavior when form submissions are incomplete.

Validation reduces predictable errors, but it does not guarantee that a downstream service will always succeed. External services can still be unavailable, credentials can expire, and APIs can reject otherwise valid requests.

## 7. Do Not Let Invalid Data Disappear Silently

A common design mistake is to route invalid data away from the main path and do nothing with it. The workflow may appear successful even though a submitted record was never processed.

There are two practical approaches.

### Option A: Log the invalid item

Connect the **False** branch of the IF node to a logging destination, such as a spreadsheet or database. Record enough information to investigate the issue later, for example:

- Timestamp
- Workflow or source form
- Validation failure reason
- Relevant submitted fields, subject to privacy requirements
- Whether the item was corrected or reprocessed

Avoid storing secrets or unnecessary personal information in logs.

### Option B: Raise an explicit error

Connect the invalid-data branch to a **Stop and Error** node. This deliberately stops that path and raises an error, making the failure visible to the workflow's monitoring and error-handling setup.

Use a clear error message, such as:

```text
Registration rejected: the email field is missing or empty.
```

Choose the behavior that matches the business requirement. Some workflows should log and continue processing other items; others should stop and raise an error when required data is missing.

> A validation branch is not complete until you decide what should happen to invalid items.

## 8. A Practical Troubleshooting Routine

Use this checklist whenever an automation fails:

1. **Locate:** Open the failed execution in the execution log.
2. **Read:** Check the error message and identify the failing node.
3. **Inspect:** Compare the node's expected inputs with the actual JSON data.
4. **Classify:** Decide whether the problem is data-related, configuration-related, or temporary.
5. **Reproduce:** Use **Debug in editor** to load the failed data.
6. **Fix:** Correct the field reference, input validation, credentials, or node parameters.
7. **Test:** Execute the relevant node or path.
8. **Save:** Save the corrected workflow.
9. **Retry carefully:** Choose the appropriate retry option if the failed data must be processed again.
10. **Prevent recurrence:** Add validation, logging, an explicit error path, or suitable retry settings.
11. **Recover if needed:** Inspect workflow history and restore a known-good version only after checking what will be replaced.

## 9. Quick Reference

| Problem | First action |
|---|---|
| A node reports a missing property | Inspect its input JSON and correct the field reference. |
| A service fails intermittently | Check service availability; consider retrying the failed execution. |
| A fix was made after the failure | Retry using the currently saved workflow when appropriate. |
| You need to reproduce the old failure | Retry with the original workflow, if available and safe. |
| Recent edits broke a working workflow | Inspect workflow history and compare versions. |
| A form field may be empty | Validate it before downstream nodes. |
| Invalid data is being ignored | Log it or route it to a Stop and Error node. |

## 10. Practice Questions

Try answering these without looking back at the chapter.

1. What is the purpose of **Debug in editor**?
2. An input item contains `email`, but an expression reads `userEmail`. What should you inspect and change?
3. What is the difference between retrying with the currently saved workflow and retrying with the original workflow?
4. Why might retrying an execution be appropriate after a temporary API outage?
5. What risk should you consider before retrying a workflow that sends emails or creates records?
6. How can workflow version history help after a problematic edit?
7. Why is it not enough to route invalid data to a branch that has no further nodes?
8. Give one example of when a **Stop and Error** node is appropriate.
9. Name three useful details to record when logging invalid input.
10. Does validating an email address guarantee that the email service will succeed? Why or why not?

### Answer Key

1. It loads the failed execution's data into an editable workflow so you can troubleshoot with the same data.
2. Inspect the actual input JSON and change the expression to reference the field that exists, such as `{{$json.email}}`.
3. The current-workflow option uses the saved configuration; the original-workflow option uses the configuration from the failed run.
4. The service may be available again, allowing the failed data to be processed.
5. Duplicate side effects, such as sending the same email or creating the same record twice.
6. It lets you inspect and potentially restore an earlier workflow configuration.
7. The workflow may appear successful while the invalid item is silently discarded.
8. When required data is missing and processing must stop with a visible error.
9. Examples include timestamp, failure reason, source, and relevant submitted fields.
10. No. Validation checks the input, but service availability, credentials, and other configuration can still cause failure.

## Key Takeaways

- Start with the execution log and the exact node error.
- Debug using the data from the failed execution rather than guessing.
- Choose retry behavior intentionally; the original and current workflow versions may behave differently.
- Use version history carefully because restoring replaces the current configuration.
- Validate important fields before they reach downstream nodes.
- Always define what happens to invalid data: log it, raise an error, or handle it through another explicit path.
