# Smart Inventory & Warehouse Automation Platform

A modular inventory and warehouse automation platform built entirely on **n8n**. It continuously monitors stock across warehouses, uses AI to recommend reorder quantities, routes high value purchases through a human approval step, reconciles deliveries automatically, reports on inventory health every week, and keeps a full audit trail of everything it does.

**Domain:** Supply Chain & Inventory Management
**Automation platform:** n8n
**Data store:** Google Sheets
**AI provider:** Groq (Llama 3 / Llama 3.3)

---

## Table of contents

1. [Problem statement](#problem-statement)
2. [Objectives](#objectives)
3. [Architecture overview](#architecture-overview)
4. [Workflows](#workflows)
5. [Event flow, end to end](#event-flow-end-to-end)
6. [Data model](#data-model)
7. [Advanced features checklist](#advanced-features-checklist)
8. [Node count summary](#node-count-summary)
9. [Repository structure](#repository-structure)
10. [Tech stack](#tech-stack)
11. [Setup and installation](#setup-and-installation)
12. [Configuration reference](#configuration-reference)
13. [Testing the platform](#testing-the-platform)
14. [Troubleshooting](#troubleshooting)
15. [Security notes](#security-notes)
16. [Possible extensions](#possible-extensions)
17. [Documentation](#documentation)
18. [License](#license)
19. [Author](#author)

---

## Problem statement

A retail company manages inventory across multiple warehouses and stores. Because stock is monitored manually, the company regularly runs into stock shortages, overstocking, delayed purchase orders, and inaccurate inventory records. Warehouse managers spend a large share of their time tracking deliveries, updating stock levels by hand, and coordinating with suppliers over email and phone, with no consistent record of who approved what or why.

This platform replaces that manual process with a connected set of automations that watch inventory continuously, predict shortages before they become critical, initiate purchase requests automatically, route spend for approval, and keep procurement, delivery, and reporting all in sync with a single source of truth.

## Objectives

- Monitor inventory levels automatically, across every product and warehouse, without manual checks
- Prevent both stock shortages and overstocking
- Automate procurement and supplier communication end to end
- Track inventory movement from purchase order through to delivery
- Generate recurring inventory analytics and reports
- Maintain a complete, queryable audit trail of every automated action and every error

## Architecture overview

The platform is composed of **six independent n8n workflows**. Four form the core operational pipeline (monitor, detect, approve, deliver), one runs an independent weekly analytics cycle, and one acts as a shared audit and error handling hub that the others report into. All six workflows read and write the same Google Sheet, which functions as the system of record.

```
                         SMART INVENTORY & WAREHOUSE AUTOMATION PLATFORM

  +---------------------------------------------------------------------------+
  |                              DATA LAYER (Google Sheets)                   |
  |   Inventory_Master | Supplier_Registry | Purchase_Orders                  |
  |   Delivery_Log | Audit_Trail                                              |
  +---------------------------------------------------------------------------+
               ^                ^                ^                ^
               |                |                |                |
  +----------------+  +----------------+  +----------------+  +----------------+
  |  WF1           |  |  WF2           |  |  WF3           |  |  WF4           |
  |  Inventory     |  |  Low Stock     |  |  Supplier      |  |  Delivery      |
  |  Monitoring    |->|  Detection &   |->|  Comms &       |->|  Tracking &    |
  |  (Cron 30 min) |  |  Purchase Req  |  |  Approval      |  |  Inventory Upd |
  +----------------+  +----------------+  +----------------+  +----------------+
               |                |                |                |
               +----------------+----------------+----------------+
                                        |
                                        v
                          +---------------------------+
                          |  WF6                       |
                          |  Audit Trail & Error       |
                          |  Handling (shared hub)     |
                          +---------------------------+

  +--------------------------------------------------------------------------+
  |  WF5  Inventory Analytics & Reports  (Cron, weekly, independent)         |
  |  Reads Inventory_Master -> AI insight -> email report                    |
  +--------------------------------------------------------------------------+

  EXTERNAL SERVICES:  Groq LLM (AI decisions)  |  Telegram (approvals/alerts)
                       Gmail (PO emails / reports)
```

**Design principles:**

- **Single responsibility per workflow.** Each workflow does one job well and hands off to the next over a webhook, rather than one large monolithic workflow.
- **Shared state, not shared execution.** Workflows don't call each other's internals; they communicate through HTTP and read/write a common data store, so any one workflow can be edited, tested, or redeployed independently.
- **Human in the loop where it matters.** Spend above a threshold always stops for a person; everything below it flows straight through.
- **Everything observable.** Every workflow can report into the same audit log, so failures and key events are traceable from one place instead of scattered across execution histories.

## Workflows

| ID | Name | Trigger | Primary role |
|----|------|---------|---------------|
| **WF1** | Inventory Monitoring & Stock Updates | Cron, every 30 minutes | Scans stock levels, flags items at or below reorder point, kicks off procurement |
| **WF2** | Low Stock Detection & Purchase Request | Webhook, `/low-stock-trigger` | Looks up supplier info, asks an LLM for a recommended order quantity, creates and stores the purchase order |
| **WF3** | Supplier Communication & Approval | Webhook, `/po-approval` | Routes purchase orders over a spend threshold to a human approver via Telegram; auto-approves the rest; emails the supplier |
| **WF4** | Delivery Tracking & Inventory Update | Webhook, `/delivery-received` | Matches an incoming delivery to its purchase order, updates stock, logs the delivery |
| **WF5** | Inventory Analytics & Reports | Cron, weekly | Aggregates stock health metrics, has an LLM write a short report, emails it out |
| **WF6** | Audit Trail & Error Handling | Webhook, `/audit-log`, plus an Error Trigger | Central logging hub for workflow events and runtime errors; sends critical alerts |

Full node by node detail for every workflow, including exact node types and what each one reads or writes, is in [`docs/Workflow_Documentation.pdf`](docs/Workflow_Documentation.pdf).

### WF1, Inventory Monitoring & Stock Updates

Runs every 30 minutes. Reads the entire `Inventory_Master` sheet, filters for any item where `current_stock <= reorder_point`, and, if it finds any, posts the list to WF2. Every run, regardless of outcome, also posts a scan summary to WF6 for the audit trail.

### WF2, Low Stock Detection & Purchase Request

Triggered by WF1. For each low stock item, it looks up the assigned supplier's details, including average delivery time, from `Supplier_Registry`, then calls a Groq hosted LLM with the item's stock levels and the supplier's lead time, asking it to recommend an order quantity and explain its reasoning. It parses that response, builds a purchase order record, appends it to `Purchase_Orders`, and posts the PO on to WF3.

### WF3, Supplier Communication & Approval

Triggered by WF2. If the PO's estimated cost is above 5,000 dollars, it sends an approval request over Telegram with Approve and Reject links, then pauses on a Wait node until a manager responds. Once approved, either automatically for smaller orders or manually, it updates the PO's status in `Purchase_Orders` and emails the supplier with the order details.

### WF4, Delivery Tracking & Inventory Update

Triggered by an external delivery event, such as a warehouse scanning in a shipment. Looks up the original PO by ID, compares the delivered quantity to what was ordered, adds the delivered quantity to the product's current stock in `Inventory_Master`, logs the delivery in `Delivery_Log`, and sends a Telegram confirmation.

### WF5, Inventory Analytics & Reports

Runs weekly, independently of the rest of the pipeline. Reads the full inventory sheet, computes counts of low stock and overstocked items, and has an LLM turn that summary into a short, readable report, which is emailed to the inventory team.

### WF6, Audit Trail & Error Handling

The shared logging backbone. It exposes a webhook that any workflow can post structured events to, WF1 uses it for scan summaries, and it also has a native n8n Error Trigger that fires automatically for any workflow configured to use WF6 as its error workflow. Every event, whether a normal log entry or an error, is timestamped, tagged with a severity, and appended to `Audit_Trail`. Critical severity errors additionally trigger a Telegram alert.

## Event flow, end to end

```
STEP 1  Cron fires (every 30 min)
        WF1 reads Inventory_Master
        WF1 filters items where current_stock <= reorder_point
        IF none low -> WF1 logs a SCAN event to WF6 and stops
        IF low stock found -> WF1 POSTs items[] to WF2, then logs to WF6

STEP 2  WF2 webhook receives items[]
        For each item: WF2 looks up Supplier_Registry
        WF2 asks the LLM for a recommended order_quantity + reasoning
        WF2 builds a PO record (po_id, qty, cost, status = "Pending Approval")
        WF2 appends the PO to Purchase_Orders
        WF2 POSTs the PO to WF3

STEP 3  WF3 webhook receives the PO
        IF estimated_cost > $5000:
            WF3 sends a Telegram approval request (Approve / Reject links)
            WF3 pauses on a Wait node until a manager responds
            On resume: IF approved -> status set to Approved, supplier emailed
                       IF rejected -> status set to Rejected, workflow stops
        ELSE (<= $5000):
            Auto-approved -> status set to Approved -> supplier emailed

STEP 4  Supplier fulfills -> a delivery event is posted to WF4
        WF4 looks up the original PO by po_id
        WF4 checks delivered quantity against ordered quantity
        WF4 fetches current stock and adds the delivered quantity
        WF4 updates Inventory_Master, appends a row to Delivery_Log
        WF4 sends a Telegram delivery confirmation

STEP 5 (parallel, weekly)
        WF5 cron fires once a week
        WF5 reads Inventory_Master and computes low stock / overstock counts
        WF5 asks the LLM to write a short analyst style report
        WF5 emails the report to the inventory team

THROUGHOUT
        WF6 listens on /audit-log for explicit events from other workflows
        WF6 also catches runtime errors from any workflow using it as their
        error workflow, logs everything to Audit_Trail, and sends a
        Telegram alert for CRITICAL severity events
```

## Data model

All five sheets live in one Google Spreadsheet.

**`Inventory_Master`**

| Column | Type | Notes |
|---|---|---|
| `product_id` | string | Primary key, used to match rows across sheets |
| `product_name` | string | |
| `category` | string | |
| `warehouse_id` | string | |
| `current_stock` | number | Updated by WF4 on delivery |
| `reorder_point` | number | Threshold WF1 checks against |
| `max_stock_level` | number | Used to compute suggested order quantity and overstock flag |
| `unit_price` | number | Used to compute estimated cost |
| `supplier_id` | string | Foreign key into `Supplier_Registry` |

**`Supplier_Registry`**

| Column | Type | Notes |
|---|---|---|
| `supplier_id` | string | Primary key |
| `supplier_name` | string | |
| `avg_delivery_days` | number | Fed into the AI reorder quantity prompt |
| additional contact fields | | as needed |

**`Purchase_Orders`**

| Column | Type | Notes |
|---|---|---|
| `po_id` | string | Generated by WF2, format `PO-######` |
| `product_id` | string | |
| `supplier_id` | string | |
| `order_quantity` | number | AI recommended, or calculated fallback |
| `estimated_cost` | number | `order_quantity * unit_price` |
| `status` | string | `Pending Approval`, `Approved`, or `Rejected` |
| `ai_recommendation` | string | Reasoning text from the LLM |

**`Delivery_Log`**

| Column | Type | Notes |
|---|---|---|
| `po_id` | string | |
| `quantity_delivered` | number | |
| additional fields | | auto mapped from the incoming webhook payload |

**`Audit_Trail`**

| Column | Type | Notes |
|---|---|---|
| `event_id` | string | `EVT-<timestamp>` or `ERR-<timestamp>` |
| `timestamp` | string | ISO 8601 |
| `workflow` | string | Source workflow name |
| `event_type` | string | e.g. `SCAN`, `ERROR` |
| `details` | string | JSON stringified event payload |
| `severity` | string | `INFO` or `CRITICAL` |

## Advanced features checklist

| Feature | Implemented in | How |
|---|---|---|
| AI powered decision making | WF2, WF5 | Groq hosted LLM recommends reorder quantities and writes weekly reports |
| Human approval step(s) | WF3 | Telegram approve or reject flow with a Wait node for orders over $5,000 |
| Error handling and retry logic | WF6, HTTP nodes in WF1 to WF4 | Dedicated Error Trigger workflow; retry on fail available on HTTP Request nodes |
| Logging and audit trail | WF6 | Central webhook based log sink, appends every event to `Audit_Trail` |
| Scheduled workflows, Cron | WF1, WF5 | 30 minute stock scan, weekly report generation |
| Webhook triggered workflows | WF2, WF3, WF4, WF6 | React immediately to upstream events instead of polling |
| Conditional branching and loops | WF1, WF3, WF4, WF2 | Low stock check, spend threshold check, quantity match check, per item iteration over low stock lists |

## Node count summary

| Workflow | Node count |
|---|---|
| WF1 | 6 |
| WF2 | 8 |
| WF3 | 8 |
| WF4 | 7 |
| WF5 | 6 |
| WF6 | 7 |
| Total | 42 |

Exceeds the assignment's minimum of 4 to 6 workflows and 20 to 25 total nodes.

## Repository structure

```
.
├── README.md
├── .gitignore
├── Architecture diagram.pdf
├── WF1_Inventory_Monitoring.json
├── WF2_Low_Stock_Detection.json
├── WF3_Supplier_Approval.json
├── WF4_Delivery_Tracking.json
├── WF5_Analytics_Reports.json
├── WF6_Audit_Error_Handling.json
├── docs/
│   ├── Technical_Documentation.pdf   (problem analysis, architecture, implementation, advanced features)
│   └── Workflow_Documentation.pdf    (node by node reference for every workflow)
├── sample-data/
│   └── example rows for seeding the Google Sheet (Inventory_Master, Supplier_Registry, etc.)
└── workflow-diagram/
    └── exported architecture and event flow diagram images
```

All six workflow JSON files live at the repository root rather than in a `workflows/` subfolder. The two write-ups plus the standalone architecture diagram are split into `docs/` and the root level PDF.

## Tech stack

- **Automation engine:** n8n, self hosted or n8n.cloud
- **Data store:** Google Sheets, via the native Google Sheets node
- **AI:** Groq API, `llama-3.3-70b-versatile` and `llama3-70b-8192`, through n8n's LangChain integration, `chainLlm` and `lmChatGroq` nodes
- **Human notifications and approval:** Telegram Bot API
- **Email:** Gmail, via OAuth2
- **Transport between workflows:** native n8n Webhook and HTTP Request nodes

## Setup and installation

### Prerequisites

- An n8n instance, self hosted, Docker, or n8n.cloud, with the LangChain community nodes available
- A Google account with access to Google Sheets and the Sheets API enabled
- A Groq API key
- A Telegram bot token and a target chat or group ID
- A Gmail account for outbound email, OAuth2 app credentials

### Steps

1. **Clone this repository.**
   ```bash
   git clone <your-repo-url>
   cd <repo-name>
   ```

2. **Create the Google Sheet.** Set up one spreadsheet with five tabs, named exactly: `Inventory_Master`, `Supplier_Registry`, `Purchase_Orders`, `Delivery_Log`, `Audit_Trail`. Add the columns listed in the Data model section to each tab, and seed `Inventory_Master` and `Supplier_Registry` with a few rows so the platform has something to act on.

3. **Import the workflows.** In n8n: Workflows, then Import from File, and import each of the six `WF*.json` files at the repository root, one at a time.

4. **Create credentials in n8n:**
   - Google Sheets OAuth2, used by every workflow that touches the spreadsheet
   - Groq API key, used by WF2 and WF5
   - Telegram API, used by WF3, WF4, WF6
   - Gmail OAuth2, used by WF3 and WF5

   The imported JSON references credentials by name only, it contains no secrets. Reassign each Google Sheets, Groq, Telegram, and Gmail node in every workflow to point at the credentials you just created.

5. **Point every workflow at your spreadsheet.** In each Google Sheets node, replace the `documentId` value with your own spreadsheet's ID, found in its URL.

6. **Update inter workflow webhook URLs.** The HTTP Request nodes in WF1, WF2, and WF3 call full URLs pointing at a specific n8n instance. After activating WF2, WF3, WF4, and WF6, so their webhook paths exist, copy each one's production webhook URL from n8n and update the corresponding HTTP Request node.

7. **Set your Telegram chat ID.** Replace the `chatId` field in every Telegram node with your own group or channel's chat ID.

8. **Set your approval threshold, optional.** WF3's `IF > $5000` node controls the human approval cutoff, change the comparison value if you want a different threshold.

9. **Activate in this order:** WF6, WF4, WF3, WF2, since these are webhook based and need their endpoints to exist first, then WF1 and WF5, which are cron based and need somewhere to send events once they fire.

10. **Wire up error workflows, optional but recommended.** In each workflow's settings, set WF6 as the Error Workflow so failures are automatically caught by WF6's Error Trigger.

## Configuration reference

| Setting | Where | Default | Purpose |
|---|---|---|---|
| Stock scan interval | WF1, Cron Trigger | 30 minutes | How often inventory is checked |
| Report cadence | WF5, Cron Trigger Weekly | Weekly | How often the analytics report is generated |
| Approval threshold | WF3, `IF > $5000` | $5,000 | Spend level above which human approval is required |
| AI model, procurement | WF2, Groq Chat Model | `llama-3.3-70b-versatile` | Model used for reorder quantity recommendations |
| AI model, reporting | WF5, AI Generate Insights | `llama3-70b-8192` | Model used for the weekly report narrative |

## Testing the platform

1. Add a test row to `Inventory_Master` with `current_stock` at or below `reorder_point`.
2. Manually execute WF1, or wait for the next cron tick.
3. Confirm a new row appears in `Purchase_Orders` with status `Pending Approval` or `Approved`, depending on the estimated cost.
4. If the order exceeded the approval threshold, confirm the Telegram approval message arrives, and click Approve.
5. Confirm the PO's status updates to `Approved` and the supplier email is sent.
6. Post a test payload to WF4's `/delivery-received` webhook with the matching `po_id` and a `quantity_delivered`, and confirm `Inventory_Master` and `Delivery_Log` update.
7. Check `Audit_Trail` for the corresponding SCAN event from WF1.
8. Manually execute WF5 and confirm the report email arrives.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| WF1 never triggers WF2 | No rows meet the low stock condition | Confirm `current_stock <= reorder_point` for at least one row |
| WF2's PO has no AI reasoning | LLM response did not parse as JSON | Check the Groq credential and model name, WF2 falls back to a calculated quantity if parsing fails |
| Telegram messages do not arrive | Wrong `chatId` or bot not added to the chat | Verify the bot is a member of the target chat and the ID is correct, group IDs are negative numbers |
| WF3 never resumes after approval | `resumeUrl` webhook not reachable, or Wait node misconfigured | Confirm the workflow is active and the base URL in `resumeUrl` matches your instance |
| Emails do not send | Gmail OAuth2 credential expired or not authorized | Reconnect the Gmail credential in n8n |
| Nothing appears in `Audit_Trail` | Workflow not calling WF6, or WF6 not activated | Activate WF6 first, confirm the audit log webhook URL is correct in the calling workflow |

## Security notes

- This repository does not contain any API keys or credential secrets. n8n stores those separately in its own credential manager, the exported JSON only references credentials by name.
- Before using this in a real environment, replace the placeholder email addresses, Telegram chat ID, and spreadsheet ID with your own.
- Avoid committing n8n `pinData` execution captures to version control, they can contain real request data, including IP addresses and headers from live executions. This repository's WF3 file has already had its `pinData` stripped for that reason.
- Treat the Google Sheet as sensitive if it contains real supplier or pricing data, and restrict its sharing settings accordingly.

## Possible extensions

- Replace Google Sheets with a proper database, such as PostgreSQL or Airtable, as the platform scales past a single spreadsheet's practical row limits
- Add supplier performance scoring to WF5's weekly report, based on delivery time accuracy from `Delivery_Log`
- Add a discrepancy handling branch in WF4 for when delivered quantity does not match the ordered quantity
- Add retry on fail configuration to every HTTP Request node calling between workflows, with exponential backoff
- Add a Slack option alongside Telegram for approval and alerting
- Add role based routing in WF3, so different categories of spend go to different approvers

## Documentation

- `docs/Technical_Documentation.pdf`, problem analysis, workflow architecture, implementation summary, and advanced features writeup
- `docs/Workflow_Documentation.pdf`, detailed node by node reference for all six workflows
- `Architecture diagram.pdf`, standalone system architecture and event flow diagrams
- `workflow-diagram/`, exported diagram images
- `sample-data/`, example rows for seeding the Google Sheet before a first run

## License

MIT

## Author 
Apoorva Yadav
