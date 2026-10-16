# How to Use the Set Node in n8n

### What you will learn
By the end of this lesson, you should be able to:
- Explain why workflows often need to reshape data before passing it to another node.
- Use the Set node (called **Edit Fields** in newer n8n versions) to add, change, or remove fields.
- Choose between manual field mapping and JSON-style input where available.
- Understand the difference between keeping other fields and keeping only the fields you define.
- Build a practical workflow that prepares clean data for an email or API request.

---

## 1. Start with a business problem

Imagine a user signs up for your learning platform. Your signup form produces data like this:

```json
{
  "first_name": "Riya",
  "last_name": "Verma",
  "email_address": "riya@example.com",
  "phone_number": "9876543210",
  "internal_form_id": "FORM-8821",
  "submitted_at": "2026-10-12T09:15:00Z"
}
```

The data is valid, but the next node may not need every field. Perhaps you want to send a welcome email and store a clean customer record with:
- `fullName`
- `email`
- `source`
- `signupDate`

You could pass the original data through unchanged, but it would be harder to work with downstream if fields have inconsistent names or contain information you don't need. This is where the **Set** node—called **Edit Fields** in newer n8n versions—helps.

Its job is to shape the data: create fields, update values, rename fields by mapping them to new names, and optionally remove fields you don't want to pass forward.

## 2. Where the Set / Edit Fields node fits

A simple workflow could be:

```text
Form Trigger
    |
    v
Set / Edit Fields
    |
    v
Send Welcome Email
```

The trigger receives the original signup data. The Set/Edit Fields node transforms it into the structure you want. The email node then uses the cleaned-up values.

The Set node does not send the email itself. It prepares the data for a later node.

## 3. Add the node and choose how to edit fields

1. Open your n8n workflow.
2. Add the **Edit Fields (Set)** node. In older versions it may appear as **Set**; in newer versions it is commonly labeled **Edit Fields**.
3. Connect it after the node that provides the input data.
4. Open its configuration and choose the available editing method.

In the manual field-mapping interface, you typically add fields one at a time and specify a name, type, and value. Depending on your n8n version, the node may also offer a JSON-style mode for defining multiple fields together.

The exact labels and layout can differ between versions, but the goal is the same: define the output data you want.

## 4. Add a new field: create `fullName`

Suppose the input contains separate `first_name` and `last_name` fields, but your email template needs one `fullName` field.

Add a field:
- **Name:** `fullName`
- **Type:** String
- **Value:** an expression that joins the two input values.

An n8n expression can look like this:

```text
{{ $json.first_name + " " + $json.last_name }}
```

For the sample input, the result is:

```json
{
  "fullName": "Riya Verma"
}
```

This expression reads the two properties from the current item and joins them with a space. Expressions are evaluated for each item that passes through the node.

If the source fields have different names in your workflow, use the exact names shown in the incoming data.

## 5. Rename or remap fields

A common data-cleaning task is converting field names from one naming convention to another.

For example:

| Original field | New field |
|---|---|
| `email_address` | `email` |
| `submitted_at` | `signupDate` |
| `phone_number` | `phone` |

In Edit Fields, add a field with the desired output name and set its value from the original field. For example:

```text
{{ $json.email_address }}
```

for the new `email` field.

This is a mapping operation: you are not changing the source form's column name; you are creating an output field with the name your later nodes expect.

## 6. Keep or remove the other input fields

One of the most important settings is whether to keep fields that you have not explicitly defined.

### Option A — Keep other input fields

If you enable the option to include or keep other input fields, the output can contain the original fields plus the new or mapped fields.

This is useful when:
- Later nodes may need other original values.
- You want to add a few derived fields without losing the rest of the record.
- You are still exploring the input data.

### Option B — Keep only the fields you define

If you disable the option to keep other input fields, the output is limited to the fields you define in the node.

For our signup example, the desired output could be:

```json
{
  "fullName": "Riya Verma",
  "email": "riya@example.com",
  "phone": "9876543210",
  "source": "website",
  "signupDate": "2026-10-12T09:15:00Z"
}
```

The fields `internal_form_id` and the original unrenamed fields are not passed forward unless you add them explicitly.

This is useful when you want a predictable, clean output structure. However, be careful: if a later node needs a field that you removed, that field will no longer be available from this node's output.

> Practical habit: after changing this setting, execute the node and inspect its output. Do not assume a field is still present—verify it.

## 7. Build the example step by step

### Step 1 — Inspect the incoming data

Run the trigger or previous node. Confirm that the expected fields exist, including `first_name`, `last_name`, and `email_address`.

### Step 2 — Add the Edit Fields node

Connect it directly after the trigger.

### Step 3 — Define the output fields

Create these fields:

