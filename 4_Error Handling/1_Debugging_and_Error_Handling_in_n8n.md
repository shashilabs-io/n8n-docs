# Chapter 1: Debugging and Error Handling in n8n
## Master AI Automation Workflows with Execution Logs

## Learning objectives

By the end of this chapter, you should be able to:

- Distinguish manual executions from production executions.
- Find and interpret workflow execution logs.
- Inspect node input, output, errors, and execution duration.
- Identify common reasons workflows fail.
- Decide when retries are useful and when they are not.
- Configure node error behavior to stop, continue, or route failed items to an error output.

---

## 1. What is a workflow execution?

A **workflow execution** is one run of a workflow, from its trigger or selected starting point through the processing steps that run and any resulting actions.

Two execution types are important:

### Manual execution

A manual execution happens when you run a workflow yourself—for example, by clicking **Execute workflow** while testing it.

### Production execution

A production execution happens when an activated workflow runs from an automatic trigger, such as a schedule or an event trigger.

A workflow generally needs an appropriate trigger before it can be activated for automatic execution. A manual test run and an activated workflow run are not the same thing, so check which type appears in the execution history.

---

## 2. What are execution logs?

The **execution log** records workflow runs. Depending on workflow settings and the available n8n plan, the execution list can show information such as:

- Workflow name
- Execution status, such as success or error
- Start time
- Duration
- Execution ID
- Whether the run was a test/manual run or a production run

Opening an execution lets you inspect the run in more detail. When execution data has been saved, you can often inspect node input and output, identify the node that failed, and read the associated error details.

Execution logs are valuable because they let you examine what happened in a particular run instead of trying to recreate the same data and conditions from scratch.

### Filtering execution history

The execution view may offer filters for workflow, status, and date/time range. Available filters can depend on the n8n version, hosting setup, and plan.

### Workflow execution-saving settings

Workflow settings can control whether failed production executions, successful production executions, and manual executions are saved. These settings affect which runs and data are available later for debugging.

**Practical advice:** Decide what execution data you need for troubleshooting and operational review. Be mindful that saved inputs and outputs may contain personal, confidential, or otherwise sensitive information.

---

## 3. Read an execution from start to finish

When investigating a run, use a repeatable process:

1. Open the **Executions** view.
2. Find the relevant workflow and run.
3. Check whether the run was manual or production-triggered.
4. Review its status, start time, and duration.
5. Open the execution.
6. Locate the failed node, if any.
7. Inspect that node's input, output, and error message.
8. Check the upstream data and configuration that led to the error.
9. Make a targeted correction and run a controlled test.

### What the log can reveal

- **Trigger or starting point:** Was the workflow triggered automatically, or did someone run a node manually?
- **Node input:** What data did the node receive?
- **Node output:** What did the node return?
- **Error details:** What did the node or external service report?
- **Runtime:** How long did the workflow take?

A partial/manual node run may not include input and output for nodes that were not executed. Do not assume every node's data will be available for every execution.

---

## 4. Diagnose errors by examining the actual data

### Example A: Invalid email address

Imagine a workflow receives form submissions and sends each customer a thank-you email. Some records contain a malformed email address or an empty email field.

The email service may reject those records. The execution log can reveal the failing email node and the service's error message. Inspecting the input can then show whether the email address was malformed or missing.

**Possible fixes:**

- Validate email input before sending.
- Require the field when the form is submitted.
- Route invalid records to a review path.
- Keep a record of failed deliveries so they can be corrected and retried later.

### Example B: Wrong file type

Imagine a downstream node expects an image URL, but the workflow passes a PDF instead. The request may fail because the input does not match what the node expects.

**Possible fixes:**

- Check the incoming file type or MIME type.
- Route files by type before sending them to a specialized node.
- Reject or quarantine unsupported files with a useful error message.

### The key debugging principle

Do not stop at “the node failed.” Find out **which node failed, what input it received, and what the error says**. Then determine whether the cause is data, configuration, a remote service, authentication, or workflow logic.

---

## 5. Common causes of failed executions

| Cause | What it can look like | What to investigate |
|---|---|---|
| Misconfigured node | A required parameter is absent or incorrect | Node parameters, credentials, field mappings |
| External service unavailable or rate-limited | Temporary service errors or too many requests | Service status, rate limits, retry policy |
| Missing or malformed data | Empty email, invalid URL, unexpected value | Input validation and upstream data |
| Authentication/authorization problem | Expired credential, rejected API key, insufficient permission | Credential status, access scope, subscription or permissions |
| Incorrect conditional logic | Boundary value takes the wrong branch | Comparisons, empty values, negative numbers, equality cases |
| Unexpected file or data type | A node receives a PDF when it expects an image | MIME type, schema, data transformations |

Errors can occur at any stage of a workflow. A reliable workflow anticipates expected failure cases instead of assuming all input data and external services will always be perfect.

---

## 6. Configure retry behavior

The **Retry On Fail** setting can help when a failure may be temporary—for example, a transient service problem or rate limit. Depending on the node and version, you can configure a maximum number of retries and a wait between attempts.

### When retries help

- A service is briefly unavailable.
- A transient network issue occurs.
- A rate-limited request may succeed after a delay.

