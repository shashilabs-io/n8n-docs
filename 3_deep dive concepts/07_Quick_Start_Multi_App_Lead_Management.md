# n8n Quick Start Workflow: Connect Google Forms, Google Sheets, Slack, Notion & Airtable

## A practical guide to multi-app lead management


## 1. What you will learn

By the end of this lesson, you should be able to:

- Explain how n8n acts as a bridge between separate applications.
- Receive form-submission data through a Webhook.
- Append a new lead to Google Sheets.
- Use an **If** node to decide whether a lead meets simple qualification rules.
- Send a Slack alert for qualified leads.
- Create a task page in a Notion database.
- Create a customer record in Airtable.
- Understand the credentials and permissions needed to connect Notion and Airtable.
- Diagnose common configuration errors shown in the lesson.

## 2. The business problem

Imagine a small business collects potential-customer details using several Google Forms. An employee previously had to:

1. Open each form and review its responses.
2. Copy new leads into one central spreadsheet.
3. Decide whether each lead was qualified.
4. Notify the sales team.
5. Create a follow-up task in Notion.
6. Add the customer to a CRM.

This repeated manual work is slow and can lead to missed leads or inconsistent records. n8n can connect the applications and automate these steps.

## 3. The big-picture workflow

```text
Google Form submission
        |
        v
Webhook (receive form data)
        |
        v
Google Sheets (append lead row)
        |
        v
If (evaluate qualification rules)
        |
        +---- TRUE / qualified ----+
        |                           |
        v                           v
Slack (notify sales)       Notion (create follow-up task)
        |                           |
        +-------------+-------------+
                      |
                      v
             Airtable (create CRM record)
```

**Important:** The transcript builds the Slack, Notion, and Airtable actions along the qualified branch. The exact layout and execution behavior depend on how you connect the nodes in your own canvas. Verify the connections before activating the workflow.

## 4. Why n8n is useful

Applications do not always share data directly in the way a business needs. n8n provides a workflow layer that can:

- **Start** when an event occurs, such as a form submission.
- **Transform or route** the data.
- **Perform actions** in other apps, such as creating records or sending messages.
- Use an **HTTP Request** node when an application does not have a suitable native n8n node.

Think of n8n as a bridge that moves data and coordinates actions across apps.

## 5. Step 1 — Receive Google Forms data with a Webhook

### What the Webhook does

A Webhook gives another application an endpoint to send data to. In this lesson, Google Forms is connected through Google Apps Script, which sends a POST request to the n8n Webhook.

### Configuration outline

1. Create a workflow in n8n.
2. Add a **Webhook** trigger node.
3. Set the HTTP method to **POST**, because the form integration sends data to n8n.
4. Use the **Test URL** while developing.
5. Put that URL into the Apps Script integration used by the form.
6. In n8n, click the control that listens for a test event.
7. Submit a new test response and confirm that the Webhook receives it.

The transcript's form fields are first name, last name, email, company name, and role.

### Test URL vs. Production URL

- **Test URL:** Used while testing. The workflow must be listening for the test event.
- **Production URL:** Used when the workflow is active and ready to receive real events.

Use the URL that belongs to the current Webhook node. A URL copied from an older workflow may be incorrect.

### Multiple forms

The lesson describes connecting multiple forms to a common workflow. Each form must be configured to send data to the intended Webhook endpoint, and the incoming fields must be compatible with the workflow's expected structure.

## 6. Step 2 — Append each lead to Google Sheets

The spreadsheet acts as a central list of incoming leads.

### Configure the node

1. Add a Google Sheets node after the Webhook.
2. Select the spreadsheet and worksheet.
3. Choose an operation that **appends a row**.
4. Map incoming fields to spreadsheet columns, for example:

| Spreadsheet column | Incoming value |
|---|---|
| First Name | First name from the form |
| Last Name | Last name from the form |
| Email | Email from the form |
| Company Name | Company name from the form |
| Role | Role from the form |

The order of the columns does not have to match the order of the incoming data if you map the fields explicitly.

### A configuration error from the lesson

The instructor initially selected **Append or Update Row**. That operation needs a matching column/value so it can identify the row to update. Since the goal was simply to add a new lead, the operation was changed to **Append Row**.

**Rule of thumb:** Use *Append Row* when you want a new row. Use an update/upsert-style operation only when you intentionally need to find and update an existing row.

