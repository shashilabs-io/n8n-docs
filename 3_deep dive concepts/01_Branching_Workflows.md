# How Branching Works in n8n Workflows
## Smart Automations with Multiple Paths

![n8n workflow showing Google Sheets, Switch, AWS SES, and Slack nodes](n8n_workflow_optimized.png)

### What you will learn
By the end of this lesson, you should be able to:
- Explain why branching is useful in a real business workflow.
- Fetch order records from Google Sheets in n8n.
- Understand why n8n represents data as a list of items.
- Configure a **Switch** node to route items by order status.
- Use expressions such as `{{ $json.orderStatus }}` to access the current item.
- Send emails and Slack messages from different branches.
- Distinguish between **routing items to different paths** and **sending the same items down multiple paths**.

---

## 1. The business problem: processing orders manually

The lesson begins with a business scenario. A person manually reviews orders that have not yet been shipped or delivered. Each order can be in a different state, and the team must take the appropriate action.

| Order status | Business need | Automated action in the lesson |
|---|---|---|
| Pending | Customer-service staff need to investigate missing information and help release the order for processing. | Send an email to the service team. |
| Processing | Some high-priority orders have been waiting for a few days and should be expedited. | Send a Slack message to the operations team. |
| Canceled | Finance needs an official record and a notification. | Send an email and a Slack message to finance. |
| Refunded | Finance needs an official record and a notification. | Send an email and a Slack message to finance. |

When this work is done manually, a person has to inspect each order, decide which team should be contacted, send the message, and keep track of what has already been done. This can take hours and can introduce mistakes.

**Goal:** automate the process in n8n without writing a custom program for the workflow.

> The lesson uses sample order data. The team names, channels, email destinations, and credentials in a real workflow must be replaced with your own configured resources.

>Link :  https://drive.google.com/drive/folders/1FgKXJklEj_nsZB0JAtVguaXtXK8CEQGk 
>  - Access all the mcok data from this folder. For this lesson we will bw using Mock_Orders_Data

## 2. Start a workflow and choose a trigger

Create a new workflow in the n8n canvas.

Start with a **Manual Trigger** so the workflow can be tested on demand. For a recurring business process, a **Schedule Trigger** is a better production candidate. For example, it could run every day at 9:00 a.m., 10:00 a.m., or another time appropriate for the business.

- **Manual Trigger:** useful while building and testing.
- **Schedule Trigger:** useful when the workflow should run automatically on a schedule.

The lesson continues with Manual Trigger for testing.

## 3. Fetch orders from Google Sheets

The sample Google Sheet is called **Mock Order Data**. It contains 100 data records, excluding the header, with fields such as:

- Order ID
- First name
- Last name
- Customer email
- Order status
- Order date

Add a **Google Sheets** node and configure it to fetch rows.

### Configuration sequence shown in the lesson

1. Select the Google Sheets credential already connected to n8n.
2. Choose the resource for a sheet within a document.
3. Set the operation to **Get Row(s)**.
4. Choose the document from the connected account's list.
5. Select the **Mock Order Data** document and its sheet.
6. Leave filters unset to fetch all 100 sample records.
7. Click **Test step**.

The output should contain the fetched records. n8n lets you inspect the result in **JSON** view or **Table** view. Table view is convenient for row-and-column data; JSON view shows the underlying objects and their properties.

### A key idea: n8n processes lists of items

In the lesson, the Google Sheets node returns 100 items. In n8n, data passed between nodes is represented as a list of items—even when only one item is present.

Conceptually, the data looks like this:

```json
[
  {
    "orderId": "ORD001",
    "firstName": "Asha",
    "lastName": "Kumar",
    "customerEmail": "customer@example.com",
    "orderStatus": "Pending",
    "orderDate": "2026-10-01"
  },
  {
    "orderId": "ORD002",
    "firstName": "Ravi",
    "lastName": "Singh",
    "customerEmail": "another@example.com",
    "orderStatus": "Processing",
    "orderDate": "2026-10-02"
  }
]
```

Each item represents an object. The workflow can evaluate each order separately, using the order-status property to decide what should happen next.

### Accessing a property with an expression

The lesson introduces `{{ $json }}` as a shorthand for the current item being processed. To access one of its properties, use dot notation:

```text
{{ $json.orderStatus }}
```

This expression means: “Get the `orderStatus` property from the current item.”

Use `$input.item` to refer to the current input item. The important practical point is that the expression is evaluated in the context of the item currently being processed.

