# How to Use the Merge Node in n8n
## A practical guide with an end-to-end example

### What you will learn
By the end of this lesson, you should be able to:
- Explain why workflows sometimes need to combine data from two paths.
- Understand the difference between **Append**, **Combine**, and **Choose Branch**.
- Configure a Merge node using two input connections.
- Understand matching fields, position-based matching, and field clashes.
- Recognize when a Merge node is unnecessary and when it is useful.

---

## 1. Start with a business problem

Imagine you run an online store. When a new order arrives, your workflow gets two kinds of information:

1. **Order details** from your store system: order ID, customer ID, and total amount.
2. **Customer details** from a customer database: customer ID, name, and email address.

The order system knows what the customer bought, but it may not include the customer's latest contact details. The customer database knows the name and email, but it does not know the details of this particular order.

You want one final record that contains both sets of information so you can send a personalized confirmation email.

For example, the order path might contain:

```json
{
  "orderId": "ORD-1042",
  "customerId": "C-17",
  "total": 2499
}
```

The customer lookup path might contain:

```json
{
  "customerId": "C-17",
  "name": "Aarav Sharma",
  "email": "aarav@example.com"
}
```

The desired result is one combined record:

```json
{
  "orderId": "ORD-1042",
  "customerId": "C-17",
  "total": 2499,
  "name": "Aarav Sharma",
  "email": "aarav@example.com"
}
```

This is the problem the **Merge** node can help solve: bringing data from different inputs together. The exact result depends on the Merge mode and how you configure it.

## 2. Understand the workflow before configuring the node

A workflow could look like this:

```text
                 ┌──> Get Order Details ─────┐
Trigger ─────────┤                            ├──> Merge ──> Send Email
                 └──> Look Up Customer ──────┘
```

Both paths must provide data to the Merge node. The Merge node has two inputs, commonly called **Input 1** and **Input 2**.

In this example:
- Input 1 contains the order.
- Input 2 contains the customer record.
- Merge produces output based on the selected mode.

**Important:** a Merge node does not magically know which records belong together. You must choose a mode that matches your data and intended result.

## 3. Add and connect the Merge node

1. Add the nodes that retrieve the two sets of data.
2. Add a **Merge** node to the canvas.
3. Connect the order-details path to **Input 1**.
4. Connect the customer-lookup path to **Input 2**.
5. Open the Merge node and review its **Mode** options.
6. Choose the mode that matches what you want to happen to the incoming items.
7. Test the upstream nodes first, then test the Merge node and inspect its output.

The node's available options can vary by n8n version. The key concepts below describe common Merge modes and their intended uses.

## 4. Mode one: Append

### What it does

**Append** places the items from the two inputs into one output list. It does not combine the properties of two records into one object.

Suppose Input 1 contains:

```json
[
  { "orderId": "ORD-1042", "total": 2499 }
]
```

And Input 2 contains:

```json
[
  { "customerId": "C-17", "name": "Aarav Sharma" }
]
```

With Append, the result is conceptually:

```json
[
  { "orderId": "ORD-1042", "total": 2499 },
  { "customerId": "C-17", "name": "Aarav Sharma" }
]
```

There are now two items in the output list. The two objects have not become one object.

### When to use Append

Use it when you want to bring items from both inputs into the same output stream—for example, collecting records from two sources for a later step that can process them in a common way.

**Remember:** Append combines the lists, not the fields of each record.

## 5. Mode two: Combine

### What it does

**Combine** is intended for cases where items from Input 1 and Input 2 should be paired and their data combined into output items. You must decide how the records should be matched.

Common ways to match items include:

- **Matching fields:** pair records when a selected field has the same value in both inputs.
- **Position:** pair the first item in Input 1 with the first item in Input 2, the second with the second, and so on.

Exact option labels and additional behaviors may differ by n8n version.

### A. Combine by matching fields

Return to the online-store example. Both paths have a `customerId` field:

Input 1:

```json
{
  "orderId": "ORD-1042",
  "customerId": "C-17",
  "total": 2499
}
```

Input 2:

```json
{
  "customerId": "C-17",
  "name": "Aarav Sharma",
  "email": "aarav@example.com"
}
```

Configure the Merge node to combine items by matching `customerId` in Input 1 with `customerId` in Input 2.

Because both values are `C-17`, the records can be paired. The output can then contain the order fields and customer fields together.

### Why a matching key matters

Suppose Input 1 contains an order for `C-17`, but Input 2 contains customer `C-29`. Those records should not be combined as though they belong to the same person. A stable identifier such as a customer ID is generally safer than matching on a name, because different people can have the same name.

### B. Combine by position

Position-based matching pairs items according to their order in the lists:

| Input 1 position | Input 2 position | Pairing |
|---|---|---|
| First item | First item | Pair 1 |
| Second item | Second item | Pair 2 |
| Third item | Third item | Pair 3 |

This is appropriate only when the two inputs are intentionally aligned in the same order and represent corresponding records.

**Potential pitfall:** if the first list is sorted by order date but the second list is sorted by customer name, pairing by position can combine unrelated records. Use a shared identifier when possible.

## 6. What happens when both inputs contain the same field?

Imagine both inputs contain a field named `status`.

Input 1:

```json
{
  "orderId": "ORD-1042",
  "status": "processing"
}
```

Input 2:

```json
{
  "customerId": "C-17",
  "status": "active"
}
```

