# Chapter 19: Creating Sub-workflows in n8n

## Learning objectives

By the end of this chapter, you should be able to:

- Explain what a sub-workflow is.
- Identify workflow logic that can be reused.
- Describe how a parent workflow passes input to a sub-workflow and receives output.
- Understand modular workflow design through a support-ticket example.
- Recognize the role of the Execute Sub-workflow / Execute Workflow node.

## 1. What is a sub-workflow?

A **sub-workflow** is a separate n8n workflow that can be called from another workflow. A parent workflow can use an **Execute Sub-workflow** or similarly named workflow-execution node to call it, pass input data, and receive output data.

Think of a sub-workflow as a reusable component. Instead of copying the same sequence of nodes into several workflows, build that logic once and call it wherever it is needed.

### Why use sub-workflows?

- **Reusability:** Write common logic once and use it in multiple workflows.
- **Maintainability:** Update shared logic in one place instead of editing several duplicated copies.
- **Scalability:** Keep larger automations organized into smaller components.
- **Readability:** Make the purpose and structure of a workflow easier to understand later.
- **Separation of concerns:** Keep one workflow focused on a specific task.

A sub-workflow can make a project easier to manage, but it does not automatically make every workflow execute faster. Actual performance depends on the work being performed, the number of executions, and how the workflows are configured.

## 2. Parent workflows and sub-workflows

A typical arrangement looks like this:

```text
Parent workflow
    |
    | Execute workflow + pass input
    v
Reusable sub-workflow
    |
    | Return output
    v
Parent workflow continues
```

The parent workflow handles the larger process. The sub-workflow handles a well-defined part of that process.

### What logic should become a sub-workflow?

Look for a group of nodes that:

- Performs one clear task.
- Is repeated in more than one workflow.
- Has inputs that can be described clearly.
- Produces an output that the calling workflow can use.
- Can be tested independently.

For example, a team might reuse customer lookup, message formatting, ticket logging, or data validation logic across multiple automations.

## 3. Example: support-ticket handling

Consider a workflow that receives a support ticket through a webhook. The incoming data includes a customer's email address and the ticket message.

The workflow needs to:

1. Find the customer record using the email address.
2. Determine whether the customer is VIP.
3. Route VIP tickets to a senior-support process.
4. Route non-VIP tickets according to region.
5. Log ticket details in a support-ticket sheet.

A customer can be classified as VIP when **either** of these conditions is true:

- The customer's order history is greater than the configured threshold.
- The customer's average rating is greater than the configured threshold.

The use of **OR** is important: either condition is enough to qualify. The exact thresholds should be set to match the business rule being implemented.

### Example data fields

A customer record may include:

- `customerId`
- `customerName`
- `email`
- `orderHistory`
- `averageRating`
- `region`

A support-ticket record may include:

- `ticketMessage`
- `customerEmail`

The customer lookup returns CRM-like details. Those details then need to be combined with the ticket information so downstream nodes can use both.

## 4. Build the parent workflow

The following is a conceptual outline of the support-ticket workflow.

### Step 1 — Receive a ticket

Use a **Webhook** trigger to receive data such as:

```json
{
  "email": "customer@example.test",
  "ticketMessage": "My product arrived damaged. I need a replacement."
}
```

For safe development, use fictional sample data or edited/pinned node output before connecting live support systems.

### Step 2 — Look up the customer

Use a data source such as Google Sheets to find the customer row matching the email received by the webhook.

In a real implementation, select the appropriate customer data source, credentials, document, sheet, and lookup/filter settings.

### Step 3 — Combine ticket and customer data

Use a **Merge** node when the downstream logic needs fields from both the webhook and the customer lookup.

Check that the merge configuration combines the intended items. Also check field names and capitalization: `email` and `Email` are not necessarily treated as the same key.

### Step 4 — Determine VIP status

Use an **If** node to check the order-history and average-rating conditions. If either condition qualifies the customer, route the record down the VIP path.

```text
Ticket + customer record
          |
       VIP check
       /       \
    VIP        Non-VIP
     |            |
Senior-support  Region check
notification      /    \
                 EU    Other
                 |       |
              EU team  US team
```

The thresholds shown in any example should be treated as demonstration values, not universal business rules.

### Step 5 — Notify the appropriate team

For VIP tickets, a Slack notification can alert a senior-support channel. The message can include the customer's name and ticket message, using expressions to insert values from the incoming item.

For non-VIP tickets, check the customer's region. In the example logic:

- Customers in the EU are assigned to the EU team.
- Customers in other regions are assigned to the US team.

This is only an example routing rule. Adapt it to the actual support team's coverage.

### Step 6 — Log the ticket