> Use the exact property name shown in your node's output. If your Google Sheets column is named `Order Status`, the expression may need to match that actual field name rather than `orderStatus`.

## 4. Why use a Switch node?

An **If** node is suitable when a condition has two outcomes: true or false. The lesson needs to handle four order statuses—pending, processing, canceled, and refunded.

You could chain multiple If nodes, but the workflow would become harder to follow. A **Switch** node allows you to define several routing rules in one place and create a separate output for each matching status.

Add a **Switch** node after Google Sheets.

### Configure the routing rules

For each rule, compare the current item's order-status value with a text value.

| Rule | Value to compare against | Suggested output name |
|---|---|---|
| 1 | `pending` | Pending Orders |
| 2 | `processing` | Processing Orders |
| 3 | `canceled` | Canceled Orders |
| 4 | `refunded` | Refunded Orders |

In the expression field, select or enter the order-status property. The lesson uses the comparison “value 1 is equal to value 2,” with the status text entered as the comparison value.

An expression for the first value could look like:

```text
{{ $json.orderStatus }}
```

Then compare it with `pending`. Repeat the rule for the other statuses.

### Rename outputs clearly

 Try to rename each output. If the outputs remain named “Output 0,” “Output 1,” and so on, it becomes difficult to tell which status each branch represents.

Use names that communicate the rule, such as **Pending Orders** and **Refunded Orders**. Clear names make the canvas easier to understand and maintain.

### Test and interpret the counts

In the demonstration, the Switch node routes:
- 13 items to pending,
- 22 items to processing,
- 9 items to canceled,
- 20 items to refunded.

These counts total 64, not 100. The remaining records have other statuses, such as **shipped** or **delivered**, which are not covered by the four rules.

Your own counts will depend on the data in your sheet.

### Fallback output and Ignore Case

The Switch node provides additional options:

- **Fallback output:** determines what happens to an item that matches none of the configured rules. The lesson leaves it unconfigured so unmatched items are ignored for this exercise. In a real workflow, you may instead route unmatched records to a review path or log them.
- **Ignore Case:** useful when a status may be written with different capitalization, such as `refunded`, `Refunded`, or `REFUNDED`.

Test the node after configuring these options and confirm that the expected items reach each output.

## 5. Branch 1 — Email the service team about pending orders

For the **Pending Orders** output,We will be using Amazon SES email node that is already configured. A standard Send Email node with SMTP configuration is another possible approach.

The email setup includes:
- An email-sending operation.
- A subject that identifies the order, including its order ID.
- A simple text body explaining that the order is pending.
- A sender address, such as a no-reply address.
- A recipient address for the service team—ideally a distribution list shared by the team.

We recommend a group mailbox rather than sending the notification to only one individual, so the appropriate team members can see it.

After configuring the node, click **Test step**. The Output shows message IDs and checks the inbox to confirm delivery.

**Result:** pending orders can be sent to the service team automatically rather than being identified and emailed manually.

## 6. Branch 2 — Send a Slack message for processing orders

For the **Processing Orders** output, the lesson adds a **Slack** node.

Before this node can send messages, your Slack workspace must be connected to n8n and the required credentials configured.

The demonstrated configuration is:

1. Choose the Slack credential.
2. Select the **Message** resource.
3. Select the **Send** operation.
4. Choose a channel rather than an individual user, so the operations team can monitor it together.
5. Select the channel for high-priority processing orders.
6. Enter a simple text message that identifies the order and asks the team to review or expedite it.
7. Test the node.

The demonstration routes 22 processing items through the Slack node. Each item is processed, and the node sends the configured message for that item.

The lesson mentions that Slack messages can also use blocks or attachments, but it uses a simple text message here.

**Result:** the operations team receives Slack notifications for processing orders that need attention.

## 7. Branches 3 and 4 — Notify finance about canceled and refunded orders

Canceled and refunded orders require **two actions**:
1. Send an email record to finance.
2. Send a Slack notification to the finance channel.

The lesson reuses the email and Slack nodes by copying them and connecting the **Canceled Orders** and **Refunded Orders** outputs to both action nodes.

This creates two outgoing connections from each of these Switch outputs:

- Canceled Orders → Email
- Canceled Orders → Slack
- Refunded Orders → Email
- Refunded Orders → Slack

Configure the email subject and body so finance can identify the canceled/refunded order. Configure the Slack node to post in the channel used for canceled and refunded orders. Then test both nodes.

