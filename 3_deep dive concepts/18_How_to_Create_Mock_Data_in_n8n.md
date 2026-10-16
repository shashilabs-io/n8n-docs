# Chapter 18: How to Create Mock Data in n8n

## Learning objectives

By the end of this chapter, you should be able to:

- Explain what mock data is and why it is useful.
- Create test datasets with a mock-data generator.
- Use an AI assistant to generate CSV or JSON test data.
- Insert sample data into an n8n node's editable output.
- Create small sample records with the Edit Fields (Set) node.
- Generate custom datasets with the Code node.
- Understand the limitations of a small built-in training data store.

## 1. What is mock data?

**Mock data** is fake, sample, or demonstration data used while building and testing an application or workflow. It resembles the kind of information a workflow will receive in real use, but it is not live production data.

For example, a workflow that processes customer sign-ups might be tested with records containing a user ID, name, email, country, and sign-up date.

### Why use mock data?

1. **Reduce repeated external calls:** Test without repeatedly calling an external API or application.
2. **Save time and cost:** Avoid unnecessary API requests during development.
3. **Test more scenarios:** Create records with missing values, unusual inputs, or different transaction types.
4. **Make results predictable:** A small, known dataset makes it easier to understand what each node should produce.
5. **Protect production data:** Keep early experiments separate from live records that could accidentally be overwritten or updated.
6. **Test bulk processing:** Generate many records to see how a workflow handles larger inputs.

> Use clearly fake test data and test destinations. Do not send mock records to real customers or production systems.

## 2. Method 1 — Use a mock-data generator

