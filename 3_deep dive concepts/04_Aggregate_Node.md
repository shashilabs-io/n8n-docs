# How to Use the Aggregate Node in n8n
## A practical guide with an order-notification example

### What you will learn
By the end of this lesson, you should be able to:
- Explain why the Aggregate node is useful when many items need to become one summary item.
- Aggregate selected fields into a list.
- Understand the difference between many input items and one output item containing a list.
- Use aggregated data to send one summary email instead of many individual emails.
- Recognize why aggregation alone does not remove duplicates or replace filtering conditions.

---

## 1. Start with a business problem

Imagine an online store has dozens of orders that are still pending. A workflow checks each order and sends a message to the customer-service team.

If there are 41 pending order items, a direct connection to an email node can result in 41 separate emails—one for each incoming item. That may be technically correct, but it can clutter the team's inbox.

A better approach is to send one summary email that lists the relevant order IDs.

The **Aggregate** node helps with this by collecting data from multiple incoming items and producing a single output item. That item can contain a list of values, such as all pending order IDs.

> Important distinction: aggregation groups data; it does not automatically remove duplicate values. If an order ID appears more than once, it may still appear more than once in the aggregated list.

## 2. Understand the input data

Suppose a previous node returns three items:

```json
[
  { "orderId": "ORD-001", "status": "pending" },
  { "orderId": "ORD-002", "status": "pending" },
  { "orderId": "ORD-003", "status": "pending" }
]
```

Each object is a separate item. If you connect these three items directly to an email node, the node normally processes each item, potentially sending three emails.

The goal is to turn the three items into one item that contains an array of order IDs.

## 3. What the Aggregate node does

The Aggregate node collects values from multiple items. In the order example, it can collect the `orderId` field from every incoming item and put those values into one list.

Conceptually, the output can look like this:

```json
[
  {
    "pendingOrderIds": [
      "ORD-001",
      "ORD-002",
      "ORD-003"
    ]
  }
]
```

Notice the two levels:

- The **outer list** still exists because n8n passes data between nodes as a list of items.
- Inside that outer list is **one item**.
- That item contains the `pendingOrderIds` array.

So three input items have become one output item. This is the central idea of aggregation.

## 4. Add and configure the Aggregate node

A simple workflow can be:

```text
Get Orders
    |
    v
Filter or Switch: Pending Orders
    |
    v
Aggregate Order IDs
    |
    v
Send Summary Email
```

The filter or Switch node is responsible for selecting the relevant orders. The Aggregate node collects values from the items that reach it.

### Configuration steps

1. Add an **Aggregate** node after the node that outputs the items you want to summarize.
2. Choose the mode for aggregating selected fields, rather than aggregating all item data.
3. Select the `orderId` field as the value to collect.
4. Name the output field `pendingOrderIds` so its purpose is clear.
5. Test the node and inspect its output.

The exact option labels may differ by n8n version. Look for the mode that aggregates specified fields into a single item.

### Before and after

| Before aggregation | After aggregation |
|---|---|
| 41 input items | 1 output item |
| Each item contains an order ID | One item contains a list of order IDs |
| Direct email connection can send many emails | Email node can send one summary email |

## 5. Use the aggregated list in an email

After testing the Aggregate node, connect it to the email node.

Instead of using a subject and body intended for one order, use a summary format.

**Example subject:**

```text
Pending orders summary
```

You can include the current date/time in the subject if that is useful for the team. The n8n expression `{{ $now }}` represents the current date/time in expressions; check the formatting you want for your workflow.

**Example email body:**

```text
Please contact customers to collect the missing details for these pending orders:

{{ $json.pendingOrderIds }}
```

Depending on how the email node renders arrays, the list may need formatting to make it easier to read. Inspect the output and adapt the expression or formatting to your node's expected input.

### What changed?

- Before aggregation, 41 items reached the email node, so it could send 41 emails.
- After aggregation, one item reaches the email node, containing a list of the order IDs.
- The email node can now send one message containing the summary.

The Aggregate node is not itself an email-sending node. It changes the shape and number of items passed to the next node.

## 6. Aggregating selected fields versus all item data

The lesson introduces two broad ways to aggregate:

### Option A — Aggregate selected fields

Use this when you only need certain values, such as all order IDs.