In the demonstration:
- 9 canceled items are sent through the actions.
- 20 refunded items are sent through the actions.
- The email node processes both groups, for 29 items in total.
- The Slack node likewise receives items from both inputs.

The key point is that a canceled or refunded item should reach **both** the email action and the Slack action.


## 8. The two types of branching explained

This is the main concept of the lesson.

### Type A: Route different items down different paths

The Switch node examines each order's status and sends it to the matching output.

```text
Google Sheets
      |
    Switch
   /  |  |  \
Pending Processing Canceled Refunded
   |       |       |        |
 Email    Slack   Finance   Finance
```

- Pending items go to the pending output.
- Processing items go to the processing output.
- Canceled items go to the canceled output.
- Refunded items go to the refunded output.

The purpose is to **separate items according to their values** so different groups receive different actions.

### Type B: Send the same items to multiple actions

For canceled and refunded orders, each item must trigger both an email and a Slack message.

```text
Canceled Orders ──┬──> Email Finance
                  └──> Slack Finance

Refunded Orders ──┬──> Email Finance
                  └──> Slack Finance
```

Here, an item is not choosing only one action. It is connected to multiple paths so that both actions can be performed.

**Remember the distinction:**
- **Routing:** Which path should this item take?
- **Multiple actions:** Which actions should all happen for this item?

This distinction helps you design more complex n8n workflows without creating a long chain of nested conditions.

## 9. Workflow recap

The workflow built in the lesson follows this sequence:

1. Manual Trigger starts a test run.
2. Google Sheets fetches the sample order records.
3. Switch evaluates each item's order status.
4. Pending → email the service team.
5. Processing → send a Slack message to operations.
6. Canceled → email finance and post to Slack.
7. Refunded → email finance and post to Slack.
8. Unmatched statuses are ignored in this demonstration.

For daily production use, the Manual Trigger could be replaced by a Schedule Trigger. Before enabling the workflow, verify the source sheet, status spelling, credentials, recipients, Slack channels, and how unmatched statuses should be handled.

## 10. Common mistakes and checks

| Problem | What to check |
|---|---|
| Some orders do not appear in any of the four branches | They may have another status, such as shipped or delivered, or their text may not match the rule. |
| A rule does not match due to capitalization | Enable **Ignore Case** or normalize the status values. |
| The output labels are confusing | Rename each Switch output to a meaningful status name. |
| Email actions fail | Verify the email credentials, sender, recipient, and sending-service configuration. |
| Slack messages fail | Verify the Slack credential, channel selection, and permissions. |
| Only one action happens for canceled/refunded items | Check that both the email and Slack nodes are connected to both relevant Switch outputs. |
| The total branch count is less than the input count | Some items may not match any configured rule; inspect unmatched statuses. |

## 11. Practice exercise (homework from the lesson)

The lesson's homework is to improve the combined canceled/refunded message.

**Task:** make the notification say the correct status for each order:
- For a canceled order: say that the order was canceled.
- For a refunded order: say that the order was refunded.

Do not use the generic phrase “canceled/refunded” for both.

Suggested approach:
1. Inspect the data available in the incoming item.
2. Use the status property to make the message specific.
3. Test with at least one canceled order and one refunded order.
4. Verify the email and Slack outputs both contain the correct status.

### Self-check questions

1. Why is a Switch node a better fit than a single If node for the four statuses in this lesson?
2. Why does n8n represent even one record as a list of items?
3. What does `{{ $json.orderStatus }}` access?
4. Why should Switch outputs be renamed?
5. What is the purpose of a fallback output?
6. What does Ignore Case help prevent?
7. Which action is configured for pending orders?
8. Which action is configured for processing orders?
9. Why do canceled and refunded orders connect to two action nodes?
10. What is the difference between routing items and sending items down multiple paths?

### Answer key

1. Switch can define several routing rules and outputs in one node.
2. n8n uses a consistent list-of-items structure between nodes.
3. It reads the `orderStatus` property from the current item.
4. Clear names make the workflow easier to read and maintain.
5. It defines how to handle items that match none of the rules.
6. It helps comparisons work when status capitalization differs.
7. An email to the service team.
8. A Slack message to the operations team.
9. Finance needs both an email record and a Slack notification.
10. Routing chooses a matching path for an item; multiple-action branching sends the item to more than one action.

---

## Key takeaway

A **Switch** node makes multi-condition routing readable, while multiple outgoing connections let one group of items trigger more than one action. Together, these patterns let you automate practical business processes—such as order monitoring, team notifications, and finance records—using n8n's visual workflow canvas.
