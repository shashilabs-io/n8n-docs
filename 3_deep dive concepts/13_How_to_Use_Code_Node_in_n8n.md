# Chapter 13: How to Use the Code Node in n8n
## Data structures, execution modes, and limitations

The Code node lets you write custom JavaScript or Python code inside an n8n workflow. It is useful when expressions are not enough—for example, when you need multi-step transformations, aggregation, or a completely new output dataset.

![Code Node Hero](code%20node.png)

## Learning objectives

By the end of this chapter, you should be able to:

- Explain the purpose of the Code node.
- Choose between **Run Once for All Items** and **Run Once for Each Item**.
- Access input data according to the selected mode.
- Understand the output structure expected by n8n.
- Aggregate multiple order rows into totals per order.
- Recognize the Code node limitations described in this chapter.

## 1. What is the Code node?

The Code node is a single workflow step where you can write multi-line custom code to manipulate data, filter records, or generate new objects.

There are two languages supported by n8n

- **JavaScript** — used throughout the practical examples.
- **Python** 

Expressions are generally suited to compact, single-field transformations. The Code node is useful when the logic needs multiple statements or a custom algorithm.

## 2. Choose the execution mode

The Code node has two important modes.

| Mode | How it works | Typical use |
|---|---|---|
| **Run Once for All Items** | Runs the code once with the full set of input items available | Summarizing, grouping, aggregating, or transforming a whole dataset |
| **Run Once for Each Item** | Runs the code separately for each incoming item | Applying a transformation independently to every item |

For example, if 291 items arrive at a Code node:

- **Run Once for All Items:** the code executes once and can inspect all 291 items.
- **Run Once for Each Item:** the code executes separately for each of the 291 items.

Choose the mode based on the shape of the result you need, not simply on which mode appears first.

## 3. Access input data

The way you access input data depends on the selected mode.

### Run Once for All Items

In JavaScript, use `$input.all()` to access the incoming items:

```javascript
const items = $input.all();
```

The result is an array of n8n items. Each item commonly has a `json` property containing its JSON data.

Example:

```javascript
const items = $input.all();

for (const item of items) {
  const orderId = item.json.orderId;
  const totalPrice = item.json.totalPrice;
}
```

### Run Once for Each Item

In JavaScript, `$json` is a convenient way to access the current item's JSON data in this mode:

```javascript
const orderId = $json.orderId;
const totalPrice = $json.totalPrice;
```

Use the variables and input helpers shown by the Code node's editor for the selected mode and n8n version.

### JavaScript and Python naming

JavaScript uses `$`-prefixed helpers, while the Python option exposes corresponding helpers with underscore-style names. Python support and its limitations may change across n8n versions, so check the current Code node documentation before relying on a specific Python feature.

## 4. Understand the expected output structure

A key rule is that n8n passes data between nodes as a collection of items. In the common JSON case, each item wraps its data under a `json` property.

Example of two output items:

```javascript
return [
  {
    json: {
      fruit: "apple",
      quantity: 2
    }
  },
  {
    json: {
      fruit: "carrot",
      quantity: 1
    }
  }
];
```

Even if the Code node creates only one output item, return it inside an array:

```javascript
return [
  {
    json: {
      message: "Processing complete"
    }
  }
];
```

The returned data must follow the item structure n8n expects. Returning a plain object in the wrong shape can cause an error or make the next node unable to read the expected fields.

## 5. Example: calculate totals for each order

Imagine a spreadsheet has order headers, while another spreadsheet contains order details. One order may contain multiple product rows.

After combining the datasets using a Merge node and matching on `orderId`, the workflow might contain rows like these:

```json
[
  { "orderId": "ORD-001", "totalPrice": 769.60 },
  { "orderId": "ORD-001", "totalPrice": 256.02 },
  { "orderId": "ORD-001", "totalPrice": 226.86 },
  { "orderId": "ORD-002", "totalPrice": 100.00 }
]
```

The goal is to produce one result per unique order:

```json
[
  { "orderId": "ORD-001", "orderTotal": 1252.48 },
  { "orderId": "ORD-002", "orderTotal": 100.00 }
]
```

### JavaScript solution

Set the Code node to **Run Once for All Items**:

```javascript
const items = $input.all();
const orderTotals = {};

for (const item of items) {
  const orderId = item.json.orderId;
  const price = Number(item.json.totalPrice ?? 0);

  if (!orderId) {
    continue;
  }

  if (orderTotals[orderId] === undefined) {
    orderTotals[orderId] = 0;
  }

  orderTotals[orderId] += price;
}

return Object.entries(orderTotals).map(([orderId, total]) => ({
  json: {
    orderId,
    orderTotal: Number(total.toFixed(2))
  }
}));
```