Example:
- Input items each contain order ID, customer name, email, status, and date.
- You select only `orderId`.
- The output contains a list of order IDs inside one item.

This is useful when a summary should include only a specific piece of information.

### Option B — Aggregate all item data into one item

Use this when you want to collect the incoming item data more broadly into a single output item. The resulting structure is different from simply selecting one field, so inspect the output carefully and choose this option only when it fits what the next node expects.

For the lesson's email example, selected-field aggregation is sufficient because the email only needs a list of order IDs.

## 7. An important workflow improvement: check all required conditions

Suppose the operations team should be notified only about **high-priority orders that are still pending or processing**.

It is not enough to check only whether `priority` is `high`. A high-priority order might already be shipped, delivered, canceled, or refunded.

The logic needs to check both:
- Priority is high.
- Status is pending or processing.

Conceptually:

```text
Priority = high
AND
(Status = pending OR Status = processing)
```

Use a Filter, If, or Switch-based design to apply the conditions before sending items to the operations team. The exact node arrangement depends on your workflow.

**Why discuss this in an Aggregate lesson?** The Aggregate node summarizes the items it receives. If the upstream logic sends the wrong items, aggregation will summarize the wrong set. Filtering and aggregation solve different problems.

## 8. Aggregation does not mean deduplication

Imagine the input contains:

```json
[
  { "orderId": "ORD-001" },
  { "orderId": "ORD-001" },
  { "orderId": "ORD-002" }
]
```

Aggregating `orderId` can produce:

```json
[
  {
    "pendingOrderIds": [
      "ORD-001",
      "ORD-001",
      "ORD-002"
    ]
  }
]
```

The repeated ID may remain in the array. Aggregation collects values; it does not necessarily make those values unique.

If you need one entry per unique order ID, add a deduplication step or use a suitable uniqueness operation before or after aggregation. Verify the result rather than assuming aggregation removes duplicates.

## 9. Common mistakes and troubleshooting

| Problem | What to check |
|---|---|
| The email node still sends multiple emails | Check how many items leave Aggregate. The desired summary pattern should normally output one item. |
| The list of order IDs is missing | Verify the selected field name and inspect the incoming items. |
| The email shows an awkward array representation | Format the array for the email body and test the rendering. |
| Duplicate order IDs remain | Add a deduplication step; Aggregate does not guarantee unique values. |
| The summary includes shipped or delivered orders | Correct the upstream status and priority conditions before aggregation. |
| The output shape is unexpected | Confirm whether you selected specific fields or aggregated all item data. |
| The node output contains one item but the array has many values | This is expected: one item can contain a list of many values. |

## 10. Practice exercises

### Exercise A — Predict the output

Input:

```json
[
  { "ticketId": "T-10", "team": "Support" },
  { "ticketId": "T-11", "team": "Support" },
  { "ticketId": "T-12", "team": "Support" }
]
```

If you aggregate the `ticketId` field into an output field named `ticketIds`, what should the conceptual output contain?

### Exercise B — Reduce notification noise

A workflow has 25 pending orders and currently sends 25 emails to the service team. Describe where you would put the Aggregate node and what field you would aggregate to send one summary email.

### Exercise C — Find the problem

A workflow aggregates high-priority orders and then sends the list to operations. The summary includes orders that are already delivered. What is wrong with the workflow logic?

### Exercise D — Handle duplicates

Your aggregated list contains `ORD-001` twice. Does that necessarily mean the Aggregate node failed? What should you do if the summary should include each order only once?

### Answer key

**Exercise A**

```json
[
  {
    "ticketIds": ["T-10", "T-11", "T-12"]
  }
]
```

**Exercise B**

Put Aggregate after the node that selects pending orders and before the email node. Aggregate the order ID field into one list, then send the resulting single item to the email node.

**Exercise C**

The upstream logic checked priority but did not also restrict the order status. Add conditions so only high-priority orders with an allowed status reach Aggregate.

**Exercise D**

No. Aggregate collects values and may preserve duplicates. Add a deduplication step or uniqueness logic and verify the resulting list.

---

## Key takeaway

The Aggregate node is useful when many incoming items should become one summary item containing a collection of values. Place it after the relevant filtering or routing logic and before the node that should send or process the summary. Remember: **aggregate to collect, filter to select, and deduplicate to remove repeats.**