Append the ticket details to a support-ticket sheet. Example columns include:

- `ticketId`
- `customerEmail`
- `issueDescription`
- `team`
- `priority`
- `status`

A hash of the ticket message can be used as a convenient demonstration identifier, but it is not guaranteed to be unique. For production ticket IDs, prefer a reliable ID-generation strategy and handle duplicate submissions explicitly.

## 5. Identify reusable logic

Once the first workflow works, review it and other workflows in the organization. Look for a repeated section that has a clear purpose and can be called with well-defined inputs.

For example, if multiple workflows need to format and log a support ticket, that common sequence may be a good candidate for a sub-workflow.

Before extracting logic, answer these questions:

1. What inputs does this section require?
2. What output should it return?
3. Does it depend on data or credentials that exist only in the parent workflow?
4. What should happen if a required field is missing?
5. How can it be tested independently?

Avoid extracting a section merely because it contains several nodes. Extract it when doing so makes the workflow more reusable, clearer, or easier to maintain.

## 6. Calling a sub-workflow

A common setup involves two parts.

### In the sub-workflow

1. Create a separate workflow for the reusable task.
2. Configure the appropriate sub-workflow trigger, often named **When Executed by Another Workflow** or similarly in the current n8n version.
3. Define or document the expected input fields.
4. Build the reusable logic.
5. Ensure the final node outputs the data the caller needs.

### In the parent workflow

1. Add the workflow-execution node (often called **Execute Sub-workflow** or **Execute Workflow**).
2. Select the target workflow.
3. Configure how input data is passed to it.
4. Run a test with representative sample data.
5. Inspect the sub-workflow result and confirm the parent workflow can use it.

Exact labels and options can differ by n8n version. Check the node descriptions in your instance.

## 7. Good design practices

- Give each sub-workflow one clear responsibility.
- Use meaningful workflow names.
- Document the expected input and output fields.
- Keep inputs and outputs consistent across callers.
- Test the sub-workflow with normal and edge-case data.
- Handle errors deliberately, especially when the parent workflow depends on the result.
- Avoid hard-coding assumptions that differ between calling workflows.
- Use test credentials and destinations during development.
- Review access and permissions for workflows that handle customer information.

## 8. Common mistakes and troubleshooting

### The sub-workflow receives no useful input

**Possible cause:** The caller's input is not mapped or the sub-workflow expects different field names.

**Check:** Inspect the incoming data, confirm field names, and test the sub-workflow directly with representative input.

### The output is missing fields

**Possible cause:** A node inside the sub-workflow returns only part of the data or changes the item structure.

**Check:** Inspect the output of each important node and make sure the final output includes what the parent needs.

### The workflow routes records incorrectly

**Possible cause:** An If condition uses the wrong comparison, field name, or AND/OR combination.

**Check:** Test one record that should pass and another that should fail. Confirm that OR is used when either of two conditions is sufficient.

### The same logic still appears in several workflows

**Possible cause:** The common section was copied rather than replaced with calls to the shared sub-workflow.

**Check:** Identify the common responsibility, create the reusable workflow, and update each caller to use it.

### A ticket ID generated from a message is duplicated

**Possible cause:** Two tickets contain identical message text, or the identifier strategy is not designed to guarantee uniqueness.

**Check:** Use a dedicated ID strategy appropriate for the production system and consider duplicate-submission handling.

## 9. Practice questions

1. What is a sub-workflow in n8n?
2. Name three benefits of using sub-workflows.
3. Which kind of node is commonly used to call a workflow from another workflow?
4. Why should input and output fields be documented?
5. In the support-ticket example, why is a Merge node useful?
6. When should an OR condition be used for VIP classification?
7. Why is a hash of the ticket message not a guaranteed unique ticket ID?
8. Give one example of logic that could be reused in multiple workflows.

### Answer key

1. A separate workflow that another workflow can call and exchange data with.
2. Reusability, maintainability, readability, modularity, and separation of concerns.
3. An Execute Sub-workflow / Execute Workflow node, depending on the version and interface.
4. So callers pass the right data and can reliably use the returned result.
5. It combines the ticket information with the customer record for downstream processing.
6. When either condition being true should be enough to qualify the customer.
7. Identical messages can produce the same hash, and a hash is not a guaranteed unique-ID service.
8. Examples include customer lookup, ticket logging, data validation, or message formatting.

## Key takeaways

- A sub-workflow is a reusable workflow called by another workflow.
- It can reduce duplicated logic and make larger automations easier to maintain.
- Use clear input and output contracts.
- Extract logic that has a distinct purpose and can be reused.
- Test routing, data merging, and error cases before using live systems.
- Do not assume modularization automatically improves execution speed; measure performance where it matters.
