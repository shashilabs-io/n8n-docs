# Master Conditional Logic in n8n
## If-Else, Execution Order, Branching, and Merging

## What you will learn

By the end of this chapter, you should be able to:

- Explain conditional logic using the If and Filter nodes.
- Distinguish conditional, multi-path, and parallel branching.
- Understand the execution order of branches in an n8n workflow.
- Merge data from different branches into one dataset.
- Choose a suitable workflow structure for different business requirements.

---

## 1. Understand conditional logic

Conditional logic allows a workflow to make decisions based on incoming data.

It follows a simple pattern:

- If a condition is true, perform one set of actions.
- Otherwise, perform a different set of actions.

Examples:

- If a support ticket is urgent, notify the manager; otherwise, assign it normally.
- If a customer opens an email, send a follow-up; otherwise, add them to another sequence.
- If a lead belongs to a high-priority industry, send it to the fast-track sales pipeline; otherwise, use the normal pipeline.

In n8n, conditional logic helps workflows respond differently to different situations.

## 2. If node versus Filter node

Both nodes evaluate conditions, but they handle the results differently.

### If node

The **If** node routes incoming items into two outputs:

- **True:** items that satisfy the configured conditions.
- **False:** items that do not satisfy the conditions.

Use it when both outcomes may require different actions.

### Filter node

The **Filter** node keeps items that match the configured conditions and discards items that do not match. It does not provide separate true and false workflow branches.

Use it when you only want matching items to continue through the workflow.

### Example: qualify a sales lead

Suppose a lead is qualified only when all these conditions are true:

- Company name is not empty.
- Email does not end with `gmail.com`.
- Email does not end with `hotmail.com`.

Conceptually:

```text
Company name is not empty
AND email does not end with "gmail.com"
AND email does not end with "hotmail.com"
```

With an **If** node, qualified leads go through the True output, while other leads go through False. With a **Filter** node, only qualified leads continue; non-matching items are discarded from that path.

These are example business rules, not universal qualification criteria. Choose rules that fit the actual business.

## 3. Understand the three types of branching

Branching means splitting a workflow so data can follow different paths or trigger different actions.

### A. Conditional branching

Conditional branching uses an If-style decision to divide items into two outcomes.

```text
Incoming lead
     |
     v
    If
   /  \
True  False
 |      |
Notify  Review or
sales   reject lead
```

Use conditional branching when the workflow needs different actions depending on whether a condition is true or false.

### B. Multi-path branching

Multi-path branching sends data into different paths based on several possible values. The **Switch** node is useful for this pattern.

For example, an order workflow may route orders based on status:

- `pending` → send an email.
- `processing` → send a Slack message.
- `cancelled` → send an email and a Slack message.
- `refunded` → send an email and a Slack message.

```text
             Order
               |
               v
            Switch
       /      |       |       \
  Pending  Processing Cancelled Refunded
     |         |         |         |
   Email     Slack    Email +    Email +
                      Slack      Slack
```

The Switch node can create multiple output paths according to the rules you configure.

### C. Parallel branching

Parallel branching occurs when one node connects to multiple downstream actions. The same incoming data can be used by more than one action.

For example, a cancelled order may need both an email and a Slack notification.

```text
       Cancelled order
              |
        +-----+-----+
        |           |
        v           v
      Email       Slack
```

Use parallel branching when the same event requires multiple actions.

## 4. Understand execution order

When a workflow has multiple branches, execution order matters—especially when one path depends on data or results from another.

The lesson's example describes these general canvas-order rules:

1. Branches are processed in sequence.
2. Higher branches are processed before lower branches.
3. When branches are at the same height, the leftmost branch is processed first.

The exact execution behavior can depend on the workflow structure and n8n version, so test workflows that rely on a specific order.

### Why execution order matters

Suppose one branch prepares data that another branch needs. If the dependent path runs before the required data is ready, the workflow may produce unexpected results.

Before running a multi-branch workflow, ask:

- Does one branch rely on output from another?
- Are the nodes connected so that dependencies are explicit?
- Does the execution view confirm the order you expect?

Avoid relying only on where nodes appear visually. Use the correct connections and a Merge node when the workflow must bring branch data together.

## 5. Merge data from different branches

A **Merge** node combines data coming from multiple inputs. It is useful when separate branches retrieve related information that later steps need together.

