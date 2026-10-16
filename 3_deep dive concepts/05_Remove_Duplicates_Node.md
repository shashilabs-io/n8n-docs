# How to Use the Remove Duplicates Node in n8n
## A practical guide to cleaning repeated records

### What you will learn
By the end of this lesson, you should be able to:
- Explain why duplicate records appear in automation workflows.
- Use the Remove Duplicates node to identify repeated items.
- Understand the difference between duplicates in the current input and records seen in earlier executions, where supported by the node's configuration.
- Choose a suitable comparison basis for your data.
- Check the output to ensure that the intended records remain.

---

## 1. Start with a practical problem

Imagine an online store has these order records:

```json
[
  { "orderId": "ORD-001", "customerEmail": "riya@example.com" },
  { "orderId": "ORD-002", "customerEmail": "kabir@example.com" },
  { "orderId": "ORD-001", "customerEmail": "riya@example.com" },
  { "orderId": "ORD-003", "customerEmail": "riya@example.com" }
]
```

There are two different kinds of repetition here:

- `ORD-001` appears twice, which may mean the same order record was repeated.
- `riya@example.com` appears in two different orders. That is not necessarily a duplicate order—one customer can place multiple orders.

This is the first lesson to understand: **what counts as a duplicate depends on what you are trying to identify**. If you want one record per order, compare order IDs. If you want one notification per customer, compare customer email addresses. Those are different business rules.

The **Remove Duplicates** node helps detect and remove repeated items according to the comparison options you configure.

## 2. Where the node fits in a workflow

For example, suppose you aggregate order IDs and then want to clean the list before sending a summary:

```text
Get Orders
    |
    v
Filter: Relevant Orders
    |
    v
Aggregate Order IDs
    |
    v
Remove Duplicates
    |
    v
Send Summary
```

The correct placement depends on the data shape. If you have a list of separate order items, remove duplicates from those items before aggregation. If you already have one item containing an array of IDs, check whether your chosen node operation compares individual items or values inside that array. Do not assume it automatically removes duplicates from nested arrays.

A common workflow pattern is to clean repeated records before sending notifications, writing to a database, or processing a batch.

## 3. Add the Remove Duplicates node

1. Add the **Remove Duplicates** node after the node that produces the records you want to check.
2. Open its configuration.
3. Review the available operation or comparison options in your n8n version.
4. Decide what makes two items duplicates: the complete item, a selected field, or another supported comparison.
5. Test the node and inspect which items remain.

The precise UI labels and available options can differ by n8n version. Always use the choices displayed in your installed node and verify the result with a small test dataset.

## 4. Example: remove duplicate order records

Suppose the incoming items are:

```json
[
  { "orderId": "ORD-001", "amount": 1200 },
  { "orderId": "ORD-002", "amount": 700 },
  { "orderId": "ORD-001", "amount": 1200 },
  { "orderId": "ORD-003", "amount": 450 }
]
```

If the goal is to keep only one record per order ID, configure the node to compare the `orderId` field, if that comparison option is available.

Conceptually, the cleaned result would be:

```json
[
  { "orderId": "ORD-001", "amount": 1200 },
  { "orderId": "ORD-002", "amount": 700 },
  { "orderId": "ORD-003", "amount": 450 }
]
```

Only one copy of `ORD-001` remains.

### Why choose a field deliberately?

Imagine these two records:

```json
[
  { "orderId": "ORD-010", "status": "pending" },
  { "orderId": "ORD-010", "status": "processing" }
]
```

If the rule is “one record per order ID,” they represent the same order key even though the statuses differ. If the rule is “remove only completely repeated records,” they are not identical objects. Choose the comparison basis according to the business requirement.

## 5. Example: one notification per customer

Now suppose the business requirement is to send a single reminder per customer, even if that customer has multiple pending orders.

Input:

```json
[
  { "orderId": "ORD-101", "customerEmail": "riya@example.com" },
  { "orderId": "ORD-102", "customerEmail": "riya@example.com" },
  { "orderId": "ORD-103", "customerEmail": "kabir@example.com" }
]
```

