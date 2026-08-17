# Smart-Inventory-Warehouse-Automation-Platform

## 1. Problem Analysis

### Business Context
In fast-paced retail and e-commerce environments, inventory management is often a highly manual, error-prone process. Tracking stock levels, reordering products when supplies run low, keeping track of supplier deliveries, and aggregating data for weekly reports take significant administrative overhead and often lead to stockouts or overstocking. 

### Stakeholders
- **Inventory Managers:** Need real-time visibility into stock levels and automated low-stock alerts.
- **Procurement Officers:** Responsible for approving purchase orders (POs) and negotiating with suppliers.
- **Warehouse Staff:** Responsible for receiving deliveries and updating system inventory.
- **Executive Team:** Needs high-level weekly analytics to understand capital tied up in inventory.

### Pain Points
- **Manual Audits:** Manually checking spreadsheets to find out what items need reordering.
- **Siloed Communication:** Purchase order approvals happen over slow email threads.
- **Data Entry Errors:** Human errors when manually updating inventory sheets after a delivery.
- **Lack of Actionable Insights:** Weekly reporting is tedious and doesn't offer proactive business recommendations.

### Objectives
- **Automate Reordering:** Automatically detect low stock and draft Purchase Orders using AI for optimal quantity calculation.
- **Streamline Approvals:** Implement human-in-the-loop approvals via instant messaging (Telegram).
- **Ensure Data Integrity:** Automatically update inventory upon delivery confirmation.
- **Centralize Audit Logs:** Keep a robust error-handling and audit-logging system to track automation failures and major system actions.
- **Generate AI Insights:** Use LLMs to read inventory data and email a summarized weekly action report.

---

## 2. Workflow Architecture

This solution utilizes an event-driven micro-workflow architecture comprising 6 independent n8n workflows that total 42 nodes. They communicate seamlessly via webhooks to handle specific domains of the inventory lifecycle.

### Overall System Architecture
- **Data Layer:** Google Sheets (Master Inventory, Supplier Registry, Purchase Orders, Deliveries, Audit Trail).
- **Orchestration Layer:** n8n (6 workflows handling CRON jobs and Webhook routing).
- **Intelligence Layer:** NVIDIA Nemotron & Groq LLMs (LangChain nodes) for purchase order quantity estimation and weekly insight generation.
- **Notification & Approval Layer:** Telegram (instant alerts and human-in-the-loop interactive buttons) and Gmail (reports).

### Workflow Interaction / Event Flow
1. **WF1** runs on a schedule (Cron), scanning the database. If low stock is detected, it triggers **WF2** via webhook.
2. **WF2** fetches supplier data, uses AI to determine order quantities, drafts a Purchase Order, and triggers **WF3**.
3. **WF3** sends an interactive Telegram message to the Procurement Officer. If approved, the PO is finalized.
4. **WF4** listens for delivery events via webhook, updates the master inventory, and notifies stakeholders via Telegram.
5. **WF5** runs weekly, aggregating inventory data, generating AI insights, and sending an email report.
6. **WF6** serves as a centralized error-handling and audit-logging webhook that other workflows trigger upon failure.

---

## 3. Implementation Details

This project exceeds the requirements by implementing **6 independent workflows** and **42 nodes in total**.

### WF1: Inventory Monitoring & Stock Updates (6 nodes)
- **Trigger:** Cron Schedule (Runs daily).
- **Function:** Reads the master inventory sheet and filters for items where `current_stock` is less than or equal to `reorder_point`.
- **Advanced Features:** Scheduled triggers (Cron), Conditional branching (IF node).

### WF2: Low Stock Detection & Purchase Request (8 nodes)
- **Trigger:** Webhook (Triggered by WF1).
- **Function:** Receives low stock items, loops through them (Item Lists), queries the Supplier Registry, and uses an AI model to calculate optimal reorder quantities based on supplier delivery averages.
- **Advanced Features:** AI-powered decision making (LangChain + NVIDIA Nemotron), Loops (Item Lists), Database Read/Write.

### WF3: Supplier Communication & Approval (8 nodes)
- **Trigger:** Webhook (Triggered by WF2).
- **Function:** Evaluates PO value. If over $5,000, it halts and sends an interactive Telegram message with "Approve" and "Reject" buttons.
- **Advanced Features:** Human approval step (Wait node), Instant messaging integration, Conditional routing.

### WF4: Delivery Tracking & Inventory Update (7 nodes)
- **Trigger:** Webhook (Triggered by external delivery system or form).
- **Function:** Looks up the PO, verifies the quantity matches, reads the current stock, calculates the new stock, updates the Inventory Sheet, and logs the delivery.
- **Advanced Features:** Data validation, multiple database read/write operations.

### WF5: Inventory Analytics & Reports (6 nodes)
- **Trigger:** Cron Schedule (Runs weekly).
- **Function:** Aggregates inventory metrics (low stock vs. overstock counts), sends the raw data into an LLM chain to generate business insights, and emails the report to management.
- **Advanced Features:** AI-powered insight generation, Email integration (Gmail).

### WF6: Audit Trail & Error Handling (7 nodes)
- **Trigger:** Webhook / Error Trigger.
- **Function:** A master utility workflow. It listens for standard audit logs from other workflows OR catches workflow errors globally. It writes these to an Audit_Trail sheet and alerts the admin via Telegram if it's a critical error.
- **Advanced Features:** Error handling and retry logic (Error Trigger), Logging and audit trail.

  
## Author 
Apoorva Yadav