### How the code works

1. `$input.all()` retrieves all incoming items.
2. `orderTotals` stores a running total for each order ID.
3. The loop reads each item's order ID and price.
4. `Number(...)` converts the price to a number; the fallback handles a missing or null value as zero.
5. The first time an order appears, its total starts at zero.
6. Each price is added to that order's total.
7. `Object.entries()` turns the lookup object into key-value pairs.
8. `.map()` creates one n8n output item per order, with the required `json` wrapper.

The example input above produces two output items instead of four. 

**Data-quality note:** Treating a missing price as zero is only appropriate if that matches the business rule. In a production workflow, you may prefer to reject or separately report rows with missing or invalid prices.

## 6. Add a field to each existing item

If your goal is to preserve the original records and add a field, you can map over the input items:

```javascript
return $input.all().map(item => ({
  json: {
    ...item.json,
    processed: true
  }
}));
```

This returns the same number of items and keeps the original JSON fields while adding `processed`.

Use this pattern when you want to enrich records rather than aggregate them into a different dataset.

## 7. Binary data and files

Most examples in this chapter use JSON data. n8n can also pass binary data, commonly associated with files.

 Binary data remain under a `binary` property and notes that file-related metadata may include a file name, MIME type, and extension. When working with files, preserve the binary data structure expected by the nodes in your workflow. Use dedicated file nodes for file operations rather than assuming that ordinary JSON examples cover every binary-data case.

## 8. Code node limitations

Consider these restrictions:

- In n8n Cloud, importing arbitrary external npm packages is restricted.
- Self-hosted installations may allow additional packages when configured appropriately.
- The Code node is not intended to access the local file system directly.
- The Code node is not the node to use for making HTTP requests or API calls.

For tasks outside the Code node's role, use dedicated workflow nodes. For example:

- Use an **HTTP Request** node for HTTP/API requests.
- Use an appropriate **read file** node for supported file-reading operations.

These details can vary with n8n version, deployment type, and configuration. Consult the current official documentation before designing around a specific package or capability.

## 9. Common mistakes

### Returning the wrong data shape

**Problem:** The code returns a plain object or an array of objects without the required `json` wrapper.

**Fix:** Return an array of n8n items, each containing a `json` property for ordinary JSON output.

### Choosing the wrong execution mode

**Problem:** Code that expects the full dataset runs separately for each item, or item-level code is run only once.

**Fix:** Use **Run Once for All Items** for grouping and aggregation. Use **Run Once for Each Item** for independent per-item work.

### Accidentally losing original fields

**Problem:** A mapped output contains only the new field and drops useful input data.

**Fix:** Spread the original JSON object first, as in `{ ...item.json, processed: true }`, when preserving fields is intended.

### Grouping records by the wrong key

**Problem:** Unrelated rows are combined, or rows belonging to the same order remain separate.

**Fix:** Confirm that the grouping key is stable and consistent, such as a reliable order ID.

### Treating invalid prices as zero without checking

**Problem:** Bad input data silently lowers totals.

**Fix:** Validate the price and handle invalid rows according to the workflow's business rules.

## 10. Practice questions

1. What is the main purpose of the Code node?
2. What is the difference between the two execution modes?
3. Which helper accesses all input items in JavaScript?
4. Why must output items use the expected n8n structure?
5. If an order has three product rows priced at 100, 250, and 50, what is its total?
6. Why is a lookup object useful for calculating order totals?
7. How can you add a field while preserving the original JSON data?
8. According to the lesson, which type of node should you use for HTTP requests?

## 11. Answer key

1. To write custom multi-line logic for manipulating, filtering, aggregating, or generating data.
2. One runs once with the full input set; the other runs separately for each item.
3. `$input.all()`.
4. Downstream nodes expect items in a particular structure, commonly an array of objects with a `json` property.
5. 400.
6. It stores a running total under each order ID, allowing multiple rows to be grouped.
7. Map over the items and return `{ json: { ...item.json, newField: value } }` for each one.
8. An HTTP Request node.

## Key takeaways

- Choose the execution mode according to the operation you need.
- Use `$input.all()` when the entire input collection must be processed together.
- Return data in the item structure expected by n8n.
- Aggregation can turn many detail rows into one summary item per group.
- Validate keys and numeric values before calculating totals.
- Use dedicated nodes for HTTP requests and file operations.