Both records use the same field name for different meanings. When combined, there is a field-name conflict. The exact result depends on the Merge configuration and version.

A good data-design practice is to make the meaning explicit before or during the merge, for example:
- `orderStatus`
- `customerStatus`

Rename ambiguous fields upstream when needed. Then inspect the Merge output to verify which values were preserved.

## 7. Mode three: Choose Branch

### What it does

**Choose Branch** is useful when the workflow has two input paths but you want to pass data from one selected input forward, rather than combining both inputs into one result.

For example, a workflow might perform a check on two paths, but a later step should continue with the data from Input 1. Choose Branch can be used to select which input's data is passed along, depending on the configuration available in your n8n version.

This mode is different from Combine:
- **Combine:** bring related data from both inputs together.
- **Choose Branch:** pass through the selected input's data.

Use the node's options and help text in your installed n8n version to confirm exactly how the selected input and execution behavior are configured.

## 8. A practical walkthrough: enrich an order with customer data

Let's build the example end to end.

### Step 1 — Get the order

Use a trigger or order source that outputs a record such as:

```json
{
  "orderId": "ORD-1042",
  "customerId": "C-17",
  "total": 2499
}
```

### Step 2 — Look up the customer

Use a database or other customer-data source to retrieve the record matching `customerId = C-17`:

```json
{
  "customerId": "C-17",
  "name": "Aarav Sharma",
  "email": "aarav@example.com"
}
```

The actual lookup node and query depend on your database and workflow.

### Step 3 — Connect both paths to Merge

- Connect the order record to Input 1.
- Connect the customer record to Input 2.
- Select **Combine**.
- Choose matching fields and select `customerId` on both inputs.

### Step 4 — Test the Merge node

Run the upstream paths and execute the Merge node. Inspect the output JSON and confirm that the result includes:
- `orderId`
- `customerId`
- `total`
- `name`
- `email`

If the customer information is missing, check:
1. Did the customer lookup return an item?
2. Do both records contain the matching field?
3. Do the matching values have the same type and value?
4. Did you choose the correct fields for Input 1 and Input 2?
5. Does your Merge mode keep only matched records or handle unmatched items differently?

### Step 5 — Use the merged data

Connect the Merge node to an email node. The email node can now use the order ID and total alongside the customer's name and email address.

For example, the message could say:

> Hi Aarav, your order ORD-1042 has been received. Your order total is ₹2,499.

This is only an example message. Configure the email node with your own provider credentials, sender address, and recipient mapping.

## 9. Common mistakes and troubleshooting

| Symptom | What to check |
|---|---|
| Output has two separate items instead of one enriched record | You may have selected Append instead of Combine. |
| The Merge node returns no matched output | Verify the matching fields and their values on both inputs. |
| Wrong customer data appears on an order | Avoid position matching unless both inputs are deliberately aligned; prefer a shared ID. |
| Some items disappear | Review the selected matching behavior and how unmatched items are handled in your version. |
| One input is empty | Test the upstream node and confirm it returns the expected item(s). |
| Duplicate or conflicting values appear | Check for same-named fields and rename ambiguous fields before merging. |
| The node behaves differently from a tutorial | Check your n8n version and the options shown in your Merge node. |

## 10. Quick comparison of the modes

| Mode | Main purpose | Simple way to remember it |
|---|---|---|
| Append | Put items from both inputs into one output list. | One list, separate items. |
| Combine | Pair items from the two inputs and combine their data. | Match records and bring fields together. |
| Choose Branch | Pass data from the selected input onward. | Choose which input to continue with. |

## 11. Practice exercises

### Exercise A — Predict the result

Input 1:

```json
[
  { "id": 1, "product": "Keyboard" },
  { "id": 2, "product": "Mouse" }
]
```

Input 2:

```json
[
  { "id": 1, "price": 1500 },
  { "id": 2, "price": 700 }
]
```

1. What is the conceptual output when you use Append?
2. What would you expect when using Combine by matching `id`?
3. Why would position matching be risky if Input 2 were sorted differently?

### Exercise B — Design the merge

A support workflow has:
- Input 1: `{ "ticketId": "T-81", "customerId": "C-17", "issue": "Login problem" }`
- Input 2: `{ "customerId": "C-17", "name": "Aarav Sharma", "plan": "Pro" }`

Choose a Merge mode and matching field so the support team can see the ticket and customer details together.

### Exercise C — Improve field names

Input 1 has `status: "open"` for a ticket. Input 2 has `status: "active"` for a customer account.

Explain why those names can be confusing after a merge, and suggest clearer field names.

### Answer key

**Exercise A**
1. Append returns the items from both inputs in one list; it does not combine each pair into one object.
2. Matching on `id` pairs the keyboard with its price and the mouse with its price, assuming the matching behavior is configured accordingly.
3. Position matching relies on order, so mismatched sorting could pair the wrong product and price.

**Exercise B**
Use Combine and match `customerId` in Input 1 with `customerId` in Input 2.

**Exercise C**
Use names such as `ticketStatus` and `customerAccountStatus` so each field's meaning remains clear.

---

## Key takeaway

Use **Append** when you need one list containing items from both inputs, **Combine** when you need related records merged together, and **Choose Branch** when you need to continue with data from a selected input. The most important decision is choosing a mode that matches the shape of your data and the result you want.