A mock-data generator lets you define a schema and generate many realistic-looking records. One example covered in this chapter is **[Mockaroo](https://www.mockaroo.com/)** (the platform name may be displayed differently over time).

### Typical workflow

1. Open the generator and choose or define fields.
2. Give each field an appropriate data type or template.
3. Set options such as the percentage of blank values.
4. Set numeric ranges, date ranges, or time ranges where supported.
5. Preview and generate the data.
6. Download the result, such as JSON or CSV.
7. Load the data into a suitable n8n node or into a spreadsheet used by the workflow.

### Example: customer records

A simple customer schema could contain:

- `id`
- `firstName`
- `lastName`
- `email`
- `company`

You might configure 20% of email values to be blank. This creates cases for testing how the workflow handles sign-ups that do not include an email address.

### Generating data with AI-assisted fields

Some generators can suggest fields based on a short description. For example, describing a dataset as **flight logs** or **stock trades** may produce relevant fields and data types.

For a stock-trade dataset, fields might include:

- `tradeId`
- `symbol`
- `transactionType`
- `purchasePrice`
- `salePrice`
- `profit`

A calculated field can derive profit from sale price minus purchase price, if the generator supports formulas.

**Best use:** Choose a dedicated generator when you need a structured dataset with many rows, configurable missing values, ranges, or domain-specific fields.

## 3. Insert downloaded data into n8n

One way to test a workflow without calling its real trigger source is to use a node's editable output.

1. Create or open a workflow.
2. Add a **Manual Trigger** if you want to run a test manually.
3. Execute the trigger once so it has output to edit.
4. Open the node's output editor (the exact UI label may vary by n8n version).
5. Replace the sample output with your mock data in the required format.
6. Save the edited output. In this workflow-testing context, saving may pin the data for later runs.
7. Connect downstream nodes and test their behavior against the sample records.

### Example: split buy and sell transactions

Suppose each record contains a `transactionType` field. An **If** node can check whether the value equals `buy`.

- **True branch:** records where `transactionType` is `buy`.
- **False branch:** records that do not satisfy the condition, such as `sell`.

This makes it easy to verify both branches before connecting a live data source.

> Pinning or editing output is a development aid. It does not mean that the original trigger source has been replaced in production.

## 4. Method 2 — Generate mock data with an AI assistant

An AI assistant can create a dataset from a plain-language specification. State the intended workflow, the fields you need, the approximate number of records, the output format, and any edge cases.

### Copy-and-use prompt

```text
I am testing an n8n workflow that processes new user sign-ups.
Generate a dataset of 100 fake users for testing.

Requirements:
- Return valid JSON as one array of objects.
- Include at least these fields: userId, firstName, lastName,
  email, signupDate, and country.
- Make all people and contact details fictional.
- Leave email blank for some records so I can test missing-email cases.
- Use consistent field names and data types.
- Return only the JSON array, without Markdown fences or explanatory text.
```

You can change `100` to the size needed for your test. If you need a CSV file instead, explicitly request CSV with a header row.

### CSV or JSON?

- **CSV** is convenient for spreadsheets and tabular data.
- **JSON** is often more convenient when a workflow needs structured objects, nested fields, or direct insertion into an n8n output editor.

### Validate the generated data

AI-generated data may be malformed, incomplete, or too large to copy safely. Before using it:

1. Check that the number of records is reasonable.
2. Confirm the required fields exist in every record.
3. Check for missing values intentionally and unintentionally.
4. Validate the JSON syntax.
5. Ensure the structure matches what the n8n node expects.
6. Test with a small subset before processing a large dataset.

### Common JSON format mistake

A node may expect a **single JSON array** of objects, while the generated response contains separate JSON objects one after another.

Incorrect for an array input:

```json
{"userId": 1, "email": "alex@example.test"}
{"userId": 2, "email": ""}
```

Correct array structure:

```json
[
  {"userId": 1, "email": "alex@example.test"},
  {"userId": 2, "email": ""}
]
```

The correct shape depends on the specific n8n editor or node. Some n8n contexts expect an array of items, with each item's data wrapped in a `json` object. If you see a parsing error, ask the AI to return a valid JSON array and then adapt it to the node's expected structure.

### Example: test missing email values

An If node can check whether `email` is present and non-empty. Use one branch for records with an email and another for records without one. This lets you verify that incomplete sign-ups are handled deliberately.

## 5. Method 3 — Use the Edit Fields (Set) node

The **Edit Fields** node (called the **Set** node in some older n8n versions) is useful for creating a small, controlled sample record while a workflow is still being built.

Example input data:

```json
{
  "orderNumber": "ORD-1001",
  "customerName": "Demo Customer",
  "address": "12 Example Street",
  "email": "customer@example.test",
  "order": {
    "products": [
      {"name": "Notebook", "quantity": 2},
      {"name": "Pen", "quantity": 5}
    ]
  }
}
```

You can either enter an object in a JSON-style field or add individual fields through manual mapping, depending on the node configuration and n8n version.

### When should you use it?

Use Edit Fields when you need:

- One or a few predictable records.
- A quick way to test expressions and transformations.
- A known nested object to inspect how later nodes handle structured data.

For hundreds or thousands of varied records, a dedicated generator, an AI-created dataset, or a Code node is usually more suitable.

## 6. Method 4 — Generate custom mock data with the Code node

The **Code** node is helpful when your test data needs custom logic, repeated patterns, calculated values, or many generated records.

In n8n's Code node, output items are commonly returned as an array of objects, with each record wrapped inside a `json` property.

### Example: create 10 review records

```typescript
const reviews = [];

for (let i = 1; i <= 10; i++) {
  reviews.push({
    json: {
      reviewId: i,
      customerName: `Demo Customer ${i}`,
      rating: (i % 5) + 1,
      reviewText: `Sample review number ${i}`,
    },
  });
}

return reviews;
```

This creates 10 items. Each item has a `json` object containing the fields that downstream nodes can use.

### Example: create varied sign-up records

```typescript
const users = [];

for (let i = 1; i <= 20; i++) {
  users.push({
    json: {
      userId: i,
      firstName: `User${i}`,
      email: i % 4 === 0 ? "" : `user${i}@example.test`,
      country: i % 2 === 0 ? "India" : "Nepal",
    },
  });
}

return users;
```

This creates 20 records. Every fourth record has a blank email, and the country alternates. The data is deliberately synthetic and suitable for testing branches that handle missing emails.

**Important:** Use the Code node's supported execution mode and return shape for your installed n8n version. The examples above follow the common array-of-items pattern.

## 7. Method 5 — Use a small training data store

A training or demo data-store node may provide a small set of predefined records. It can be useful for exploring how nodes behave, but it is not necessarily a scalable source of test data.

A key distinction:

- A node may return a fixed number of records when run independently.
- If it receives many input items, it may execute once per input item and therefore produce a much larger total output.

Check the node's purpose and execution behavior before interpreting the output count. A small built-in sample store is generally less flexible than generating your own dataset.

## 8. Which method should you choose?

| Requirement | Suggested method |
|---|---|
| Many structured records with configurable field types | Mock-data generator |
| Dataset from a plain-language description | AI assistant |
| One or a few known records | Edit Fields (Set) node |
| Custom patterns, calculated values, or programmatic generation | Code node |
| Quick exploration of a node using built-in examples | Training/demo data store |

For most workflow testing, start with the first three main approaches: a dedicated data generator, AI-generated data, or a small manually defined record. Use the Code node when you need precise control over how the dataset is produced.

## 9. A practical testing checklist

Before using mock data in a workflow:

- [ ] Use fictional data rather than real personal information.
- [ ] Include normal, missing, invalid, and boundary-case values where relevant.
- [ ] Confirm field names and data types match the workflow.
- [ ] Validate JSON or CSV before importing it.
- [ ] Test each branch of If or Switch logic.
- [ ] Start with a small dataset, then increase the record count.
- [ ] Confirm downstream nodes are connected to test destinations, not live production systems.
- [ ] Check execution results and error messages.
- [ ] Keep a reproducible copy of the test dataset for debugging.

## 10. Troubleshooting

### Error: unexpected character after JSON

**Likely cause:** Multiple JSON objects were pasted without a surrounding array, or extra text was included.

**Try this:** Request a single valid JSON array, remove Markdown fences and commentary, and validate the result before pasting it.

### The node rejects the data structure

**Likely cause:** The node expects a different shape, such as n8n items wrapped with a `json` property rather than a plain array of records.

**Try this:** Check the editor's expected input/output structure and adapt the data accordingly.

### Missing-email branch never runs

**Likely cause:** All records have a non-empty email, or the condition does not match the actual field name/value.

**Try this:** Inspect sample records, ensure some `email` values are empty, and test the condition on a single record first.

### The workflow processes more items than expected

**Likely cause:** A node may execute once per incoming item, multiplying the total number of outputs.

**Try this:** Run the node independently with a small input and compare the input and output item counts.

### The dataset is too large to paste

**Try this:** Begin with 5–20 records, test the workflow, and then use a file, generator, or Code node for larger batches.

## 11. Practice questions

1. What is mock data, and why is it useful during workflow development?
2. Give two benefits of using fake records instead of live production records.
3. When is a dedicated mock-data generator more suitable than the Edit Fields node?
4. What is the difference between CSV and JSON for test data?
5. Why might n8n reject several JSON objects pasted one after another?
6. What does the `json` property do in the common n8n Code node output structure?
7. How could you test the branch that handles a missing email address?
8. Why should a workflow be tested with a small dataset before a large one?

### Answer key

1. Mock data is synthetic sample information used to test a workflow without relying on live records.
2. It can reduce repeated external calls and help prevent accidental changes to production data.
3. When many varied records or a larger dataset are needed.
4. CSV is tabular and spreadsheet-friendly; JSON supports structured and nested objects.
5. The input may not be valid JSON as a single document; an array may be required.
6. It wraps the record's data in the common n8n item format.
7. Include records with blank email values and verify the condition routes them to the intended branch.
8. A small run helps reveal data-shape or logic errors before they affect many items.

## Key takeaways

- Mock data makes workflow development faster, safer, and more predictable.
- A dedicated generator and an AI assistant can create larger datasets.
- Edit Fields is ideal for a small, controlled example.
- The Code node supports custom, repeatable dataset generation.
- JSON syntax and n8n's expected item structure both matter.
- Always test edge cases and keep mock data away from live production actions.
