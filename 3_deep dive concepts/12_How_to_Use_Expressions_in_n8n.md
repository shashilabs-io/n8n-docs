# Chapter 12: How to Use Expressions in n8n
## Transform data dynamically without writing a full Code node

Expressions let you read workflow data and perform small transformations directly inside node fields. They are a practical bridge between no-code workflows and low-code automation.

## Learning objectives

By the end of this chapter, you should be able to:

- Explain what an expression is and when to use one.
- Switch a field from fixed input to expression mode.
- Read fields using `$json` and access nested arrays and objects.
- Use built-in n8n functions and JavaScript string methods.
- Provide fallback values for missing data.
- Format dates and calculate date differences.
- Recognize when a Code node is more appropriate.

## 1. What are expressions?

Expressions are small pieces of logic used inside node fields. They can dynamically read input data, refer to other nodes, and transform values without requiring a separate Code node for every operation.

| Place | Best suited for |
|---|---|
| **Expression field** | A value or relatively small transformation inside a node field |
| **Code node** | More involved custom logic, multi-step calculations, or multi-line code |

Expressions use JavaScript syntax and can use n8n's built-in functions, workflow variables, and date/time helpers. The lesson also introduces JMESPath as a way of querying JSON data.

## 2. Switch a field to Expression mode

When configuring a field, n8n typically lets you choose between a fixed value and an expression.

- **Fixed:** the value you enter stays the same for each item.
- **Expression:** the value is calculated dynamically from available workflow data.

Expressions are commonly written inside double curly braces:

```text
{{ $json.customerName }}
```

Anything inside the braces is evaluated as an expression. Keep expression fields focused on returning the value needed by that field.

## 3. Read data from the current input item with `$json`

`$json` is a convenient way to access the JSON data of the current input item.

Suppose a node receives this item:

```json
{
  "orderNumber": "ORD-1001",
  "customerName": "John Doe",
  "email": "john@example.com",
  "order": {
    "products": [
      { "name": "V iPhone", "quantity": 1 },
      { "name": "Case", "quantity": 2 }
    ]
  }
}
```

### Read a top-level field

```text
{{ $json.customerName }}
```

Result: `John Doe`

### Use bracket notation

```text
{{ $json["email"] }}
```

Result: `john@example.com`

Dot notation is convenient for ordinary property names. Bracket notation is useful when a key needs to be expressed as a string or has characters that make dot notation unsuitable.

### Read a nested field in an array

The `order` field contains an object, and `products` contains an array. Array indexes start at zero.

```text
{{ $json.order.products[0].name }}
```

Result: `V iPhone`

You can use bracket notation for the same path:

```text
{{ $json["order"]["products"][0]["name"] }}
```

**Remember:** `[0]` means the first item, `[1]` means the second item, and so on.

## 4. Read data from another node

`$json` is used for the current input item. When you need output from a specifically named earlier node, reference that node by name. A common form is:

```text
{{ $("Customer Data").item.json.email }}
```

Replace `Customer Data` with the actual node name. The exact item-linking expression can depend on branches and item pairing, so use the expression editor's suggestions and preview to confirm the result.

## 5. Use built-in functions and JavaScript methods

Before adding a Code node, check whether an expression or built-in function can perform the transformation you need.

### Extract a domain from an email address

A built-in n8n string helper can extract the domain:

```text
{{ $json.email.extractDomain() }}
```

For `john@example.com`, the intended result is `example.com`.

Another approach uses JavaScript's `split()` method:

```text
{{ $json.email.split("@")[1] }}
```

`split("@")` divides the string at the `@` symbol and returns an array. Index `[1]` selects the part after the symbol.

The built-in helper is easier to read when it directly matches your use case. The `split()` approach illustrates how JavaScript string methods can be used inside expressions.

### Check the available helpers

n8n provides helpers for common operations, including:

- **Strings:** extract domains, emails, URLs, or URL paths.
- **Arrays:** operations such as chunking, compacting, differences, and averages where applicable.
- **Numbers:** rounding helpers such as ceiling and floor, formatting, and even/odd checks.
- **Booleans:** conversion helpers such as converting boolean values to integers.
- **Objects:** checks such as whether an object or field is empty or present.
- **Dates and times:** formatting, adding or subtracting time, and calculating differences.

