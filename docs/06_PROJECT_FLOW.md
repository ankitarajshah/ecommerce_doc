# 06. Project Flow & Workflows
## Enterprise Men's Clothing Manufacturing & Sales Platform

**Version:** 2.0  
**Last Updated:** March 26, 2026  
**Status:** ✅ Production-Ready

---

## 📋 Table of Contents

1. [Order Fulfillment Flow](#1-order-fulfillment-flow)
2. [Manufacturing Workflow](#2-manufacturing-workflow)
3. [Inventory Management Flow](#3-inventory-management-flow)
4. [Marketplace Integration Flow](#4-marketplace-integration-flow)
5. [Return & Refund Process](#5-return--refund-process)
6. [Agent Commission Flow](#6-agent-commission-flow)
7. [CRM Workflow](#7-crm-workflow)

---

## 1. Order Fulfillment Flow

### 1.1 Complete Order Lifecycle

```
┌──────────────────────────────────────────────────────────────────┐
│                     ORDER FULFILLMENT FLOW                        │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────┐
│  Customer       │
│  Places Order   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      1. ORDER CREATION                           │
├─────────────────────────────────────────────────────────────────┤
│  • Validate cart items                                           │
│  • Check inventory availability                                  │
│  • Calculate pricing (retail/wholesale/agent)                    │
│  • Apply discounts & promotions                                  │
│  • Calculate tax (GST 18%)                                       │
│  • Add shipping cost                                             │
│  • Reserve inventory                                             │
│  • Generate order number                                         │
│  • Create order record (status: pending)                         │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      2. PAYMENT PROCESSING                       │
├─────────────────────────────────────────────────────────────────┤
│  [Payment Gateway: Razorpay]                                     │
│                                                                   │
│  ┌──────────────┐         ┌──────────────┐                     │
│  │   Credit/    │         │     UPI      │                     │
│  │  Debit Card  │         │   Payment    │                     │
│  └──────┬───────┘         └──────┬───────┘                     │
│         │                         │                              │
│         └────────┬────────────────┘                             │
│                  │                                               │
│                  ▼                                               │
│         ┌─────────────────┐                                     │
│         │  Razorpay API   │                                     │
│         └────────┬────────┘                                     │
│                  │                                               │
│         ┌────────┴────────┐                                     │
│         │                  │                                     │
│      Success            Failed                                   │
│         │                  │                                     │
│         ▼                  ▼                                     │
│   Update Order      Send Payment                                │
│   (status:          Failed Email                                │
│   confirmed)        (status: failed)                            │
│                                                                   │
│  • Create payment record                                         │
│  • Generate invoice                                              │
│  • Update order status                                           │
│  • Trigger order confirmation email                              │
│  • Create ledger entry                                           │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    3. ORDER PROCESSING                           │
├─────────────────────────────────────────────────────────────────┤
│  • Queue order processing job (Graphile Worker)                  │
│  • Verify inventory availability                                 │
│  • Allocate warehouse & bin location                             │
│  • Generate pick list                                            │
│  • Update order status (processing)                              │
│  • Send processing notification                                  │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      4. WAREHOUSE OPERATIONS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Step 4.1: PICKING                                        │  │
│  │  • Warehouse staff scans pick list                        │  │
│  │  • Navigate to bin locations                              │  │
│  │  • Scan item barcodes                                     │  │
│  │  • Verify quantities                                      │  │
│  │  • Mark items as picked                                   │  │
│  │  • Update inventory (reduce stock)                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Step 4.2: QUALITY CHECK                                  │  │
│  │  • Inspect product condition                              │  │
│  │  • Check for defects                                      │  │
│  │  • Verify correct items                                   │  │
│  │  • Take photos if needed                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Step 4.3: PACKING                                        │  │
│  │  • Select appropriate box size                            │  │
│  │  • Pack items securely                                    │  │
│  │  • Add invoice & thank you note                           │  │
│  │  • Seal package                                           │  │
│  │  • Generate shipping label                                │  │
│  │  • Apply barcode                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Step 4.4: HANDOVER                                       │  │
│  │  • Create shipment record                                 │  │
│  │  • Update order status (shipped)                          │  │
│  │  • Assign tracking number                                 │  │
│  │  • Handover to courier                                    │  │
│  │  • Send shipping notification                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                       5. SHIPPING & TRACKING                     │
├─────────────────────────────────────────────────────────────────┤
│  [Courier Partners: Delhivery, Blue Dart, DTDC]                 │
│                                                                   │
│  In Transit ──► Out for Delivery ──► Delivered                  │
│      │                │                    │                     │
│      ▼                ▼                    ▼                     │
│   SMS/Email      SMS/Email            SMS/Email                 │
│   Notification   Notification         Notification              │
│                                                                   │
│  • Track shipment status via API                                │
│  • Update order status automatically                             │
│  • Send real-time notifications                                  │
│  • Calculate estimated delivery                                  │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                        6. DELIVERY                               │
├─────────────────────────────────────────────────────────────────┤
│  • Customer receives package                                     │
│  • Courier updates delivery status                               │
│  • System updates order (status: delivered)                      │
│  • Trigger post-delivery workflow:                               │
│    - Send review request email (after 2 days)                   │
│    - Award loyalty points                                        │
│    - Update customer stats                                       │
│    - Calculate agent commission                                  │
│    - Mark invoice as completed                                   │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                     7. POST-DELIVERY                             │
├─────────────────────────────────────────────────────────────────┤
│  • Request product review                                        │
│  • Track customer satisfaction                                   │
│  • Handle returns if needed                                      │
│  • Update analytics                                              │
│  • Generate commission for agent                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Order State Transitions

```
┌─────────┐    Payment    ┌───────────┐    Processing   ┌────────────┐
│ Pending │──────────────►│ Confirmed │────────────────►│ Processing │
└─────────┘   Successful   └───────────┘                 └────────────┘
     │                                                          │
     │ Payment Failed                                          │
     ▼                                                          │
┌─────────┐                                                     │
│ Failed  │                                                     │
└─────────┘                                                     │
                                                                │
                                                                ▼
                                                         ┌────────────┐
                                                         │   Shipped  │
                                                         └────────────┘
                                                                │
                                                                │
                                                                ▼
                                                         ┌────────────┐
                                                         │ Delivered  │
                                                         └────────────┘

Cancellation Flow:
Any State ──► Cancel Request ──► Refund Initiated ──► Cancelled
```

---

## 2. Manufacturing Workflow

### 2.1 Production Planning to Finished Goods

```
┌──────────────────────────────────────────────────────────────────┐
│                    MANUFACTURING WORKFLOW                         │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────┐
│  Sales Forecast │
│  & Demand       │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  1. PRODUCTION PLANNING                          │
├─────────────────────────────────────────────────────────────────┤
│  • Analyze sales trends                                          │
│  • Identify low stock items                                      │
│  • Check seasonal demand                                         │
│  • Review marketplace orders                                     │
│  • Create production schedule                                    │
│  • Define target quantities                                      │
│                                                                   │
│  Output: Production Plan for next 30 days                        │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  2. MATERIAL REQUIREMENT PLANNING                │
├─────────────────────────────────────────────────────────────────┤
│  • Review Bill of Materials (BOM)                                │
│  • Calculate raw material requirements                           │
│  • Check current raw material inventory                          │
│  • Identify material shortage                                    │
│  • Generate purchase requirements                                │
│                                                                   │
│  Example BOM (Premium Formal Shirt):                             │
│  - Fabric: 1.5 meters                                           │
│  - Buttons: 10 pieces                                            │
│  - Thread: 50 meters                                             │
│  - Labels: 2 pieces                                              │
│  - Packaging: 1 poly bag                                         │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  3. PROCUREMENT                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────────────────────────────┐                │
│  │  3.1: Create Purchase Order (PO)           │                │
│  │  • Select supplier                          │                │
│  │  • Add required materials                   │                │
│  │  • Negotiate pricing                        │                │
│  │  • Set delivery date                        │                │
│  │  • Generate PO document                     │                │
│  └────────────────┬───────────────────────────┘                │
│                   │                                              │
│                   ▼                                              │
│  ┌────────────────────────────────────────────┐                │
│  │  3.2: Approval Workflow                    │                │
│  │  • Purchase Manager: Review                │                │
│  │  • Finance: Budget Check                   │                │
│  │  • Senior Manager: Final Approval          │                │
│  └────────────────┬───────────────────────────┘                │
│                   │                                              │
│                   ▼                                              │
│  ┌────────────────────────────────────────────┐                │
│  │  3.3: Send to Supplier                     │                │
│  │  • Email PO to supplier                    │                │
│  │  • Track acknowledgment                    │                │
│  │  • Set delivery reminders                  │                │
│  └────────────────┬───────────────────────────┘                │
│                   │                                              │
│                   ▼                                              │
│  ┌────────────────────────────────────────────┐                │
│  │  3.4: Material Receipt                     │                │
│  │  • Verify delivery against PO              │                │
│  │  • Quality inspection                      │                │
│  │  • Update inventory                        │                │
│  │  • Create goods receipt note               │                │
│  │  • Process payment                         │                │
│  └────────────────────────────────────────────┘                │
│                                                                   │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  4. WORK ORDER CREATION                          │
├─────────────────────────────────────────────────────────────────┤
│  • Select product & variant                                      │
│  • Define production quantity                                    │
│  • Set priority (high/medium/low)                               │
│  • Assign start & due dates                                      │
│  • Reserve raw materials                                         │
│  • Generate work order number                                    │
│                                                                   │
│  Work Order: WO-2026-0001                                        │
│  Product: Premium Formal Shirt (Blue, Size M)                   │
│  Quantity: 100 units                                             │
│  Start: 2026-03-27                                               │
│  Due: 2026-04-10                                                 │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  5. PRODUCTION STAGES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Stage 1: CUTTING (2 days)                                │  │
│  │  • Cut fabric as per pattern                             │  │
│  │  • Separate pieces for each component                    │  │
│  │  • Mark quantities                                        │  │
│  │  Progress: 100 units cut                                 │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Stage 2: STITCHING (5 days)                             │  │
│  │  • Sew collar, sleeves, body                             │  │
│  │  • Attach buttons & buttonholes                          │  │
│  │  • Create pockets                                        │  │
│  │  • Join all components                                   │  │
│  │  Progress: 85 units stitched                             │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Stage 3: QUALITY CHECK (1 day)                          │  │
│  │  • Inspect stitching quality                             │  │
│  │  • Check measurements                                    │  │
│  │  • Verify button placement                               │  │
│  │  • Test wash durability                                  │  │
│  │  • Mark defects                                          │  │
│  │  Pass: 82 units, Reject: 3 units, Rework: 15 units      │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Stage 4: FINISHING (2 days)                             │  │
│  │  • Iron and press                                        │  │
│  │  • Attach brand labels                                   │  │
│  │  • Add care instruction tags                             │  │
│  │  • Fold properly                                         │  │
│  │  Progress: 97 units finished                             │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Stage 5: PACKAGING (1 day)                              │  │
│  │  • Place in poly bags                                    │  │
│  │  • Add product tags                                      │  │
│  │  • Pack in cartons                                       │  │
│  │  • Apply barcodes                                        │  │
│  │  • Update inventory                                      │  │
│  │  Complete: 97 units ready                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  6. INVENTORY UPDATE                             │
├─────────────────────────────────────────────────────────────────┤
│  • Move finished goods to warehouse                              │
│  • Assign bin locations                                          │
│  • Update product inventory (+97 units)                         │
│  • Update raw material inventory (consumed)                     │
│  • Calculate production cost                                     │
│  • Close work order                                              │
│  • Generate production report                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Defect Management

```
Defect Found
     │
     ▼
Classify: Minor / Major / Critical
     │
     ├──► Minor: Send to Rework
     │         │
     │         ▼
     │    Fix & Re-inspect ──► Pass ──► Continue Production
     │
     ├──► Major: Detailed Analysis
     │         │
     │         ▼
     │    Root Cause Analysis ──► Implement Fix ──► Rework
     │
     └──► Critical: Reject
               │
               ▼
          Record Loss ──► Dispose ──► Adjust Inventory
```

---

## 3. Inventory Management Flow

### 3.1 Stock Movement Workflow

```
┌──────────────────────────────────────────────────────────────────┐
│                    INVENTORY MANAGEMENT                           │
└──────────────────────────────────────────────────────────────────┘

STOCK IN OPERATIONS:
┌─────────────────────────────────────────────────────────────────┐
│  1. Purchase Receipt                                             │
│     • Verify against PO                                          │
│     • Quality check                                              │
│     • Update inventory (+quantity)                               │
│                                                                   │
│  2. Manufacturing Output                                         │
│     • Receive from production                                    │
│     • Quality check                                              │
│     • Update inventory (+quantity)                               │
│                                                                   │
│  3. Return from Customer                                         │
│     • Inspect condition                                          │
│     • Decide: Restock / Dispose                                 │
│     • Update inventory (if restocking)                           │
│                                                                   │
│  4. Transfer In (from other warehouse)                           │
│     • Scan barcode                                               │
│     • Verify quantity                                            │
│     • Update inventory (+quantity)                               │
└─────────────────────────────────────────────────────────────────┘

STOCK OUT OPERATIONS:
┌─────────────────────────────────────────────────────────────────┐
│  1. Order Fulfillment                                            │
│     • Pick items                                                 │
│     • Update inventory (-quantity)                               │
│     • Record movement                                            │
│                                                                   │
│  2. Manufacturing Consumption                                    │
│     • Issue materials to production                              │
│     • Update inventory (-quantity)                               │
│     • Link to work order                                         │
│                                                                   │
│  3. Damaged/Lost                                                 │
│     • Report damage                                              │
│     • Get approval                                               │
│     • Write off inventory (-quantity)                            │
│                                                                   │
│  4. Transfer Out (to other warehouse)                            │
│     • Create transfer order                                      │
│     • Pick & pack items                                          │
│     • Update inventory (-quantity)                               │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Low Stock Alert Flow

```
Cron Job (Every 2 hours)
     │
     ▼
Check All Products
     │
     ▼
Quantity <= Reorder Point?
     │
     ├──► NO ──► Skip
     │
     ▼ YES
Generate Alert
     │
     ▼
Send Email to Inventory Manager
     │
     ▼
Create Notification in Dashboard
     │
     ▼
Suggest: Create Purchase Order
```

---

## 4. Marketplace Integration Flow

### 4.1 Product Sync to Marketplace

```
┌──────────────────────────────────────────────────────────────────┐
│              MARKETPLACE INTEGRATION WORKFLOW                     │
└──────────────────────────────────────────────────────────────────┘

Product Created/Updated in System
     │
     ▼
Check Auto-Sync Setting
     │
     ├──► Enabled ──► Proceed
     │
     ▼ Disabled
Manual Sync Triggered
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Prepare Product Data                                            │
│  • Map fields to marketplace format                              │
│  • Optimize images (resize, compress)                            │
│  • Generate marketplace-specific description                     │
│  • Set pricing rules                                             │
│  • Calculate shipping dimensions                                 │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Sync to Marketplace APIs                                        │
│                                                                   │
│  Amazon Seller Central ──► Create/Update Listing                │
│  Flipkart API          ──► Create/Update Listing                │
│  Meesho API            ──► Create/Update Listing                │
│                                                                   │
│  Response: Listing ID, Status                                    │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
Store Listing Mapping
     │
     ▼
Update Sync Log (success/failure)
     │
     ▼
Send Notification to Admin
```

### 4.2 Marketplace Order Import

```
Cron Job (Every 30 minutes)
     │
     ▼
Fetch New Orders from Each Marketplace
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  For Each Order:                                                 │
│  1. Map marketplace order to internal format                     │
│  2. Find product by listing ID                                   │
│  3. Create customer (if new)                                     │
│  4. Create internal order                                        │
│  5. Reserve inventory                                            │
│  6. Mark as "marketplace order"                                  │
│  7. Trigger fulfillment workflow                                 │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
Order Fulfilled
     │
     ▼
Update Marketplace Status via API
     │
     ▼
Marketplace marks as "Shipped/Delivered"
```

### 4.3 Inventory Sync

```
Inventory Updated in System
     │
     ▼
Queue Marketplace Sync Job
     │
     ▼
Update Inventory on All Marketplaces
     │
     ├──► Amazon: Update quantity
     ├──► Flipkart: Update quantity
     └──► Meesho: Update quantity
     │
     ▼
Log Sync Result
```

---

## 5. Return & Refund Process

### 5.1 Return Workflow

```
┌──────────────────────────────────────────────────────────────────┐
│                    RETURN & REFUND FLOW                           │
└──────────────────────────────────────────────────────────────────┘

Customer Requests Return
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. Return Request Validation                                    │
│     • Check return window (30 days)                              │
│     • Verify order status (delivered)                            │
│     • Check return policy eligibility                            │
│     • Validate reason                                            │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. Approval Workflow                                            │
│     • Auto-approve if within policy                              │
│     • Manual review if exception                                 │
│     • Customer service approval for high-value items             │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼ Approved
┌─────────────────────────────────────────────────────────────────┐
│  3. Pickup Scheduling                                            │
│     • Generate return ID                                         │
│     • Schedule courier pickup                                    │
│     • Send pickup details to customer                            │
│     • Provide return tracking                                    │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. Product Receipt & Inspection                                 │
│     • Receive returned product                                   │
│     • Verify item condition                                      │
│     • Check for damage                                           │
│     • Match with return request                                  │
│     • Take photos                                                │
└────────┬────────────────────────────────────────────────────────┘
         │
         ├──► Good Condition ──► Restock Inventory
         │
         └──► Damaged ──► Mark as Damaged Stock
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. Refund Processing                                            │
│     • Calculate refund amount                                    │
│     • Deduct shipping (if applicable)                            │
│     • Initiate payment gateway refund                            │
│     • Update order status                                        │
│     • Create credit note                                         │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. Refund Completion                                            │
│     • Customer receives refund (5-7 days)                        │
│     • Send refund confirmation email                             │
│     • Close return request                                       │
│     • Update customer lifetime value                             │
└─────────────────────────────────────────────────────────────────┘

REFUND TIMELINES:
• Prepaid Orders: 5-7 business days to original payment method
• COD Orders: Bank transfer within 7-10 business days
• Wallet: Instant credit to customer wallet
```

---

## 6. Agent Commission Flow

### 6.1 Commission Calculation & Payout

```
┌──────────────────────────────────────────────────────────────────┐
│                  AGENT COMMISSION WORKFLOW                        │
└──────────────────────────────────────────────────────────────────┘

Order Delivered Successfully
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. Commission Eligibility Check                                 │
│     • Order placed by agent's customer                           │
│     • Order status: delivered                                    │
│     • No return initiated                                        │
│     • Payment received                                           │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. Commission Calculation                                       │
│                                                                   │
│  Commission Rate Structure:                                      │
│  • Base Rate: 5% of order value                                 │
│  • Tier Bonuses:                                                 │
│    - Bronze (<₹50K/month): 5%                                   │
│    - Silver (₹50K-₹1L/month): 6%                                │
│    - Gold (₹1L-₹2L/month): 7%                                   │
│    - Platinum (>₹2L/month): 8%                                  │
│  • Product Category Multiplier:                                  │
│    - Premium Products: +1%                                       │
│    - Clearance Items: -1%                                        │
│                                                                   │
│  Example:                                                        │
│  Order Value: ₹5,000                                             │
│  Agent Tier: Silver (6%)                                         │
│  Product: Premium Shirt (+1%)                                    │
│  Commission: ₹5,000 × 7% = ₹350                                 │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. Commission Record Creation                                   │
│     • Create commission entry                                    │
│     • Link to order                                              │
│     • Link to agent                                              │
│     • Status: pending                                            │
│     • Lock-in period: 15 days (return window)                   │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
Wait 15 Days (Return Window)
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. Commission Approval                                          │
│     • Verify no returns                                          │
│     • Verify payment received                                    │
│     • Update status: approved                                    │
│     • Add to monthly payout batch                                │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. Monthly Payout (1st of every month)                          │
│     • Calculate total commission per agent                       │
│     • Deduct TDS (10%)                                           │
│     • Generate payout report                                     │
│     • Initiate bank transfer                                     │
│     • Send payout statement                                      │
│     • Update commission status: paid                             │
└────────┬────────────────────────────────────────────────────────┘
         │
         ▼
Agent Receives Payment
```

### 6.2 Commission Dashboard Metrics

```yaml
Agent Commission Dashboard:
  Current Month:
    - Total Sales: ₹1,50,000
    - Commission Earned: ₹10,500
    - Orders: 35
    - Conversion Rate: 28%
  
  Pending Commission:
    - Amount: ₹5,200
    - Orders in Lock-in: 12
    - Available After: 5 days
  
  Paid Commission:
    - This Month: ₹8,500
    - Last Month: ₹10,200
    - Total Lifetime: ₹2,45,000
  
  Performance:
    - Current Tier: Silver
    - Next Tier: Gold (₹15,000 more needed)
    - Top Products: Formal Shirts, Trousers
```

---

## 7. CRM Workflow

### 7.1 Customer Lifecycle Management

```
┌──────────────────────────────────────────────────────────────────┐
│                    CUSTOMER LIFECYCLE FLOW                        │
└──────────────────────────────────────────────────────────────────┘

┌─────────────────┐
│  Lead/Prospect  │  ──► Marketing campaigns, Landing pages
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Registration   │  ──► Welcome email, Onboarding tutorial
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  First Purchase │  ──► Thank you email, Loyalty points
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Active Customer│  ──► Personalized recommendations
└────────┬────────┘      ──► Exclusive offers
         │                ──► Birthday discounts
         │
         ├──► Regular Buyer (>5 orders) ──► VIP Segment
         │                                   ──► Priority support
         │                                   ──► Early access
         │
         └──► Inactive (>60 days) ──► Re-engagement campaign
                                      ──► Win-back offers
                                      ──► Feedback survey
```

### 7.2 Support Ticket Workflow

```
Customer Raises Issue
     │
     ▼
Create Ticket
     │
     ▼
Auto-Assign Based on Category
     │
     ├──► Product Issue ──► Quality Team
     ├──► Order Issue ──► Operations Team
     ├──► Payment Issue ──► Finance Team
     └──► General Query ──► Customer Service
     │
     ▼
Agent Responds
     │
     ▼
Resolution Process
     │
     ├──► Simple: Immediate resolution
     │
     └──► Complex: Escalate to Senior Agent
                      │
                      ▼
                 Investigate & Resolve
     │
     ▼
Customer Confirms Resolution
     │
     ▼
Close Ticket
     │
     ▼
Request Feedback Rating
```

---

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Next Review:** April 26, 2026  
**Status:** ✅ Production-Ready

---

[← Back to Index](./00_MASTER_INDEX.md) | [Next: SOP Engine →](./07_SOP_ENGINE.md)
