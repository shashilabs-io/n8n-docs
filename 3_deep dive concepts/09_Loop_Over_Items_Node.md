# How to Use the Loop Over Items Node in n8n
## Batch Processing, Controlled Iteration, and Rate Limits

### Learning objectives

By the end of this chapter, you should be able to:

- Explain what the **Loop Over Items** node does.
- Decide when a loop is useful and when normal item processing is enough.
- Configure a batch size and understand the **Loop** and **Done** outputs.
- Build a workflow that processes email attachments one at a time.
- Add a Wait node to reduce the risk of API rate-limit errors.
- Recognize common data-shape and file-type errors.

---

## 1. What is a loop?

A **loop** repeats a set of actions for a collection of items. In n8n, the **Loop Over Items** node splits incoming items into batches and sends each batch through a set of nodes. When the loop work is complete, the workflow can continue through the node's **Done** output.

Loops are useful when you need control over how a collection is processed—for example, when an external API limits request frequency, when you want to handle a large collection in smaller batches, or when each item needs a sequence of actions.

### Typical use cases

- Process records in batches of 10, 20, or 50 instead of sending the entire collection at once.
- Call an API with a delay between batches or requests.
- Upload files one at a time.
- Perform several steps for each item before moving to the next batch.

## 2. How items move through a loop

Conceptually, the node takes the incoming items, sends a batch through the **Loop** output, and repeats until all batches have been processed. The **Done** output is for work that should happen after the loop finishes.

```text
Incoming items
      |
      v
Loop Over Items
   |         |
 Loop       Done
   |         |
   v         v
Process     After-loop
batch       actions
   |
   +---- connect the final loop step back to
         Loop Over Items to continue
```

The **Loop** output connects to the nodes that should run for each batch. After the last node in that path, connect back to the Loop Over Items node so it can continue with the next batch. Once no batches remain, the Done path can run.

A placeholder such as **Replace Me** may appear in the loop path. It is only a placeholder; replace it with the work you want to perform, or remove it if it is not needed.

## 3. Do you always need this node?

No. n8n nodes commonly process all incoming items as part of their normal operation. If you already have five items and a node can safely process each item, a Loop Over Items node with a batch size of one may be unnecessary.

The important difference is **control**. A loop lets you structure a batch-processing path and add steps such as a Wait node between batches. Simply sending five items to an ordinary node does not, by itself, give you a reliable one-request-per-second schedule.

Use Loop Over Items when the workflow needs deliberate batching or a repeated multi-step path. For straightforward item-by-item transformations, a regular node may be simpler.

## 4. Configure the node

1. Add **Loop Over Items** after the node that produces the collection.
2. Set **Batch Size** to the number of items to send through the loop path per iteration.
3. Connect the **Loop** output to the action or sequence of actions to repeat.
4. Connect the last node in that sequence back to **Loop Over Items**.
5. Connect **Done** to any action that should happen only after all batches have finished.
6. Execute the workflow and inspect the number of items at each step.

A batch size of `1` means one item is sent through the loop path per iteration. A batch size of `10` means batches contain up to ten items. The last batch may contain fewer items than the configured size.

---

## 5. Practical example: process email attachments

Imagine an email arrives with five attachments. The goal is to upload image attachments to a task board, creating a page or record for each image.

The workflow can be organized like this:

```text
Gmail Trigger
     |
     v
Filter: Email has attachments
     |
     v
Split Out: attachment data
     |
     v
Loop Over Items (Batch Size = 1)
     |
   [Loop]
     |
     v
Upload file to temporary storage
     |
     v
Create task-board page with image URL
     |
     v
Wait (e.g. 1–2 seconds)
     |
     +------ back to Loop Over Items

   [Done]
     |
     v
Workflow finished / post-loop action
```

The temporary-storage step is needed in this example because the task-board image block expects a URL. A downloaded email attachment is binary data inside n8n and does not automatically have a publicly accessible URL. Use a storage service that you trust and that is appropriate for the file's sensitivity; temporary public upload services may expose files to anyone with the link.

### Step 1: Receive the email and download attachments

Configure a Gmail Trigger to detect incoming messages. In the trigger's options, enable attachment downloading if that option is available in your installed version. The exact fields and labels can vary by n8n version.

Run a test event and inspect the output. Check whether attachment data is present in the item's **Binary** section.

### Step 2: Keep only emails that contain attachments

Use a Filter or IF node to allow only items that contain binary attachment data to continue. Inspect the actual output shape before setting the condition: binary data is not the same as a regular JSON property.

### Step 3: Split attachments into separate items

One email may be represented by a single item that contains several binary properties. To process each attachment independently, use **Split Out** or the appropriate item-splitting approach for the data shape you have. Configure it based on the actual input available in your n8n version.

After splitting, verify that the output contains one item per attachment. Do not assume the number of email items equals the number of attachment items.