### When retries do not solve the underlying problem

Retries will not correct deterministic errors such as:

- An invalid email address
- A required field that is empty
- A wrong binary property name
- A file type that the target node does not accept
- A permanently invalid credential or parameter

Repeatedly sending the same invalid input usually causes the same failure again. Fix the input or configuration instead.

**Tip:** Keep retry counts and delays appropriate for the service. Excessive retries can increase execution time and may worsen rate-limit problems.

---

## 7. Choose what happens when a node errors

A node's **On Error** setting controls how the workflow behaves when processing fails. The available options and labels may vary by n8n version.

### Option A: Stop the workflow

The workflow stops when the node encounters an error. This is appropriate when continuing could produce an incorrect or unsafe result, or when the later steps depend on the failed operation.

### Option B: Continue using the regular output

The workflow continues after an error using the regular output path, where supported by the selected option and node behavior.

This can allow later items to be processed when one item fails. However, if failures are merely ignored, the overall execution may appear successful even though some records were not processed.

Use this approach only when you have a deliberate way to account for failed items. Otherwise, errors can become silent data loss.

### Option C: Continue using the error output

Where supported, this option routes failed items to a separate **error output**, while successful items continue through the normal output.

This is useful when you want two paths:

```text
                 Node processes items
                         |
                 +-------+-------+
                 |               |
              Success          Error
                 |               |
                 v               v
          Continue normal   Log details, alert,
          workflow steps    or send for review
```

The success path can continue with successfully processed records. The error path can log the item and error message, notify a team, or invoke a separate error-handling workflow.

### A practical pattern

For a batch of customer records:

1. Validate each record before sending an email.
2. Send valid records through the email node.
3. Use an error output where suitable for failures from the email operation.
4. Record failed records and error messages.
5. Notify a person or send failures to a review workflow.
6. Reprocess corrected records when appropriate.

---

## 8. Why “continue” needs careful monitoring

Continuing past an error can improve throughput when a batch contains both valid and invalid records. But it can also hide failures if nobody checks the error data.

For example, a batch might contain five records, of which two emails are sent successfully and three fail. If the workflow continues without recording or surfacing the three failures, the workflow may complete even though not every customer received an email.

A robust workflow should distinguish between:

- **Workflow completion:** The execution reached its end.
- **Business success:** The intended work was actually completed for the required records.

These are not always the same thing.

---

## 9. A repeatable debugging checklist

When a workflow fails:

- [ ] Open the relevant execution log.
- [ ] Confirm whether it was a manual or production execution.
- [ ] Locate the failed node.
- [ ] Read the full error message.
- [ ] Inspect the node's input data.
- [ ] Check required fields, formats, and data types.
- [ ] Verify credentials and permissions if an external service is involved.
- [ ] Check whether the issue is temporary or deterministic.
- [ ] Use retries only when they fit the failure type.
- [ ] Choose an appropriate On Error behavior.
- [ ] Route, log, or alert on failed items instead of silently discarding them.
- [ ] Test the fix with both valid and invalid sample inputs.
- [ ] Review the new execution to confirm the expected result.

---

## 10. Practice questions

1. What is the difference between a manual execution and a production execution?
2. Name four details that execution logs may help you inspect.
3. Why is node input important when debugging?
4. Give two examples of errors that retries may help with.
5. Give two examples of errors that retries are unlikely to fix.
6. What is the risk of continuing past an error without monitoring the failed items?
7. How can a separate error output improve a workflow?
8. A node expects an image but receives a PDF. What should you inspect?
9. Why might a workflow reach its end without achieving business success?
10. What sensitive-data concern should you consider when saving execution data?

### Answer key

1. Manual execution is initiated by a person during testing; production execution is triggered by an activated workflow's automatic trigger.
2. Examples include status, start time, duration, execution ID, node input/output, and error details, depending on saved data and available features.
3. Input shows the exact values the node received and can reveal missing fields, invalid formats, or unexpected types.
4. A temporary service interruption and a transient network issue.
5. A malformed email address and a PDF passed to a node that expects an image.
6. Some items can fail while the overall run appears successful, leading to silent data loss.
7. It lets successful items continue normally while failed items are logged, reviewed, or sent to a dedicated error-handling path.
8. Inspect the file type/MIME type, incoming data, and the node's expected input.
9. The workflow may continue after individual items fail, while the errors are not surfaced or resolved.
10. Execution inputs and outputs may contain personal or confidential information, so saving and access should be considered carefully.

---

## Chapter summary

- A workflow execution is one run; manual and production executions have different triggers.
- Execution logs help identify when a run occurred, which nodes ran, what data they processed, and where errors appeared—provided the relevant data was saved.
- Inspect node input and error details before deciding how to fix a failure.
- Common causes include misconfiguration, unavailable services, invalid or missing data, authentication problems, and incorrect logic.
- Retries are useful for some temporary failures but do not fix bad input or deterministic configuration errors.
- Choose between stopping, continuing, or routing errors to a separate output based on the workflow's needs.
- Never let “continue” silently hide failed records; log, route, or alert on errors so business outcomes remain visible.
