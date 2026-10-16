# Setting Up n8n Cloud
## A beginner-friendly guide to n8n Cloud vs self-hosting


## 1. What this lesson is about

This lesson begins the practical section of the course. It moves from understanding automation and comparing tools to getting ready to build workflows in n8n.

Upcoming section will cover:
- setting up n8n on a local machine or server;
- registering for n8n Cloud;
- understanding the n8n interface and workspace;
- learning about nodes, triggers, workflow execution, branches, and activation;
- building a workflow from scratch, one node at a time.



## 2. The two main ways to use n8n

### Option A — n8n Cloud

n8n Cloud is the hosted option. You use it through a browser instead of installing and maintaining the application yourself.

**Advantages**
- Quick to start: open the website, register, and sign in.
- No local installation or server setup is required.
- Templates and integrations are available within the platform.
- Suitable for beginners who want to start building workflows without managing infrastructure.

**Trade-offs**
- Free/trial access is limited, and ongoing use may require a paid plan.
- You rely on the hosted service for your workspace, data, and workflow logic, so you have less infrastructure-level control than with a self-hosted setup.

### Option B — Local or self-hosted n8n

Local or self-hosted n8n are for people who are comfortable managing computers or servers and want more control over data and workflow logic.

**Potential advantages**
- More direct control over where the service runs and how it is configured.
- Greater flexibility over the deployment environment.

**Responsibilities**
- You may need to manage installation, updates, security, backups, monitoring, and server capacity.
- Self-hosting is not automatically more secure; the setup must be configured and maintained properly.


## 3. Cloud vs self-hosting: comparison

| Question | n8n Cloud | Local / self-hosted n8n |
|---|---|---|
| How do I start? | Register and sign in through a browser | Install and configure n8n on a machine or server |
| Infrastructure management | Mostly handled by the hosted service | Your responsibility, or your hosting provider’s responsibility |
| Beginner-friendliness | High for getting started quickly | Depends on technical comfort |
| Control over deployment | Less direct control | More direct control |
| Maintenance | Less server maintenance for you | Plan for updates, backups, security, and monitoring |
| Best fit from the lesson | Beginners who want to start quickly | Technical users who want more control and flexibility |

**Decision shortcut:** if your immediate goal is to learn workflow building with the least setup, start with Cloud. Consider self-hosting later if you need deployment control and are ready to maintain it.

---

## 4. Step-by-step: signing up for n8n Cloud


### Step 1 — Open the official website

Open **[n8n.io](https://n8n.io/)** in your browser. 

### Step 2 — Choose the free-start option

On the website, choose the option to get started for free. Website labels and the sign-up flow may change.

### Step 3 — Complete the registration form

Fills in:
- name;
- a company email address;
- a secure password;
- a unique account/workspace name.

The workspace name becomes part of the workspace URL. Choose a name carefully and avoid including private information.

### Step 4 — Review the trial terms

You will receive a **14-day trial** and no payment card was requested in that sign-up flow.This is not guarantee that the same offer or requirements are available to every user today. Check the current sign-up page before proceeding.

### Step 5 — Answer onboarding questions

Answers short questions about their team, company size, coding experience, and how they heard about n8n. These answers help complete the onboarding flow; use your own accurate answers.

### Step 6 — Invite teammates or skip

You sees an option to invite other team members and skips it for now. If you are learning alone, you can generally defer invitations when the interface offers a skip option.

### Step 7 — Open the workspace

After setup, open the new workspace. The workspace URL appears in the browser, and the n8n interface is ready to use.

### Step 8 — Create a workflow

Select **Create workflow**. The workflow canvas opens, and you can begin adding nodes. The next lessons are intended to explain the interface and node configuration in more detail.

---

## 5. Understanding three terms before building

### Workspace
The account area where your n8n work is organized. The workspace has its own URL.

### Workflow canvas
The visual area where you assemble the steps of an automation by adding and connecting nodes.

### Node
A component in the workflow that performs a job, such as starting a workflow, checking information, transforming data, or taking an action. This lesson previews nodes but does not explain every node type in detail.

### Basic mental model

```text
Sign up / sign in
       |
       v
Open your workspace
       |
       v
Create a workflow
       |
       v
Workflow canvas
       |
       v
Add and connect nodes
       |
       v
Configure and test the workflow
```

The final configuration and testing steps are the natural next steps for building a workflow.

---

## 6. Which option should you choose as a learner?

**Start with n8n Cloud if:**
- you are new to workflow automation;
- you want to focus on nodes, triggers, conditions, and workflow logic first.

**Consider local/self-hosted n8n if:**
- you are comfortable with command-line tools or server administration;
- you need more control over the deployment environment;
- you are willing to maintain updates, backups, security, and monitoring.

You do not have to make a permanent choice on day one. Learn the interface and basic workflows first, then reassess based on your actual needs.

---

## 7. Before you begin: checklist

- [ ] I can access the official n8n website.
- [ ] I have an email address that the current registration form accepts.
- [ ] I can create and store a strong password safely.
- [ ] I have reviewed the current trial and plan terms.
- [ ] I have chosen a workspace name that does not reveal private information.
- [ ] I have opened the workspace successfully.
- [ ] I can find **Create workflow** and open the canvas.
- [ ] I know whether I am using Cloud or a self-hosted instance.

---

## 8. Troubleshooting notes

- **Registration option looks different:** follow the current official website; the interface may have changed.
- **Email is rejected:** read the form’s current requirements and use an address that satisfies them.
- **Workspace name is unavailable:** try a different unique name.
- **Trial or payment terms differs :** trust the current terms shown by n8n.
- **You cannot find Create workflow:** confirm that you are inside the workspace and signed in.

---

## 9. Active-recall quiz

Try answering before checking the key.

1. What is the main benefit of n8n Cloud for a beginner?
2. What are two trade-offs of using a hosted cloud workspace?
3. Who might prefer a self-hosted setup?
4. Why does the workspace name matter?
5. Does the supplied transcript show a full local installation procedure?
6. Why should you verify the trial details instead of assuming they have not changed?

### Answer key

1. It allows you to start through a browser without installing or managing the application server.
2. Trial/plan limits may apply, and you have less direct infrastructure-level control.
3. Someone comfortable managing a machine or server who wants more deployment control and flexibility.
4. It contributes to the workspace URL and identifies the workspace.
5. No. The excerpt introduces local/self-hosted use but demonstrates Cloud registration.
6. Trial availability, registration requirements, and plans can change over time.

---

## 10. Key takeaways

- The lesson marks the shift from theory to hands-on n8n practice.
- n8n Cloud is the quickest path in the lesson because it avoids local installation and server management.
- Self-hosting offers more direct deployment control but comes with maintenance responsibilities.
- The demonstrated sign-up journey is: official website → free-start option → registration details → onboarding questions → workspace → **Create workflow**.
- The next lessons are expected to explain the interface, nodes, triggers, execution, branches, activation, and how to build a workflow from scratch.

**Study suggestion:** follow the sign-up steps in your own browser, then write down what you see on the canvas. Keep your own screenshots or notes so you can connect the next lesson’s terminology to the actual interface.