| Output field | Type | Value |
|---|---|---|
| `fullName` | String | `{{ $json.first_name + " " + $json.last_name }}` |
| `email` | String | `{{ $json.email_address }}` |
| `phone` | String | `{{ $json.phone_number }}` |
| `source` | String | `website` |
| `signupDate` | String | `{{ $json.submitted_at }}` |

For `source`, use a fixed text value. For the other fields, use expressions that read the incoming item.

### Step 4 — Decide which original fields to keep

For a clean output object, turn off the option to keep other input fields and define everything needed by later nodes. If you want to preserve additional data, enable the setting and inspect the resulting structure.

### Step 5 — Test the node

Execute the Edit Fields node and inspect the output. It should contain the values you defined. Confirm that the full name is joined correctly and that the email and signup date are not missing.

### Step 6 — Connect the email node

Connect a Send Email node after Edit Fields. Use the output `email` as the recipient and `fullName` in the email body.

Example email text:

```text
Hi {{ $json.fullName }},

Welcome to our learning platform! We're glad you're here.
```

The expression syntax shown here is illustrative of n8n expressions. Configure your actual email node and credentials separately.

## 8. JSON-style editing

Some versions and configurations of the Edit Fields node provide a JSON-style editing mode. Instead of adding each field through the UI, you can define the desired object together.

For the clean signup output, the intended structure is conceptually:

```json
{
  "fullName": "{{ $json.first_name + ' ' + $json.last_name }}",
  "email": "{{ $json.email_address }}",
  "phone": "{{ $json.phone_number }}",
  "source": "website",
  "signupDate": "{{ $json.submitted_at }}"
}
```

This snippet illustrates the intended mapping, not necessarily a paste-ready JSON configuration for every n8n version. Some interfaces require expressions to be entered using their expression controls rather than as quoted template strings in raw JSON. Follow the mode's UI instructions and test the output.

Use manual mapping if you prefer to configure fields one at a time. Use JSON-style editing when you are comfortable describing the complete output structure together.

## 9. Common mistakes and troubleshooting

| Problem | What to check |
|---|---|
| `fullName` is empty or incomplete | Verify the spelling of `first_name` and `last_name` in the input data. |
| Expressions appear as literal text | Make sure the field is in expression mode where required. |
| Original fields disappear | Check whether the option to keep/include other input fields is disabled. |
| A later node cannot find `email_address` | You may have mapped it to `email` and removed the original field. Update the later node to use the new name. |
| A field has the wrong type | Check the field type selected in Edit Fields and the data returned by the source. |
| A tutorial's node name or setting differs | Your n8n version may use the newer **Edit Fields** label or a different UI layout. |
| A downstream email is sent to the wrong address | Inspect the output and ensure the recipient expression uses the intended field. |

## 10. When should you use the Set / Edit Fields node?

Use it when you need to:
- Create a derived field such as `fullName`.
- Rename or map fields into a consistent naming convention.
- Add a constant value such as `source: "website"`.
- Keep only the fields needed by the next node.
- Prepare data for an email, database insert, API request, or another workflow step.

It is not a replacement for every transformation. For complicated calculations, repeated data structures, or more advanced logic, you may need other nodes or expressions. Start with Edit Fields when the main job is simply to shape the data.

## 11. Practice exercises

### Exercise A — Create a display name

Input:

```json
{
  "firstName": "Kabir",
  "lastName": "Das",
  "email": "kabir@example.com"
}
```

Create a `displayName` field containing `Kabir Das`.

### Exercise B — Prepare a clean order record

Input:

```json
{
  "order_id": "ORD-205",
  "customer_email": "meera@example.com",
  "amount": 1800,
  "debug_token": "TEMP-XYZ"
}
```

Use Edit Fields to output only:
- `orderId`
- `email`
- `amount`

Do not pass `debug_token` to the next node.

### Exercise C — Keep or remove original fields?

You add `fullName` to a record and need the original `email_address` field for a later node. What should you check in the setting for keeping other input fields, and why?

### Answer key

**Exercise A**

Use a String field named `displayName` with the expression:

```text
{{ $json.firstName + " " + $json.lastName }}
```

**Exercise B**

Map `order_id` to `orderId`, `customer_email` to `email`, and preserve `amount`. Turn off the option to keep other input fields if you want only these three output fields.

**Exercise C**

If the original fields are not explicitly defined, ensure the option to keep/include other input fields is enabled—or create an output field for the original email. Otherwise, the original field may be absent from the output.

---

## Key takeaway

The Set node—now commonly called **Edit Fields**—is a data-shaping node. It lets you add fields, map existing values into new names, define constant values, and control whether the original fields remain in the output. Always inspect the node's output before connecting it to later steps.
