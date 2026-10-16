# Master Error Handling in n8n
## Build Reliable Workflows That Recover from Errors

### Learning objectives

By the end of this chapter, you should be able to:

- Explain why error handling matters in automation.
- Distinguish retrying an operation from continuing after a failure.
- Configure a node to stop, continue with an error item, or use a separate error output.
- Route failed items to logging or notification steps.
- Preserve useful context so a developer can understand what failed.
- Build a loop that continues processing later items after an individual item fails.

---

## 1. What is error handling?

**Error handling** is the process of deciding what a workflow should do when an operation fails. Without an appropriate strategy, a single failure can stop the remaining execution. With error handling, a workflow can retry temporary failures, record permanent failures, notify someone, or continue processing other records.

External services and APIs can become temporarily unavailable. File uploads can fail. A destination may reject a file type or malformed request. A robust workflow plans for these cases rather than assuming every operation will succeed.

### Common examples

- Retry an API request that may have failed temporarily.
- Record details when a file upload fails.
- Notify a developer when an email or integration step fails.
- Continue processing the remaining records when one record cannot be handled.
- Queue a failed message for later processing, where the system supports it.

**Important:** Error handling does not mean hiding every error. It means choosing a suitable response and making failures visible enough to investigate.

## 2. Example: one invalid attachment stops a workflow

Imagine an email contains nine attachments. The workflow uploads image attachments to a task board, creating a page for each image. The task board accepts images, but one attachment is a PDF.

When the workflow reaches the PDF, the destination rejects it because the selected block expects an image. If the node is configured to stop the workflow on error, processing halts at that attachment and later items may not be processed.

The desired outcome is different:

- Upload the supported images.
- Record or notify someone about the unsupported PDF.
- Continue processing the remaining attachments.

This example illustrates why a workflow needs both an error policy and a clear path for failed items.

## 3. Three different strategies

### Strategy A: Retry on Fail

A retry is useful when an error may be temporary—for example, a brief network problem or a temporarily unavailable service.

In the node's **Settings**, enable **Retry On Fail** if it is available. Configure the retry count and wait interval according to the options shown in your n8n version.

Conceptually:

```text
Attempt operation
      |
      v
   Failed?
   /    \
 No      Yes
 |        |
Continue  Retry if attempts remain
          |
          v
     Stop/report if retries fail
```

A retry is not a fix for a permanent problem. If a PDF is sent to an operation that accepts only images, retrying the same request with the same input will generally fail again. Validate the input or route it to a suitable handler instead.

### Strategy B: Continue with an error item in the regular output

The node's **On Error** setting may offer an option named **Continue (using regular output)** or similar wording, depending on the n8n version.

With this approach, the workflow can continue after a node failure, and error information may appear in the regular output as an item. A later IF node can check whether an error field exists and route the item to a logging or notification path.

Conceptually:

```text
Operation
   |
   v
Regular output
   |
   v
IF: does an error field exist?
   |                  |
  Yes                 No
   |                  |
Log / notify       Continue normal work
```

Inspect the actual output of the failed node before choosing the condition. Do not assume every node or version uses precisely the same error structure.

### Strategy C: Continue using a separate error output

Where supported, choose **Continue (using error output)** or the equivalent option. The node then provides a normal success output and a separate error output.

- Connect the success output to the next normal step.
- Connect the error output to a logger, database, or notification node.
- Include useful information from the failed item in the error report.
- If the workflow is inside a loop, ensure the intended paths reconnect correctly so later items can be processed.

Conceptually:

```text
                 +--> Success output --> Wait / next loop step
Input --> Action |
                 +--> Error output ----> Log details / notify team
```

This approach makes success and failure paths explicit and can avoid needing a separate IF node merely to detect an error item.

**Version note:** Available setting labels and output ports can vary. Use the options visible in your installed n8n node and verify them with a test execution.

---

## 4. Build a practical error-handling workflow

The following pattern uses the attachment-processing example.

```text
Gmail Trigger
     |
     v
Filter attachments
     |
     v
Split attachments into items
     |
     v
Loop Over Items
     |
   [Loop]
     |
     v
Upload file
     |
     v
Create task-board page
     | success                 | error
     v                         v
Wait / continue loop       Send error notification
     |                         |
     +---- back to loop        +---- log failed item

   [Done]
     |
     v
After-loop action
```

### Step 1: Decide which errors are retryable

Before configuring the node, decide whether the failure is likely temporary or caused by invalid input.

| Failure example | Possible response |
|---|---|
| Temporary network or service outage | Retry a limited number of times |
| Unsupported file type | Filter it out or route it to an alternate handler |
| Missing required field | Validate the input and log the invalid record |
| Invalid credentials | Notify the owner and fix the credential configuration |
| API rate limit | Respect the service's policy and adjust request pacing |

These are starting points, not universal rules. The right response depends on the service and the consequences of repeating the operation.

### Step 2: Configure retry behavior when appropriate

Open the relevant node's **Settings** and inspect **Retry On Fail**. Enable it only when repeating the operation is reasonable. Choose a limited retry count and interval based on the service's behavior and the available configuration.

