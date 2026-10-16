# Chapter 11: How to Optimize Workflows in n8n
## Make automations faster, more efficient, and easier to scale

Workflow optimization means achieving the same required result with less unnecessary work. In n8n, this usually means reducing redundant nodes, avoiding repeated API requests, running independent tasks in parallel, and handling loops and merged data carefully.

## Learning objectives

By the end of this chapter, you should be able to:

- Identify and remove unnecessary workflow nodes.
- Recognize tasks that can safely run in parallel.
- Reduce API calls by fetching data once and reusing it.
- Understand when batch APIs are useful.
- Combine related notifications into a single summary where appropriate.
- Compare an unoptimized workflow with an improved design.

## 1. Remove redundant nodes

A node represents an individual step in an automation. Extra nodes can add execution overhead and make a workflow harder to understand and maintain.

Before adding or keeping a node, ask:

1. Does this node transform, filter, route, store, or send data that the workflow actually needs?
2. Can the same result be achieved with an existing node?
3. Is a “do nothing” or pass-through step unnecessary?
4. Is the same operation repeated for each item even though the data could be processed together?

**Example:** If a workflow contains a no-operation node that does not change the data or control flow, remove it when it serves no purpose.

**Important:** Fewer nodes are not automatically better if removing one makes the workflow less reliable or harder to maintain. Optimize unnecessary work, not essential clarity or validation.

## 2. Run independent tasks in parallel

If several actions depend on the same result but do not depend on one another, arrange them as separate branches instead of making one action wait for another.

### Example: qualified lead handling

After a lead is marked as qualified, the workflow needs to:

- Send a message to Slack.
- Create or update a page in Notion.
- Add an entry to Airtable.

These actions can run on separate branches if each only needs the qualified lead data.

```text
                    ┌──> Send Slack message
Qualified lead ─────┼──> Create Notion entry
                    └──> Add Airtable record
```

A sequential design would wait for Slack, then Notion, then Airtable. Independent branches can avoid those unnecessary dependencies.

**Use parallel branches only when the actions are genuinely independent.** If one step needs the output of another, that dependency must remain sequential. Parallel execution does not guarantee that branches finish in a particular order.

## 3. Reduce API calls

An API request is a call from one application or service to another. Repeated calls can increase execution time and may consume rate limits.

### Less efficient pattern

For each of 100 feedback records:

1. Fetch one customer from the CRM or database.
2. Update that customer's feedback.
3. Send a thank-you email.
4. Check the rating.
5. Send a support alert when the rating is low.

Fetching a customer separately for every feedback item can mean 100 customer-lookup requests.

### Improved pattern

1. Fetch all feedback records.
2. Fetch all customer records once.
3. Merge the datasets using a shared field, such as email.
4. Process the merged records for updates and personalized emails.
5. Collect low-rating feedback and send one summary notification.

```text
Fetch feedback ───────┐
                      ├──> Merge by email ──> Update records
Fetch customers ──────┘                    ├─> Send thank-you emails
                                           └─> Filter low ratings
                                                  └─> Build summary
                                                         └─> One support alert
```

The merge operation joins the feedback record with its matching customer record. The example uses email as the matching field; in a real workflow, choose a stable identifier that is reliable for both datasets.

## 4. Use batch APIs when available

Some services allow multiple records to be created or updated in a single API request. This is commonly called a **batch** or **bulk** API.

For example, instead of making 100 individual Airtable update requests, a supported batch endpoint may allow several records to be updated together.

| Approach | Potential effect |
|---|---|
| One request per record | More network requests and overhead |
| Batch several records per request | Fewer requests and potentially faster processing |
| One batch for all records | Only if the service's limits and API design permit it |

Always check the service's current API documentation for maximum batch size, required payload shape, error behavior, and rate limits. A batch request may still need to be split into multiple batches.

Batching is appropriate for record updates when the API supports it. It is not a substitute for genuinely individual actions, such as sending a separate personalized email to each customer.

## 5. Use loops and merges deliberately

Loops are useful when each item needs an individual action. However, repeated operations should be reviewed carefully.

Ask:

- Does this action really need to run once per item?
- Can data be fetched once before processing all items?
- Can a batch API handle multiple records?
- Can multiple alerts be summarized into one message?
- Is the loop needed, or can a node process the incoming list directly?

A merge node can join two datasets, such as feedback and customer records. A loop is useful for repeated item-by-item work. An aggregate or code step can combine many records into one summary when the next action only needs a consolidated result.

Do not remove a loop just to reduce node executions if doing so changes the required behavior.