Available methods depend on the data type and n8n version. Consult the [official n8n built-in methods and variables documentation](https://docs.n8n.io/code/builtin/overview/) when choosing a helper.

## 6. Provide a fallback when a value is empty

Data from external systems may be missing or empty. A fallback expression can prevent a downstream field from receiving an unusable blank value.

The lesson demonstrates n8n's `ifEmpty` helper:

```text
{{ $ifEmpty($json.email, "Email not found") }}
```

Conceptually:

- If the email is present, return the email.
- If the email is empty, return `Email not found`.

Test this with both a populated email and an empty or missing email. A fallback text value is not the same as `null`; choose the result that makes sense for the next step.

## 7. Work with dates and times

Expressions can use n8n's date/time helpers, including `$now` and `$today`, and Luxon-style date operations.

### Format the current date

For year-month-day:

```text
{{ $now.toFormat("yyyy-MM-dd") }}
```

For day-month-year:

```text
{{ $now.toFormat("dd-MM-yyyy") }}
```

Date format tokens matter: lowercase `dd` represents the day of the month, while uppercase `DD` has a different meaning in Luxon-style formatting. Verify the preview.

### Calculate days elapsed since the start of the year

The basic idea is to calculate the difference between the current date and a date object representing the beginning of the year.

```text
{{ $now.diff(DateTime.fromISO("2026-01-01"), "days").days }}
```

Here `$now` represents the current date and time, `DateTime.fromISO(...)` creates a date object, and `diff(..., "days").days` reads the difference in days. Replace the year with the relevant year if you use a fixed start date. The numeric result may include a fractional day because the current time is part of the calculation.

## 8. Expressions are not a substitute for every Code node

Expressions are ideal for returning a value from a field, accessing data, formatting strings, using helper methods, and performing compact calculations.

A Code node is a better fit when you need:

- Multiple statements or multi-line logic.
- A custom function with several steps.
- Complex loops or branching calculations.
- Processing that is difficult to express clearly in a single field.
- Custom transformations not covered by available helpers.

The lesson demonstrates a multi-line function inside an expression producing an invalid-syntax error. The key lesson is not to force a complex function into an expression field: move that logic to a Code node.

### A practical decision rule

```text
Need one dynamic field value?
          |
          v
Can a built-in helper or short expression do it?
       /       \
     Yes        No
      |          |
 Use an       Use a Code node
 expression   for custom logic
```

## 9. Troubleshooting expressions

### The expression returns `undefined` or an empty result

- Check that the field exists in the input item.
- Verify spelling and capitalization.
- Inspect the input JSON in execution data.
- Check whether the field is missing on some items.

### Nested data returns the wrong value

- Confirm the path through each object.
- Remember that array indexes start at zero.
- Check that the array contains the expected number of items.

### A string method fails

- Confirm that the input is a string rather than `null`, a number, or an object.
- Add an appropriate fallback or convert the value carefully.
- Preview the expression using realistic input data.

### An expression gives a syntax error

- Check matching parentheses, brackets, quotes, and curly braces.
- Keep the expression concise.
- Move multi-step custom logic to a Code node.

### A reference to another node does not work

- Verify the node name.
- Confirm that the node has executed and has output data.
- Check item linking and branches in the workflow.
- Use the expression editor's data preview rather than guessing the path.

## 10. Practice questions

1. What is the main purpose of an expression in n8n?
2. What is the difference between a fixed field value and an expression?
3. Write an expression to read the `customerName` field from the current item.
4. Given `order.products` as an array, how do you read the name of its first product?
5. What does `split("@")[1]` return when applied to a normal email address?
6. Why is a fallback value useful when the email field is empty?
7. Which date helper can be used to format the current date?
8. When should you choose a Code node instead of an expression?

## 11. Answer key

1. To dynamically access or transform data directly inside a node field.
2. A fixed value stays the same; an expression calculates a value from workflow data or other available variables.
3. `{{ $json.customerName }}`
4. `{{ $json.order.products[0].name }}`
5. The part after the `@` symbol, typically the email domain.
6. It gives downstream steps a meaningful value instead of an empty field, when appropriate.
7. `$now` can be formatted with a helper such as `.toFormat("yyyy-MM-dd")`.
8. When the logic needs multiple statements, a custom function, complex processing, or is not clear as a short expression.

## Key takeaways

- Expressions bring dynamic, low-code data transformation into node fields.
- Use `$json` for the current input item and a named-node reference when you need data from a specific earlier node.
- Use array indexes carefully; the first item is index zero.
- Prefer a built-in helper when it clearly solves the problem.
- Test empty and unusual values, not only the happy path.
- Use a Code node for complex or multi-line custom logic.