Avoid repeatedly retrying permanent errors such as unsupported formats or invalid input.

### Step 3: Choose the node's On Error behavior

Open the node's **Settings** and find **On Error**.

- **Stop Workflow:** the execution stops when this node fails.
- **Continue with regular output:** execution can continue with error information in the regular output.
- **Continue using error output:** failed items are sent through a separate error output, where supported.

The exact labels may vary across n8n versions. Test the selected behavior with a deliberately invalid input in a safe test workflow.

### Step 4: Send a useful error notification

Connect the error output to an email or another notification node. A message such as “An error occurred” is usually not enough to investigate.

Include available context such as:

- Workflow or operation name.
- The failed item identifier, filename, or URL.
- The error message returned by the node.
- The time of the failure, if available.
- The next action taken, such as “item logged; remaining items will continue.”

Use only fields actually present in the output. A notification should not assume a URL, filename, or error property exists for every failure.

### Step 5: Continue the loop correctly

If the failed item should not stop processing, connect the appropriate success and error paths so the loop can proceed as intended. In the example, the success path waits and reconnects to **Loop Over Items**. The error path logs or notifies and should also rejoin the intended loop path if the goal is to continue with the remaining items.

Test with a small collection that includes both a valid item and a known invalid item. Confirm that the valid items before and after the failure are processed and that the invalid item is logged.

---

## 5. Why context matters

An error message is most useful when it is connected to the item that caused it.

For example, compare these notifications:

**Not useful enough**

```text
Error occurred in your workflow.
```

**More useful**

```text
The task-board page could not be created.
File: invoice.pdf
Reason: The destination expects an image for this block.
Action: The failed item was logged for review.
```

The second message is illustrative. In a real workflow, populate the fields dynamically from data actually returned by the failed node. Do not hard-code a sample filename or error message as if it were the real failure.

When using a separate error output, inspect its data. It may contain both error details and context from the item that failed, allowing the notification to identify the specific URL, filename, record, or operation.

## 6. Common mistakes

| Mistake | Why it causes problems | Better approach |
|---|---|---|
| Retrying every error | Permanent input errors will usually fail repeatedly | Retry only errors that may be temporary |
| Continuing without logging | The workflow appears to succeed while failed items go unnoticed | Record the failure or alert someone |
| Sending a generic notification | The team cannot identify the failed item | Include available item and error details |
| Assuming a failed item was processed successfully | “Workflow completed” does not mean every operation succeeded | Inspect outputs and record failures separately |
| Forgetting to reconnect the loop path | The loop may stop after the current item or batch | Check the success and error connections and test several items |
| Assuming all error outputs have the same fields | Output structure can differ by node and error mode | Inspect a real test execution before mapping fields |
| Retrying unsupported files | Repeating the same invalid request does not make the file valid | Filter or route unsupported file types before the destination node |

## 7. Testing checklist

Before relying on a workflow in production, test the cases that matter:

- [ ] A valid item succeeds.
- [ ] A known invalid item reaches the intended error path.
- [ ] A retryable failure is retried only the configured number of times.
- [ ] A permanent failure is logged or reported.
- [ ] Later items continue when that is the intended behavior.
- [ ] Error notifications identify the failed item using real output fields.
- [ ] The workflow does not accidentally repeat one-time actions.
- [ ] Credentials, URLs, and sensitive file contents are not exposed in unnecessary notifications.

## 8. Practice exercises

### Exercise 1: Temporary API failure

An API request fails because the service is temporarily unavailable.

- Which strategy might help?
- Why should the number of retries be limited?

### Exercise 2: Unsupported file type

A task-board image block rejects a PDF attachment.

- Should you keep retrying the same request?
- What is a better response?

### Exercise 3: Continue after one failure

A workflow processes ten records. Record 4 fails, but records 5–10 should still be processed.

- Which On Error behavior could help?
- What should happen to the failed record?
- What loop connection should you verify?

### Exercise 4: Improve an alert

The current notification only says, “Error occurred.”

Name three useful details to include, if they are available in the failed node's output.

### Answer key

1. **Exercise 1:** Use **Retry On Fail** for a potentially temporary problem. Limit retries to avoid unnecessary repeated requests and long execution delays.
2. **Exercise 2:** Do not repeatedly retry the same unsupported file. Filter it out, convert it if appropriate, or route it to an alternate handler and log the result.
3. **Exercise 3:** Use a continue mode, such as a separate error output where supported. Log or notify about record 4, then make sure the error path reconnects to the intended loop path if processing should continue.
4. **Exercise 4:** Examples include the failed record ID, filename or URL, error message, operation name, and timestamp—only when those values are available.

---

## Key takeaways

- Error handling decides what happens when a workflow step fails.
- **Retry On Fail** is useful for potentially temporary problems, not permanent invalid inputs.
- **On Error** determines whether the workflow stops or continues, and may provide a separate error output.
- Error paths should record or report failures with enough context to investigate.
- In loops, connect the success and error paths intentionally so one failed item does not unintentionally prevent later items from being processed.
- Test both successful and failing cases before relying on the workflow.