### Example: combine order headers and order details

Imagine two Google Sheets nodes retrieve different parts of an order:

**Order headers**

```json
[
  {
    "orderId": "ORD-101",
    "firstName": "Riya",
    "lastName": "Verma",
    "status": "pending"
  }
]
```

**Order details**

```json
[
  {
    "orderId": "ORD-101",
    "productName": "Keyboard",
    "quantity": 1,
    "totalPrice": 1200
  }
]
```

The order header contains customer and order information. The order details contain product and pricing information. To use both in later steps, combine records using the shared `orderId` field.

Conceptually, the merged record could look like:

```json
[
  {
    "orderId": "ORD-101",
    "firstName": "Riya",
    "lastName": "Verma",
    "status": "pending",
    "productName": "Keyboard",
    "quantity": 1,
    "totalPrice": 1200
  }
]
```

### Configure the Merge node

1. Connect the order-header source to one Merge input.
2. Connect the order-details source to the other Merge input.
3. Choose the merge operation that combines matching records.
4. Select `orderId` as the matching field in both inputs.
5. Execute the node and inspect the merged output.

The precise option names can vary by n8n version. Confirm that the selected operation matches records by the shared key.

If one order contains several product rows, the merged output may contain several rows for that order—one per matching product detail. That is expected when the data relationship is one-to-many.

## 6. Optimize workflows by combining shared actions

Suppose both cancelled and refunded orders need the same email and Slack notification. You can avoid duplicating the same action nodes by routing both types into a common path before those actions, where the workflow structure and required data allow it.

A possible pattern is:

```text
Switch: Order Status
       |                 |
       v                 v
 Cancelled             Refunded
       |                 |
       +--------+--------+
                |
                v
       Combine common path
                |
                v
       Remove Duplicates
                |
                v
             Aggregate
             /       \
            v         v
          Email     Slack
```

The exact nodes needed depend on the data. Use **Remove Duplicates** when repeated items must be removed, and **Aggregate** when multiple items should be collected into a summary. Neither node replaces the status-routing logic.

When merging branches, make sure the selected merge behavior fits the data and that the downstream nodes receive the expected number and shape of items.

## 7. Troubleshooting common problems

| Problem | What to check |
|---|---|
| Items do not reach the expected If output | Review the conditions and whether they are combined with AND or OR. |
| Non-matching items disappear | This is expected behavior for Filter; use If if both outcomes need separate paths. |
| The Switch node does not create the expected route | Check the configured values and each output rule. |
| A downstream node runs with missing data | Verify dependencies and inspect the execution data for each branch. |
| Merge produces too many rows | Check whether the matching key has multiple matches in either input. |
| Merge does not combine records | Confirm the matching fields and the chosen merge operation. |
| The same notification appears more than once | Check whether multiple paths send the same action and whether deduplication is required. |
| Workflow behavior differs from the expected canvas order | Inspect the execution view and test with a small dataset. |

## 8. Practice exercises

### Exercise A — If or Filter?

You need to send qualified leads to sales and send unqualified leads to a review queue. Should you use If or Filter?

### Exercise B — Choose a branching type

An order workflow must send `pending`, `processing`, `cancelled`, and `refunded` orders down different paths. Which node is suitable?

### Exercise C — Parallel actions

Every cancelled order must trigger both an email and a Slack notification. What branching pattern would you use?

### Exercise D — Merge two datasets

One input contains customer and order information. Another contains product details. Both include `orderId`. Which node can combine matching records?

### Exercise E — Diagnose the output

One order has three product-detail rows. After merging by `orderId`, the order appears three times. Why might this be correct?

### Suggested answers

1. Use **If**, because qualified and unqualified leads need different paths.
2. Use **Switch** for multi-path branching based on order status.
3. Use parallel branching to connect the cancelled-order path to both actions.
4. Use the **Merge** node and match records using `orderId`.
5. One order can match three product rows. The merged dataset may therefore contain three rows, one for each product detail.

## Key takeaway

Conditional logic decides which items should follow which paths. **If** separates true and false outcomes, **Filter** keeps only matching items, **Switch** supports multiple routes, parallel branching runs multiple actions from the same data, and **Merge** combines related data from different inputs. Use these concepts together to build workflows that are clear, reliable, and easier to maintain.