### Step 4: Process each attachment in the loop

Set **Batch Size** to `1`. Put the upload and task-board actions in the Loop path.

For each item:

1. Upload the binary file to an appropriate storage location.
2. Read the resulting file URL.
3. Create the task-board page or record and include the URL.
4. Wait before continuing if the destination API requires a pause.
5. Connect the final loop step back to Loop Over Items.

### Step 5: Continue after all attachments

Use the **Done** output for actions that should happen once all attachments have been processed, such as sending a completion notification or recording a summary.

---

## 6. Why add a Wait node?

External services often impose rate limits. A workflow that sends many requests too quickly can receive errors such as HTTP `429 Too Many Requests`.

A Wait node adds a pause in the loop path. For example, a one-second wait can slow down repeated operations. However, a delay is not a guarantee that you comply with every service's limits: the actual limit may be measured across multiple requests, users, or time windows. Always check the target API's documentation and tune the batch size and delay accordingly.

**Example:** If five files are processed with a one-second wait between iterations, the workflow introduces pauses between processing steps rather than trying to perform the entire sequence as quickly as possible.

## 7. Common problems and how to debug them

| Problem | Likely reason | What to check |
|---|---|---|
| The loop only processes one batch | The final loop-path node is not connected back to Loop Over Items | Connect the final step back to the loop node |
| The Done path runs too early or not as expected | The loop connections are incorrect or the execution path is misunderstood | Inspect the Loop and Done connections and test with a few items |
| An upload node says binary data is missing | The binary property name is hard-coded or the expected property is absent | Inspect the current item's Binary tab and use the actual property name |
| A later attachment fails while the first works | Each item may use a different binary property key, such as `attachment_0`, `attachment_1`, etc. | Avoid assuming every item uses the same numbered key; inspect each item and use a dynamic expression where appropriate |
| The task-board API rejects a file | The API may accept only certain file types or formats | Filter unsupported file types before the upload/create step |
| Requests are still rate-limited | The delay or batch size does not match the API's policy | Check the API's documented limit and adjust pacing |
| The workflow is unnecessarily complicated | Normal n8n item processing may already be sufficient | Remove the loop if you do not need controlled batching or a repeated multi-step path |

### Dynamic binary property names

A common source of failure is hard-coding a key such as `attachment_0`. If a later item uses `attachment_1`, the upload node may report that the binary property does not exist.

Inspect the Binary data for each item. Where the node configuration supports expressions, use an expression based on the current item's binary keys instead of assuming a fixed numbered suffix. Test this carefully against the actual shape of the data.

## 8. Best practices

- **Start small:** test the loop with two or three items before processing a large collection.
- **Verify the data shape:** inspect JSON and Binary output before configuring later nodes.
- **Choose the batch size deliberately:** larger batches can improve throughput but may increase pressure on external services.
- **Respect API limits:** use a Wait node where appropriate, but follow the service's actual documented limits.
- **Filter early:** remove unsupported or irrelevant items before expensive upload or API steps.
- **Handle errors intentionally:** decide what should happen if one item fails. A loop alone does not automatically make failed operations safe or retry them.
- **Protect files:** avoid uploading sensitive attachments to public temporary storage. Use an approved storage service and appropriate access controls.
- **Use Done for after-loop work:** do not place one-time summary actions inside the loop unless they should repeat for every batch.

## 9. Practice exercises

### Exercise 1: Batch customer records

A Google Sheets node returns 45 customer items. You want to send them to an API in batches of 15.

1. Which batch size should you configure?
2. How many batches are expected?
3. Where should a final summary notification be connected?

### Exercise 2: Slow down requests

A service rate-limits requests and your workflow processes one file per iteration.

1. Which node can introduce a pause?
2. Why is a pause alone not a universal guarantee against rate-limit errors?

### Exercise 3: Fix the attachment error

The first attachment uploads successfully, but a later one fails because `attachment_0` is missing.

1. What is the likely cause?
2. What should you inspect before changing the expression?

### Answer key

1. **Exercise 1:** Batch size `15`; three batches of 15 items; connect the final notification to **Done**.
2. **Exercise 2:** Use a **Wait** node in the loop path. The service may have limits across different time windows, concurrent requests, or multiple workflows, so check its documentation.
3. **Exercise 3:** The binary property key may differ between items. Inspect each item's Binary output and configure the upload node to use the correct property dynamically where supported.

---

## Key takeaways

- **Loop Over Items** gives you control over batch-based processing.
- The **Loop** output is for repeated work; the **Done** output is for work after all batches finish.
- Connect the last node in the loop path back to Loop Over Items to continue iteration.
- Normal n8n item processing is often enough; use a loop when controlled batching or a repeated multi-step path is needed.
- Use Wait nodes thoughtfully and respect external API limits.
- Inspect binary keys and file types when working with email attachments.
