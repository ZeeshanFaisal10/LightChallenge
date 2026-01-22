# Invoice Approval Workflow (Light Challenge)

This repository implements an invoice approval routing workflow.

The core idea is that *workflow rules are data*, not hardcoded `if/else` logic. A workflow is represented as a small decision graph (nodes + transitions). You can swap the active workflow at runtime by a POST request to a new definition.

---

## What this app does

When you submit an invoice, the backend:

1. Loads the **active workflow definition** (in-memory for this challenge).
2. Evaluates decision nodes against the invoice context (amount, department, manager approval flag).
3. Produces one or more **actions** (e.g., Slack to CFO, email to Finance Manager).
4. Resolves the “target key” (CFO / FinanceTeam / etc.) to a concrete approver name.
5. Sends a **console notification** like `sending approval via Slack to CFO - Daniel`.

This matches the challenge requirements:
- Inputs: invoice amount, department, manager approval requirement
- Output: list of selected approvers
- Notifications: printed to console (Slack/email message text)

## Database model design (proposed)

Although this challenge uses in-memory repositories, the following relational model supports both:

- **Workflow configuration** (editable without code changes)
- **Workflow execution audit trail** (what happened for each invoice)
  
![code_exercise_diagram](https://github.com/ZeeshanFaisal10/LightChallenge/blob/main/mermaid-diagram-2026-01-22-123238.png?raw=true)

---

## API endpoints

### 1) Execute the workflow

`POST /workflow`

**Request**
```json
{
  "invoiceAmount": 12000,
  "department": "Marketing",
  "managerApprovalRequired": true
}
```

**Response**
```json
{
  "selectedApprovers": [
    {
      "approverName": "CMO - Trevor",
      "channel": "EMAIL",
      "targetKey": "CMO"
    }
  ]
}
```

Notes:
- `department` is validated (non-blank + must match known departments derived from the workflow definition).
- Approver selection is deterministic for the same input (seeded selection) so repeated calls are stable.

### 2) List departments supported by the active workflow

`GET /departments`

Returns departments extracted from the workflow’s decision nodes.

### 3) Get or replace the active workflow definition

`GET /workflow-definitions`

Returns the active workflow definition.

`POST /workflow-definitions`

Replaces the active workflow definition with the JSON you provide. This is the main “dynamic workflow” mechanism: modify routing without code changes.

---

## Default workflow logic (the graph)

The default workflow is configured in-memory as nodes and transitions:

- **N1**: if `amount > 10000`
  - YES → **N2**
  - NO → **N3**
- **N2**: if `department == Marketing`
  - YES → **A2** (Email CMO)
  - NO → **A1** (Slack CFO)
- **N3**: if `amount > 5000`
  - YES → **N4**
  - NO → **A3** (Slack FinanceTeam)
- **N4**: if `managerApprovalRequired == true`
  - YES → **A4** (Email FinanceManager)
  - NO → **A3** (Slack FinanceTeam)
- Actions always go to **END**

Because the workflow is a data structure (nodes + transitions), you can add new decision points (e.g., region, vendor risk score, payment method) by updating the workflow definition schema and data, without rewriting the engine.

---

## Implementation notes

### Workflow representation
A workflow is a graph of:
- `Node`:
  - `DECISION` node with `Condition(field, op, value)`
  - `ACTION` node with `Action(channel, targetKey)`
  - `END` node
- `Transition(fromNode, outcome YES/NO, toNode)`

### Workflow engine
The engine starts from `startNodeId` and walks the graph until it hits `END`:
- DECISION → evaluate condition → follow YES/NO transition
- ACTION → append action → follow YES transition (one-way)

### Approver resolution
Actions point to a logical `targetKey` (e.g., `CFO`, `FinanceTeam`). The approver directory maps target keys to one or more real names. For this exercise it’s in-memory and a deterministic random choice is used when there are multiple candidates.

### Notifications
Instead of calling Slack/email, the app prints:
- `sending approval via Slack to <name>`
- `sending approval via email to <name>`

---

## Simple UI (mobile app)

The `/app` folder contains a small React Native / Expo UI to call the backend.

Typical flow:
1. Enter amount.
2. Pick a department (populated from `GET /departments`).
3. Toggle manager approval requirement.
4. Tap “Submit Workflow”.
5. Displays the `selectedApprovers` list.