## 7. Step 3 — Qualify the lead with an If node

The lesson uses simple conditional logic rather than AI.

A lead is considered qualified in the example when the company name is present and the email does not end in certain personal-email domains. These are demonstration rules, not universal sales criteria.

Example conditions described in the transcript:

- Company name **is not empty**.
- Email **does not end with** `gmail.com`.
- Email **does not end with** `hotmail.com`.

Depending on the node configuration, multiple conditions may be combined with AND/OR logic. Check the condition-combination setting: if all conditions must pass, the lead must satisfy every configured rule.

### Conceptual example

```text
IF company_name is not empty
AND email does not end with "gmail.com"
AND email does not end with "hotmail.com"
    -> qualified branch
ELSE
    -> not-qualified branch
```

The transcript tests the node and observes that a lead with a company name and a non-matching email domain follows the true branch.

### Improve this rule for a real business

- Define qualification criteria with the sales team.
- Validate email formats separately if necessary.
- Decide how to handle missing or malformed values.
- Do not assume that every Gmail/Hotmail address is unqualified; this is only a business-specific example.
- Consider adding a separate route to record or review unqualified leads rather than silently dropping them.

## 8. Step 4A — Send a Slack notification

The Slack action alerts the sales team that a new qualified lead has arrived.

### Configuration outline

1. Add a Slack node on the qualified path.
2. Select the message resource/action that sends a message.
3. Choose the target channel.
4. Write a message and insert dynamic values from the incoming lead.

Example message:

```text
New lead alert
Name: {{ first name }} {{ last name }}
Email: {{ email }}
Company: {{ company name }}
```

The placeholder notation above is explanatory. In n8n, insert fields using the expression/data picker appropriate to the node and version you are using.

Test the node and confirm the message appears in the selected Slack channel.

## 9. Step 4B — Create a follow-up task in Notion

Notion is used as the sales team's task tracker.

### A. Create an internal integration

The lesson demonstrates an internal Notion integration for one workspace:

1. Open Notion settings and the area for managing integrations.
2. Create a new integration and associate it with the correct workspace.
3. Copy the integration secret and store it securely.
4. Select only the capabilities the workflow needs.
5. Open the relevant task-tracker page/database and add the integration through the page's Connections/Share controls.
6. In n8n, create the Notion credential using the integration secret and test the connection.

**Key point:** Creating the integration is not always enough. The integration must also be granted access to the specific Notion page/database it needs to use.

### B. Create a database page

The transcript uses a Notion **database page** creation action for the task tracker.

1. Choose the correct Notion database.
2. Set the task title, for example, `New lead alert`.
3. Add a paragraph/block containing lead details such as name, email, and company.
4. Map any relevant database properties, such as status or effort level, if they exist in the target database.
5. Test the action and inspect the created page.

Notion database columns are properties of each database page/row. A block in the page body is different from a database property, so configure the title, properties, and body content in the appropriate places.

### Internal vs. public integrations

- **Internal integration:** Intended for a workspace you control.
- **Public integration:** Designed for use by users/workspaces beyond your own and may require additional setup, review, or published details.

For this lesson's single-workspace automation, the transcript uses an internal integration.

## 10. Step 4C — Create an Airtable CRM record

Airtable acts as a lightweight CRM in this example.

### A. Connect Airtable to n8n

The lesson demonstrates two authentication approaches:

**Personal Access Token (PAT)**
1. In Airtable's developer area, create a personal access token.
2. Give it the required scopes shown in the lesson:
   - `data.records:read`
   - `data.records:write`
   - `schema.bases:read`
3. Select the base or resources it should access.
4. Copy the token and save it as an n8n credential.
5. Test the credential.

**OAuth2**
1. Copy the redirect URL shown by n8n.
2. Register an OAuth integration in Airtable and provide the redirect URL.
3. Copy the client ID and generate/copy the client secret.
4. Configure the required scopes.
5. Save the integration and complete the account-authorization flow from n8n.

Exact labels and screens can change over time, so follow the current n8n and Airtable credential documentation if your interface differs.

### B. Follow least privilege

The transcript shows an example that grants access to all resources, but explicitly notes that this is not the recommended approach for a real setup.

**Best practice:** Grant access only to the specific base/workspace required by the workflow, and only the permissions needed. Treat PATs, client secrets, and integration tokens as passwords. Never paste them into public notes or commit them to source control.