## 6. Case study: customer feedback processing

Imagine receiving 100 survey responses each day. Each response includes customer information, product feedback, and a rating.

### Required behavior

For each feedback entry:

1. Find the corresponding customer.
2. Add the feedback to the customer's record.
3. Send that customer a thank-you email.
4. Notify the support team if the rating is below 3.

In the example dataset, 38 of the 100 records have ratings below 3.

### Original design

The workflow fetches a customer separately for each feedback item. It then updates the record, sends the email, evaluates the rating, and sends an alert for each low-rated response.

This repeats several steps for every item. The example's execution-count discussion estimates roughly 441 node executions before the workflow is stopped for comparison.

### Optimized design

The improved workflow:

1. Fetches all feedback records.
2. Fetches all customer records in one step.
3. Merges the datasets by email.
4. Updates customer feedback records.
5. Sends individualized thank-you emails.
6. Filters feedback with ratings below 3.
7. Combines those low-rated records into one summary.
8. Sends one support notification containing the summary.

The example reports roughly 206 node executions for this version, compared with about 441 for the original design. This is an illustration from the example workflow, not a universal performance guarantee.

### What improved—and what did not?

- **Customer lookups:** Reduced from one lookup per feedback item to one fetch of the customer dataset.
- **Support notifications:** Reduced from 38 separate alerts to one summary alert.
- **Personalized emails:** Still sent individually, because each customer should receive their own message.
- **Record updates:** Still occur per item in the demonstrated version. A suitable Airtable batch API could reduce the number of update requests further.

The example shows why reducing node executions and reducing elapsed runtime are related but different measurements. The workflow can have fewer executions while still spending time on individual record updates and emails.

## 7. Practical optimization checklist

Before publishing or scaling a workflow, review it with this checklist:

- [ ] Remove nodes that perform no necessary work.
- [ ] Check whether independent actions can run on separate branches.
- [ ] Fetch shared datasets once when practical, rather than repeatedly fetching one record.
- [ ] Merge datasets using a reliable common key.
- [ ] Check whether the destination service offers a batch or bulk API.
- [ ] Combine repeated notifications when one summary is sufficient.
- [ ] Keep personalized or otherwise individual actions separate when required.
- [ ] Verify that optimization has not changed the workflow's business logic.
- [ ] Test with realistic data volumes and inspect execution history.
- [ ] Check API limits and error handling before increasing throughput.

## 8. Common mistakes

### Making every action parallel
Parallel branches are only appropriate when actions do not depend on one another. Preserve real dependencies.

### Assuming fewer nodes always means faster execution
Network latency, API limits, data volume, and the time taken by external services also affect runtime.

### Sending one email to everyone
Combining messages can expose recipients' email addresses or private information. Keep individual customer communication separate unless a safe, appropriate group-message design is intended.

### Sending one alert for every low rating
If the support team only needs a digest, summarize the matching records and send one alert. If every event must trigger immediate action, individual alerts may still be necessary.

### Ignoring API batch limits
Batch APIs can have size limits and may report partial failures. Follow the provider's documentation and handle unsuccessful records.

## 9. Practice questions

1. What is a redundant node? Give one example.
2. When is it appropriate to use parallel branches?
3. Why can fetching all customer records once be more efficient than fetching each customer separately?
4. What is a batch API, and when might it help?
5. In the case study, why are thank-you emails still sent individually?
6. How does merging feedback and customer records help the workflow?
7. If 38 feedback records have ratings below 3, how can the workflow send one support notification rather than 38?
8. Why should execution count and total runtime be evaluated separately?

## 10. Answer key

1. A node that does no necessary work or duplicates an existing operation; for example, an unnecessary no-operation step.
2. When the actions use the same available input and do not depend on each other's outputs.
3. It avoids repeated network requests for customer lookups and allows the records to be matched locally in the workflow.
4. A way to send or update multiple records in one API request, when the service supports it and the payload respects its limits.
5. Each customer needs a personalized message, and combining recipients can expose private contact information.
6. It associates each feedback entry with its customer record using a common field such as email.
7. Filter ratings below 3, combine the matching records into a summary, and send one notification.
8. Fewer node executions can reduce overhead, but external requests and individual emails can still take substantial time.

## Key takeaways

- Remove unnecessary work, not essential logic.
- Parallelize independent actions.
- Avoid repeated data-fetching API calls.
- Prefer batch APIs for bulk record operations when supported.
- Use merge, loop, and aggregation steps for the job they are best suited to.
- Optimize for correctness, maintainability, and real measured performance—not just a smaller node count.