If the goal is one item per customer, compare `customerEmail` rather than `orderId`, where supported.

Conceptually, one record per email address remains:

```json
[
  { "orderId": "ORD-101", "customerEmail": "riya@example.com" },
  { "orderId": "ORD-103", "customerEmail": "kabir@example.com" }
]
```

This example illustrates a decision you must make: when two records share the same customer email but have different order IDs, which order details should remain? The node's selected duplicate-handling behavior determines which item survives. If you need to preserve all orders while sending one email per customer, you may need to group or aggregate the orders by customer instead of simply dropping duplicate customer records.

## 6. Duplicates in one input versus duplicates across executions

Depending on the available configuration in your n8n version, the node may support checking duplicates in the current input and/or comparing items against values recorded from previous executions.

These solve different problems:

- **Within the current input:** remove repeated records from the batch being processed now.
- **Across executions:** avoid processing the same identifier again when a workflow runs later, if the relevant tracking option is available and configured.

For example, a daily workflow may receive the same event twice. Within-batch deduplication can remove repetitions in today's input. Cross-execution tracking can help identify an event already handled yesterday. These behaviors are not interchangeable; review the node's current options and test persistence behavior before relying on it.

For long-term business guarantees—such as preventing duplicate payments or enforcing unique database records—also use idempotency controls and database constraints where appropriate. A workflow node alone should not be treated as a complete transactional guarantee.

## 7. Test the node carefully

Use a small input with:
- One record that appears twice.
- Two different records with the same customer email.
- Two records with the same order ID but different status values.

After executing the node, ask:

1. Did the repeated record get removed?
2. Did distinct orders from the same customer remain when that was the goal?
3. Did the node keep the item you expected?
4. Does the output item count match your intended rule?
5. If checking across executions, does a second test run behave as expected?

Inspect both the input and output panels. The item count is useful, but it does not tell the whole story: verify the actual values that remain.

## 8. Common mistakes and troubleshooting

| Problem | What to check |
|---|---|
| Legitimate orders disappear | You may be comparing customer email when you intended to compare order ID. |
| Duplicates remain | Confirm that the duplicate field or comparison method is configured correctly. |
| The wrong copy remains | Check the node's behavior for choosing which duplicate item to keep. |
| Repeated values inside an array remain | Confirm whether the node processes items or elements inside a nested array; you may need to transform the array first. |
| The same event returns on a later workflow run | Check whether cross-execution tracking is available and configured, or store processed IDs in a suitable persistent system. |
| A tutorial shows different options | Your n8n version may have a different interface or feature set. |

## 9. Practice exercises

### Exercise A — Duplicate orders

Input:

```json
[
  { "orderId": "A-1", "total": 500 },
  { "orderId": "A-2", "total": 700 },
  { "orderId": "A-1", "total": 500 }
]
```

You want one record per order. Which field should you use to identify duplicates?

### Exercise B — One email per customer

Input:

```json
[
  { "orderId": "B-1", "email": "sam@example.com" },
  { "orderId": "B-2", "email": "sam@example.com" },
  { "orderId": "B-3", "email": "lee@example.com" }
]
```

You want one notification per customer. Which field is the likely comparison key? What information might be lost if you simply remove duplicate customer emails?

### Exercise C — Array versus item

A single item contains:

```json
{
  "orderIds": ["A-1", "A-1", "A-2"]
}
```

Why should you inspect the node's behavior before assuming Remove Duplicates will change the array to `["A-1", "A-2"]`?

### Answer key

**Exercise A:** Use `orderId`, because the requirement is one record per order.

**Exercise B:** Use `email`. Simply removing duplicate customer records may discard details of additional orders. If all order details must be retained, group or aggregate orders per customer instead.

**Exercise C:** The node may compare incoming items rather than individual values inside a nested array. You may need to split, transform, or otherwise process the array elements before deduplicating them.

---

## Key takeaway

The Remove Duplicates node is only as useful as the rule you define for what counts as a duplicate. Choose the right key, test which item survives, and distinguish duplicate items from legitimate records that happen to share a customer or another field.