### C. Create the record

1. Add an Airtable node on the qualified path.
2. Select the **Record** resource and **Create** operation.
3. Choose the intended base and table (the transcript uses a customers table).
4. Map the incoming fields to the Airtable columns, such as name and email.
5. Test the action and confirm that the record appears.

### Field-type mismatch

The transcript points out that a free-text form value can conflict with a restricted Airtable field, such as a single-select field with predefined options. For example, a role typed as `owner` may not match an allowed option.

To prevent this:
- Use controlled choices in the form when the CRM expects a fixed set of values.
- Or normalize/validate the value in n8n before creating the record.
- Do not map a field if the incoming value cannot safely fit the target field type.

## 11. How data moves through the workflow

A single form response might contain:

```json
{
  "firstName": "Dheeraj",
  "lastName": "Sharma",
  "email": "dheeraj@example.com",
  "companyName": "Example Ltd",
  "role": "Owner"
}
```

This JSON is an illustrative example of the kind of data involved, not a guaranteed exact payload from every Google Forms/Apps Script setup. Inspect the Webhook's actual output and use the exact field names it provides.

The same lead data is then mapped into different destinations:

- Google Sheets: a row in the central lead sheet.
- If node: the decision about whether it qualifies.
- Slack: a short alert for the team.
- Notion: a task page with follow-up details.
- Airtable: a customer record in the CRM.

## 12. Troubleshooting checklist

| Symptom | Likely cause | What to check |
|---|---|---|
| Webhook receives nothing | Test listener is not active or wrong URL is configured | Start listening for a test event; copy the current node's URL |
| Apps Script opens with an account/session error | Browser cookies or multiple signed-in accounts | Try a clean/private browser session, as suggested in the transcript |
| Google Sheets asks for a match column | An append-or-update operation was selected | Choose Append Row if you only want to add a new row |
| Notion database is not visible | Integration lacks access to the page/database | Connect/share the page with the integration and verify permissions |
| Airtable credential fails | Wrong token, missing scopes, or wrong access scope | Verify token and required scopes; limit access appropriately |
| Airtable rejects a value | Incoming value does not match the destination field type/options | Use controlled form choices or validate/transform the value |
| Slack/Notion/Airtable action fails | Credential, resource, or field mapping is wrong | Test each node independently and inspect its error/output |

## 13. Safe build order

Build and test the workflow incrementally:

1. Webhook receives one sample response.
2. Google Sheets appends the expected row.
3. If node routes a qualified and unqualified test case correctly.
4. Slack sends one message on the intended branch.
5. Notion creates one task with correct properties and content.
6. Airtable creates one record with valid field values.
7. Run an end-to-end test with both qualified and unqualified leads.
8. Activate the workflow only after checking the branch connections, credentials, and duplicate-submission behavior.

## 14. Practice exercises

### Exercise 1
Why does this workflow use `POST` for the Webhook?

### Exercise 2
What is the difference between **Append Row** and **Append or Update Row** in the Google Sheets step?

### Exercise 3
A lead has no company name. Which branch should it follow under the example qualification rule?

### Exercise 4
Why must the Notion integration be connected to the target task-tracker page/database even after creating the integration?

### Exercise 5
Why is granting an Airtable token access to every base less desirable than granting access only to the required base?

### Exercise 6
A form lets users type any role, but Airtable expects a fixed single-select value. Name two ways to prevent a failed record creation.

### Suggested answers

1. The form integration sends data to n8n, rather than asking n8n to retrieve it.
2. Append Row adds a new row; append-or-update also needs a matching key to identify an existing row for updating.
3. It should go to the not-qualified/false branch if company name must not be empty.
4. The integration needs permission to access the particular Notion content.
5. Least-privilege access limits the damage if a token is exposed or misused.
6. Use fixed choices in the form, or validate/normalize the value in n8n before sending it to Airtable.

## 15. Final recap

This lesson demonstrates a practical multi-app lead-management workflow:

**Google Forms → Webhook → Google Sheets → If → Slack + Notion + Airtable**

The central ideas are:

- Use a trigger to start the workflow.
- Map fields deliberately when sending data between applications.
- Use conditional logic to decide which actions should run.
- Set up credentials and resource permissions correctly.
- Test each integration separately before testing the complete workflow.
- Use least privilege and validate data before writing to business systems.
