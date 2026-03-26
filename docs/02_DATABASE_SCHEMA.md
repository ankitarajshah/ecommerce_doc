# 02. Database Schema
## Enterprise Men's Clothing Manufacturing & Sales Platform

**Version:** 2.0  
**Last Updated:** March 26, 2026  
**Database:** PostgreSQL 15+  
**Status:** ✅ Production-Ready

---

## 📋 Table of Contents

1. [Database Overview](#database-overview)
2. [Entity Relationship Diagram](#entity-relationship-diagram)
3. [Schema Details](#schema-details)
4. [Indexing Strategy](#indexing-strategy)
5. [Data Integrity Rules](#data-integrity-rules)
6. [Performance Optimization](#performance-optimization)

---

## 1. Database Overview

### 1.1 Database Configuration

```yaml
Database Engine: PostgreSQL 15.x
Character Set: UTF8
Collation: en_US.UTF-8
Timezone: UTC
Max Connections: 200
Shared Buffers: 25% of RAM
Effective Cache Size: 75% of RAM
Work Memory: 16MB
Maintenance Work Memory: 256MB
```

### 1.2 Schema Organization

```
ecomm_db
├── public (default schema)
│   ├── Core Tables (users, roles, permissions)
│   ├── Product Tables (products, categories, variants)
│   ├── Order Tables (orders, order_items, shipments)
│   ├── Inventory Tables (inventory, stock_movements)
│   ├── Manufacturing Tables (work_orders, bom, suppliers)
│   ├── Financial Tables (payments, invoices, ledger)
│   └── System Tables (audit_logs, settings)
├── analytics (analytics schema)
│   ├── Materialized Views
│   └── Aggregated Data
└── archive (historical data)
    └── Archived Tables (>2 years old)
```

---

## 2. Entity Relationship Diagram

### 2.1 Complete ER Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AUTHENTICATION & AUTHORIZATION                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│      users       │              │      roles       │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│    email         │◄────┐        │    name          │
│    password_hash │     │        │    description   │
│    phone         │     │        │    created_at    │
│    first_name    │     │        └────────┬─────────┘
│    last_name     │     │                 │
│    status        │     │                 │ Many-to-Many
│    email_verified│     │                 │
│    created_at    │     │        ┌────────▼─────────┐
│    updated_at    │     │        │   user_roles     │
└────────┬─────────┘     │        ├──────────────────┤
         │               │        │ PK id            │
         │               └────────┤ FK user_id       │
         │                        │ FK role_id       │
         │ One-to-Many            │    created_at    │
         │                        └────────┬─────────┘
┌────────▼─────────┐                       │
│    sessions      │                       │ Many-to-Many
├──────────────────┤                       │
│ PK id            │              ┌────────▼─────────┐
│ FK user_id       │              │  permissions     │
│    token         │              ├──────────────────┤
│    device_info   │              │ PK id            │
│    ip_address    │              │    resource      │
│    expires_at    │              │    action        │
│    created_at    │              │    description   │
└──────────────────┘              └────────┬─────────┘
                                           │
                                           │ Many-to-Many
                                  ┌────────▼──────────┐
                                  │ role_permissions  │
                                  ├───────────────────┤
                                  │ PK id             │
                                  │ FK role_id        │
                                  │ FK permission_id  │
                                  │    created_at     │
                                  └───────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          PRODUCT CATALOG SYSTEM                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│   categories     │              │    products      │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│ FK parent_id     │◄──┐          │ FK category_id   │◄──┐
│    name          │   │          │    sku           │   │
│    slug          │   │ Self     │    name          │   │
│    description   │   │ Ref      │    slug          │   │
│    image_url     │   │          │    description   │   │
│    sort_order    │   │          │    brand         │   │
│    is_active     │   │          │    season        │   │
│    created_at    │   │          │    status        │   │
└────────┬─────────┘   │          │    created_at    │   │
         │              │          │    updated_at    │   │
         └──────────────┘          └────────┬─────────┘   │
                                            │             │
                                            │ One-to-Many │
                                   ┌────────▼─────────┐   │
                                   │product_variants  │   │
                                   ├──────────────────┤   │
                                   │ PK id            │   │
                                   │ FK product_id    │───┘
                                   │    sku           │
                                   │    size          │
                                   │    color         │
                                   │    material      │
                                   │    fit           │
                                   │    retail_price  │
                                   │    wholesale_price│
                                   │    cost_price    │
                                   │    barcode       │
                                   │    is_active     │
                                   │    created_at    │
                                   └────────┬─────────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    │                       │                       │
                    │ One-to-Many           │ One-to-Many           │
         ┌──────────▼─────────┐  ┌─────────▼──────────┐ ┌─────────▼──────────┐
         │  product_images    │  │product_attributes  │ │ pricing_rules      │
         ├────────────────────┤  ├────────────────────┤ ├────────────────────┤
         │ PK id              │  │ PK id              │ │ PK id              │
         │ FK product_id      │  │ FK product_id      │ │ FK variant_id      │
         │    url             │  │    attribute_name  │ │    customer_type   │
         │    alt_text        │  │    attribute_value │ │    min_quantity    │
         │    sort_order      │  │    created_at      │ │    discount_pct    │
         │    is_primary      │  └────────────────────┘ │    start_date      │
         │    created_at      │                         │    end_date        │
         └────────────────────┘                         │    is_active       │
                                                        └────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           INVENTORY MANAGEMENT                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│   warehouses     │              │  bin_locations   │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│    code          │──────────────┤ FK warehouse_id  │
│    name          │ One-to-Many  │    zone          │
│    type          │              │    aisle         │
│    address       │              │    rack          │
│    city          │              │    bin           │
│    state         │              │    capacity      │
│    country       │              │    is_active     │
│    pincode       │              │    created_at    │
│    is_active     │              └────────┬─────────┘
│    created_at    │                       │
└────────┬─────────┘                       │
         │                                 │ Many-to-One
         │ One-to-Many                     │
┌────────▼─────────┐              ┌───────▼──────────┐
│    inventory     │              │                  │
├──────────────────┤              │                  │
│ PK id            │              │                  │
│ FK variant_id    │◄─────────────┤ FK variant_id    │
│ FK warehouse_id  │              │                  │
│ FK bin_loc_id    │◄─────────────┘                  │
│    quantity      │                                  │
│    reserved_qty  │                                  │
│    available_qty │  (computed: quantity - reserved_qty)
│    reorder_point │                                  │
│    reorder_qty   │                                  │
│    updated_at    │                                  │
└────────┬─────────┘                                  │
         │                                            │
         │ One-to-Many                                │
┌────────▼─────────────┐                             │
│  stock_movements     │                             │
├──────────────────────┤                             │
│ PK id                │                             │
│ FK variant_id        │◄────────────────────────────┘
│ FK from_warehouse_id │
│ FK to_warehouse_id   │
│ FK created_by        │
│    movement_type     │ (IN, OUT, TRANSFER, ADJUSTMENT)
│    quantity          │
│    reference_type    │ (ORDER, RETURN, PRODUCTION, etc)
│    reference_id      │
│    notes             │
│    created_at        │
└──────────────────────┘

┌──────────────────┐
│     batches      │
├──────────────────┤
│ PK id            │
│ FK variant_id    │
│    batch_number  │
│    mfg_date      │
│    expiry_date   │
│    quantity      │
│    status        │
│    created_at    │
└──────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         MANUFACTURING SYSTEM                                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│    suppliers     │              │  raw_materials   │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│    name          │──────────────┤ FK supplier_id   │
│    code          │ One-to-Many  │    name          │
│    contact_person│              │    sku           │
│    email         │              │    category      │
│    phone         │              │    unit          │
│    address       │              │    unit_cost     │
│    city          │              │    stock_qty     │
│    gst_number    │              │    reorder_point │
│    payment_terms │              │    created_at    │
│    rating        │              └────────┬─────────┘
│    is_active     │                       │
│    created_at    │                       │ Many-to-One
└────────┬─────────┘                       │
         │                                 │
         │ One-to-Many            ┌────────▼─────────┐
┌────────▼─────────────┐          │       bom        │
│  purchase_orders     │          ├──────────────────┤
├──────────────────────┤          │ PK id            │
│ PK id                │          │ FK product_id    │
│ FK supplier_id       │          │ FK material_id   │
│ FK created_by        │          │    quantity_req  │
│    po_number         │          │    wastage_pct   │
│    po_date           │          │    unit          │
│    expected_delivery │          │    created_at    │
│    total_amount      │          └────────┬─────────┘
│    status            │                   │
│    approved_by       │                   │ Many-to-One
│    approved_at       │                   │
│    notes             │          ┌────────▼─────────┐
│    created_at        │          │   work_orders    │
└────────┬─────────────┘          ├──────────────────┤
         │                        │ PK id            │
         │ One-to-Many            │ FK product_id    │
┌────────▼──────────────┐         │ FK created_by    │
│  po_items            │         │    wo_number     │
├──────────────────────┤         │    quantity      │
│ PK id                │         │    start_date    │
│ FK po_id             │         │    end_date      │
│ FK material_id       │         │    priority      │
│    quantity          │         │    status        │
│    unit_price        │         │    total_cost    │
│    received_qty      │         │    notes         │
│    status            │         │    created_at    │
│    created_at        │         └────────┬─────────┘
└──────────────────────┘                  │
                                          │ One-to-Many
                                 ┌────────▼─────────────┐
                                 │ production_stages    │
                                 ├──────────────────────┤
                                 │ PK id                │
                                 │ FK work_order_id     │
                                 │ FK assigned_to       │
                                 │    stage_name        │
                                 │    sequence          │
                                 │    status            │
                                 │    started_at        │
                                 │    completed_at      │
                                 │    notes             │
                                 │    created_at        │
                                 └────────┬─────────────┘
                                          │
                                          │ One-to-Many
                                 ┌────────▼─────────────┐
                                 │  quality_checks      │
                                 ├──────────────────────┤
                                 │ PK id                │
                                 │ FK stage_id          │
                                 │ FK checked_by        │
                                 │    check_type        │
                                 │    result            │
                                 │    defect_count      │
                                 │    defect_type       │
                                 │    severity          │
                                 │    notes             │
                                 │    created_at        │
                                 └──────────────────────┘

┌──────────────────┐
│     defects      │
├──────────────────┤
│ PK id            │
│ FK wo_id         │
│ FK stage_id      │
│    defect_type   │
│    quantity      │
│    severity      │
│    reported_by   │
│    action_taken  │
│    created_at    │
└──────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           ORDER MANAGEMENT                                   │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│    customers     │              │      orders      │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│ FK user_id       │──────────────┤ FK customer_id   │
│    customer_type │ One-to-Many  │ FK agent_id      │
│    company_name  │              │    order_number  │
│    gst_number    │              │    channel       │
│    credit_limit  │              │    order_type    │
│    credit_used   │              │    status        │
│    segment       │              │    subtotal      │
│    lifetime_value│              │    tax           │
│    created_at    │              │    shipping_cost │
└──────────────────┘              │    discount      │
                                  │    total_amount  │
┌──────────────────┐              │    payment_status│
│      agents      │              │    currency      │
├──────────────────┤              │    notes         │
│ PK id            │              │    created_at    │
│ FK user_id       │──────────────┤    updated_at    │
│    agent_code    │ One-to-Many  └────────┬─────────┘
│    territory     │                       │
│    commission_pct│                       │ One-to-Many
│    target_amount │              ┌────────▼─────────┐
│    current_sales │              │   order_items    │
│    status        │              ├──────────────────┤
│    created_at    │              │ PK id            │
└──────────────────┘              │ FK order_id      │
                                  │ FK variant_id    │
         ┌────────────────────────┤    quantity      │
         │                        │    unit_price    │
         │ One-to-Many            │    discount      │
┌────────▼──────────────┐         │    tax           │
│  order_addresses      │         │    total         │
├───────────────────────┤         │    created_at    │
│ PK id                 │         └────────┬─────────┘
│ FK order_id           │                  │
│    address_type       │ (BILLING/SHIPPING)
│    first_name         │                  │ One-to-Many
│    last_name          │         ┌────────▼─────────────┐
│    phone              │         │ order_status_history │
│    address_line1      │         ├──────────────────────┤
│    address_line2      │         │ PK id                │
│    city               │         │ FK order_id          │
│    state              │         │ FK changed_by        │
│    country            │         │    status            │
│    pincode            │         │    notes             │
│    created_at         │         │    created_at        │
└───────────────────────┘         └──────────────────────┘
                                            │
                                            │ Many-to-One
         ┌──────────────────────────────────┤
         │                                  │
┌────────▼─────────┐              ┌─────────▼─────────┐
│    shipments     │              │     returns       │
├──────────────────┤              ├───────────────────┤
│ PK id            │              │ PK id             │
│ FK order_id      │              │ FK order_id       │
│ FK warehouse_id  │              │ FK customer_id    │
│    tracking_no   │              │    return_number  │
│    carrier       │              │    reason         │
│    carrier_code  │              │    status         │
│    shipped_at    │              │    refund_amount  │
│    delivered_at  │              │    refund_method  │
│    status        │              │    approved_by    │
│    label_url     │              │    approved_at    │
│    notes         │              │    notes          │
│    created_at    │              │    created_at     │
└──────────────────┘              └───────┬───────────┘
                                          │
                                          │ One-to-Many
                                  ┌───────▼───────────┐
                                  │  return_items     │
                                  ├───────────────────┤
                                  │ PK id             │
                                  │ FK return_id      │
                                  │ FK order_item_id  │
                                  │    quantity       │
                                  │    condition      │
                                  │    action         │
                                  │    created_at     │
                                  └───────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         PAYMENT & FINANCE                                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│     payments     │              │     invoices     │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│ FK order_id      │◄─────────────┤ FK order_id      │
│ FK customer_id   │ One-to-One   │ FK customer_id   │
│    payment_id    │              │    invoice_number│
│    gateway       │              │    invoice_date  │
│    method        │              │    due_date      │
│    amount        │              │    subtotal      │
│    currency      │              │    tax           │
│    status        │              │    discount      │
│    transaction_id│              │    total_amount  │
│    response_data │              │    status        │
│    created_at    │              │    paid_at       │
└────────┬─────────┘              │    created_at    │
         │                        └────────┬─────────┘
         │ One-to-Many                     │
┌────────▼─────────┐                       │ One-to-Many
│     refunds      │              ┌────────▼─────────┐
├──────────────────┤              │  invoice_items   │
│ PK id            │              ├──────────────────┤
│ FK payment_id    │              │ PK id            │
│ FK order_id      │              │ FK invoice_id    │
│    refund_id     │              │    description   │
│    amount        │              │    quantity      │
│    reason        │              │    unit_price    │
│    status        │              │    tax           │
│    processed_at  │              │    total         │
│    created_at    │              │    created_at    │
└──────────────────┘              └──────────────────┘

┌──────────────────┐              ┌──────────────────┐
│      ledger      │              │     expenses     │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│ FK customer_id   │              │    category      │
│ FK reference_id  │              │    description   │
│    entry_type    │ (DEBIT/CREDIT)│    amount       │
│    amount        │              │    expense_date  │
│    balance       │              │    vendor        │
│    description   │              │    payment_method│
│    created_at    │              │    receipt_url   │
└──────────────────┘              │    approved_by   │
                                  │    created_at    │
                                  └──────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                       CRM & CUSTOMER MANAGEMENT                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│customer_segments │              │customer_seg_map  │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│    name          │──────────────┤ FK customer_id   │
│    description   │ One-to-Many  │ FK segment_id    │
│    criteria      │ (JSON)       │    assigned_at   │
│    is_active     │              └──────────────────┘
│    created_at    │
└──────────────────┘

┌──────────────────┐              ┌──────────────────┐
│     tickets      │              │  ticket_messages │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│ FK customer_id   │──────────────┤ FK ticket_id     │
│ FK assigned_to   │ One-to-Many  │ FK user_id       │
│    ticket_number │              │    message       │
│    subject       │              │    is_internal   │
│    priority      │              │    attachments   │
│    status        │              │    created_at    │
│    category      │              └──────────────────┘
│    created_at    │
│    resolved_at   │
└──────────────────┘

┌──────────────────┐              ┌──────────────────┐
│   commissions    │              │ loyalty_points   │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│ FK agent_id      │              │ FK customer_id   │
│ FK order_id      │              │    points        │
│    amount        │              │    earned_from   │
│    percentage    │              │    expires_at    │
│    status        │              │    status        │
│    paid_at       │              │    created_at    │
│    created_at    │              └──────────────────┘
└──────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    MARKETPLACE INTEGRATIONS                                  │
└─────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────┐        ┌──────────────────┐
│marketplace_listings    │        │   sync_logs      │
├────────────────────────┤        ├──────────────────┤
│ PK id                  │        │ PK id            │
│ FK product_id          │────────┤ FK listing_id    │
│    marketplace         │One-Many│    sync_type     │
│    listing_id          │        │    status        │
│    title               │        │    error_message │
│    price               │        │    request_data  │
│    inventory_sync      │        │    response_data │
│    status              │        │    created_at    │
│    last_synced_at      │        └──────────────────┘
│    created_at          │
└────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│    webhooks      │              │  webhook_logs    │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│    url           │──────────────┤ FK webhook_id    │
│    event_type    │ One-to-Many  │    payload       │
│    secret        │              │    response_code │
│    is_active     │              │    response_body │
│    created_at    │              │    retry_count   │
└──────────────────┘              │    created_at    │
                                  └──────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          SYSTEM & AUDIT                                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│   audit_logs     │              │  notifications   │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│ FK user_id       │              │ FK user_id       │
│    action        │              │    type          │
│    resource_type │              │    channel       │
│    resource_id   │              │    title         │
│    old_values    │ (JSON)       │    message       │
│    new_values    │ (JSON)       │    data          │ (JSON)
│    ip_address    │              │    status        │
│    user_agent    │              │    sent_at       │
│    created_at    │              │    read_at       │
└──────────────────┘              │    created_at    │
                                  └──────────────────┘

┌──────────────────┐              ┌──────────────────┐
│     settings     │              │   email_logs     │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│    key           │              │    to_email      │
│    value         │              │    subject       │
│    type          │              │    template      │
│    description   │              │    variables     │ (JSON)
│    is_public     │              │    status        │
│    updated_at    │              │    sent_at       │
└──────────────────┘              │    error         │
                                  │    created_at    │
                                  └──────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          ANALYTICS & REPORTING                               │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐              ┌──────────────────┐
│  daily_metrics   │              │ product_views    │
├──────────────────┤              ├──────────────────┤
│ PK id            │              │ PK id            │
│    date          │              │ FK product_id    │
│    total_orders  │              │ FK user_id       │
│    total_revenue │              │    session_id    │
│    avg_order_val │              │    referrer      │
│    new_customers │              │    duration      │
│    returning     │              │    created_at    │
│    conversion_rt │              └──────────────────┘
│    created_at    │
└──────────────────┘              ┌──────────────────┐
                                  │  search_logs     │
                                  ├──────────────────┤
                                  │ PK id            │
                                  │ FK user_id       │
                                  │    query         │
                                  │    filters       │ (JSON)
                                  │    results_count │
                                  │    clicked_item  │
                                  │    created_at    │
                                  └──────────────────┘
```

---

## 3. Schema Details

### 3.1 Users & Authentication Tables

```sql
-- Users Table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended')),
    email_verified BOOLEAN DEFAULT FALSE,
    email_verified_at TIMESTAMP,
    phone_verified BOOLEAN DEFAULT FALSE,
    last_login_at TIMESTAMP,
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Roles Table
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    is_system BOOLEAN DEFAULT FALSE, -- Cannot be deleted
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Permissions Table
CREATE TABLE permissions (
    id SERIAL PRIMARY KEY,
    resource VARCHAR(50) NOT NULL, -- e.g., 'products', 'orders'
    action VARCHAR(50) NOT NULL,   -- e.g., 'create', 'read', 'update', 'delete'
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(resource, action)
);

-- User Roles Junction Table
CREATE TABLE user_roles (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id INTEGER NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, role_id)
);

-- Role Permissions Junction Table
CREATE TABLE role_permissions (
    id SERIAL PRIMARY KEY,
    role_id INTEGER NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id INTEGER NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(role_id, permission_id)
);

-- Sessions Table
CREATE TABLE sessions (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token VARCHAR(500) UNIQUE NOT NULL,
    refresh_token VARCHAR(500) UNIQUE,
    device_info TEXT,
    ip_address VARCHAR(45),
    user_agent TEXT,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.2 Product Catalog Tables

```sql
-- Categories Table
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    parent_id INTEGER REFERENCES categories(id) ON DELETE SET NULL,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    image_url VARCHAR(500),
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    meta_title VARCHAR(200),
    meta_description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products Table
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    category_id INTEGER REFERENCES categories(id),
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    short_description TEXT,
    brand VARCHAR(100),
    season VARCHAR(50), -- 'summer', 'winter', 'spring', 'fall', 'all-season'
    status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'inactive', 'discontinued')),
    is_featured BOOLEAN DEFAULT FALSE,
    meta_title VARCHAR(200),
    meta_description TEXT,
    meta_keywords TEXT,
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Product Variants Table
CREATE TABLE product_variants (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku VARCHAR(50) UNIQUE NOT NULL,
    size VARCHAR(20), -- 'S', 'M', 'L', 'XL', '32', '34', etc.
    color VARCHAR(50),
    material VARCHAR(100),
    fit VARCHAR(50), -- 'regular', 'slim', 'relaxed'
    retail_price DECIMAL(10, 2) NOT NULL,
    wholesale_price DECIMAL(10, 2),
    cost_price DECIMAL(10, 2),
    compare_at_price DECIMAL(10, 2),
    barcode VARCHAR(100) UNIQUE,
    weight DECIMAL(8, 2), -- in grams
    dimensions VARCHAR(50), -- 'LxWxH in cm'
    is_active BOOLEAN DEFAULT TRUE,
    requires_shipping BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Product Images Table
CREATE TABLE product_images (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    url VARCHAR(500) NOT NULL,
    thumbnail_url VARCHAR(500),
    alt_text VARCHAR(255),
    sort_order INTEGER DEFAULT 0,
    is_primary BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Product Attributes Table
CREATE TABLE product_attributes (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    attribute_name VARCHAR(100) NOT NULL,
    attribute_value TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Pricing Rules Table
CREATE TABLE pricing_rules (
    id BIGSERIAL PRIMARY KEY,
    variant_id BIGINT REFERENCES product_variants(id) ON DELETE CASCADE,
    customer_type VARCHAR(50) NOT NULL, -- 'retail', 'wholesale', 'agent'
    min_quantity INTEGER DEFAULT 1,
    max_quantity INTEGER,
    discount_type VARCHAR(20) CHECK (discount_type IN ('percentage', 'fixed')),
    discount_value DECIMAL(10, 2),
    start_date DATE,
    end_date DATE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.3 Inventory Tables

```sql
-- Warehouses Table
CREATE TABLE warehouses (
    id SERIAL PRIMARY KEY,
    code VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50), -- 'main', 'regional', 'retail'
    address TEXT,
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100),
    pincode VARCHAR(20),
    phone VARCHAR(20),
    email VARCHAR(255),
    manager_id BIGINT REFERENCES users(id),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bin Locations Table
CREATE TABLE bin_locations (
    id BIGSERIAL PRIMARY KEY,
    warehouse_id INTEGER NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    zone VARCHAR(20), -- 'A', 'B', 'C'
    aisle VARCHAR(20), -- '01', '02'
    rack VARCHAR(20),  -- 'R1', 'R2'
    bin VARCHAR(20),   -- 'B1', 'B2'
    capacity INTEGER,  -- Max units
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(warehouse_id, zone, aisle, rack, bin)
);

-- Inventory Table
CREATE TABLE inventory (
    id BIGSERIAL PRIMARY KEY,
    variant_id BIGINT NOT NULL REFERENCES product_variants(id) ON DELETE CASCADE,
    warehouse_id INTEGER NOT NULL REFERENCES warehouses(id) ON DELETE CASCADE,
    bin_location_id BIGINT REFERENCES bin_locations(id),
    quantity INTEGER DEFAULT 0 CHECK (quantity >= 0),
    reserved_quantity INTEGER DEFAULT 0 CHECK (reserved_quantity >= 0),
    reorder_point INTEGER DEFAULT 10,
    reorder_quantity INTEGER DEFAULT 50,
    last_counted_at TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(variant_id, warehouse_id, bin_location_id)
);

-- Stock Movements Table
CREATE TABLE stock_movements (
    id BIGSERIAL PRIMARY KEY,
    variant_id BIGINT NOT NULL REFERENCES product_variants(id),
    from_warehouse_id INTEGER REFERENCES warehouses(id),
    to_warehouse_id INTEGER REFERENCES warehouses(id),
    movement_type VARCHAR(20) NOT NULL CHECK (movement_type IN ('IN', 'OUT', 'TRANSFER', 'ADJUSTMENT', 'RETURN')),
    quantity INTEGER NOT NULL,
    reference_type VARCHAR(50), -- 'order', 'return', 'production', 'adjustment'
    reference_id BIGINT,
    notes TEXT,
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Batches Table
CREATE TABLE batches (
    id BIGSERIAL PRIMARY KEY,
    variant_id BIGINT NOT NULL REFERENCES product_variants(id),
    batch_number VARCHAR(50) UNIQUE NOT NULL,
    manufacturing_date DATE,
    expiry_date DATE,
    quantity INTEGER NOT NULL,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'expired', 'recalled')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Stock Adjustments Table
CREATE TABLE stock_adjustments (
    id BIGSERIAL PRIMARY KEY,
    variant_id BIGINT NOT NULL REFERENCES product_variants(id),
    warehouse_id INTEGER NOT NULL REFERENCES warehouses(id),
    adjustment_type VARCHAR(20) CHECK (adjustment_type IN ('damaged', 'lost', 'found', 'correction')),
    quantity INTEGER NOT NULL, -- Can be negative
    reason TEXT NOT NULL,
    approved_by BIGINT REFERENCES users(id),
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.4 Manufacturing Tables

```sql
-- Suppliers Table
CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    code VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    contact_person VARCHAR(100),
    email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    city VARCHAR(100),
    state VARCHAR(100),
    country VARCHAR(100),
    pincode VARCHAR(20),
    gst_number VARCHAR(50),
    pan_number VARCHAR(50),
    payment_terms VARCHAR(50), -- 'Net 30', 'Net 60', 'COD'
    rating DECIMAL(2, 1) CHECK (rating BETWEEN 0 AND 5),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Raw Materials Table
CREATE TABLE raw_materials (
    id BIGSERIAL PRIMARY KEY,
    supplier_id INTEGER REFERENCES suppliers(id),
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(50) UNIQUE NOT NULL,
    category VARCHAR(100), -- 'fabric', 'buttons', 'zippers', 'thread'
    unit VARCHAR(20), -- 'meters', 'pieces', 'kg'
    unit_cost DECIMAL(10, 2) NOT NULL,
    stock_quantity DECIMAL(10, 2) DEFAULT 0,
    reorder_point DECIMAL(10, 2) DEFAULT 100,
    reorder_quantity DECIMAL(10, 2) DEFAULT 500,
    description TEXT,
    specifications JSON,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Purchase Orders Table
CREATE TABLE purchase_orders (
    id BIGSERIAL PRIMARY KEY,
    supplier_id INTEGER NOT NULL REFERENCES suppliers(id),
    po_number VARCHAR(50) UNIQUE NOT NULL,
    po_date DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_delivery_date DATE,
    total_amount DECIMAL(12, 2) NOT NULL,
    tax_amount DECIMAL(10, 2),
    status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'submitted', 'approved', 'received', 'cancelled')),
    payment_terms VARCHAR(50),
    notes TEXT,
    created_by BIGINT REFERENCES users(id),
    approved_by BIGINT REFERENCES users(id),
    approved_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Purchase Order Items Table
CREATE TABLE purchase_order_items (
    id BIGSERIAL PRIMARY KEY,
    po_id BIGINT NOT NULL REFERENCES purchase_orders(id) ON DELETE CASCADE,
    material_id BIGINT NOT NULL REFERENCES raw_materials(id),
    quantity DECIMAL(10, 2) NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    tax_rate DECIMAL(5, 2),
    received_quantity DECIMAL(10, 2) DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'partial', 'received', 'cancelled')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bill of Materials (BOM) Table
CREATE TABLE bom (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    material_id BIGINT NOT NULL REFERENCES raw_materials(id),
    quantity_required DECIMAL(10, 3) NOT NULL,
    wastage_percentage DECIMAL(5, 2) DEFAULT 5.00,
    unit VARCHAR(20) NOT NULL,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(product_id, material_id)
);

-- Work Orders Table
CREATE TABLE work_orders (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id),
    wo_number VARCHAR(50) UNIQUE NOT NULL,
    quantity INTEGER NOT NULL,
    start_date DATE NOT NULL,
    target_end_date DATE,
    actual_end_date DATE,
    priority VARCHAR(20) DEFAULT 'medium' CHECK (priority IN ('low', 'medium', 'high', 'urgent')),
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'completed', 'on_hold', 'cancelled')),
    total_cost DECIMAL(12, 2),
    notes TEXT,
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Production Stages Table
CREATE TABLE production_stages (
    id BIGSERIAL PRIMARY KEY,
    work_order_id BIGINT NOT NULL REFERENCES work_orders(id) ON DELETE CASCADE,
    stage_name VARCHAR(100) NOT NULL, -- 'cutting', 'stitching', 'finishing', 'packaging'
    sequence INTEGER NOT NULL,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'completed', 'failed')),
    assigned_to BIGINT REFERENCES users(id),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(work_order_id, sequence)
);

-- Quality Checks Table
CREATE TABLE quality_checks (
    id BIGSERIAL PRIMARY KEY,
    stage_id BIGINT NOT NULL REFERENCES production_stages(id) ON DELETE CASCADE,
    checked_by BIGINT NOT NULL REFERENCES users(id),
    check_type VARCHAR(50), -- 'visual', 'measurement', 'functional'
    result VARCHAR(20) CHECK (result IN ('passed', 'failed', 'conditional')),
    defect_count INTEGER DEFAULT 0,
    defect_type VARCHAR(100),
    severity VARCHAR(20) CHECK (severity IN ('minor', 'major', 'critical')),
    action_taken TEXT,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Defects Table
CREATE TABLE defects (
    id BIGSERIAL PRIMARY KEY,
    work_order_id BIGINT NOT NULL REFERENCES work_orders(id),
    stage_id BIGINT REFERENCES production_stages(id),
    defect_type VARCHAR(100) NOT NULL,
    defect_category VARCHAR(50), -- 'stitching', 'fabric', 'sizing', 'finishing'
    quantity INTEGER NOT NULL,
    severity VARCHAR(20) CHECK (severity IN ('minor', 'major', 'critical')),
    reported_by BIGINT REFERENCES users(id),
    action_taken TEXT,
    status VARCHAR(20) DEFAULT 'open' CHECK (status IN ('open', 'in_progress', 'resolved', 'rejected')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.5 Order Management Tables

```sql
-- Customers Table
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE REFERENCES users(id) ON DELETE SET NULL,
    customer_type VARCHAR(20) DEFAULT 'retail' CHECK (customer_type IN ('retail', 'wholesale', 'agent')),
    company_name VARCHAR(255),
    gst_number VARCHAR(50),
    pan_number VARCHAR(50),
    credit_limit DECIMAL(12, 2) DEFAULT 0,
    credit_used DECIMAL(12, 2) DEFAULT 0,
    payment_terms VARCHAR(50),
    segment VARCHAR(50), -- 'vip', 'regular', 'new'
    lifetime_value DECIMAL(12, 2) DEFAULT 0,
    total_orders INTEGER DEFAULT 0,
    notes TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Agents Table
CREATE TABLE agents (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    agent_code VARCHAR(20) UNIQUE NOT NULL,
    territory VARCHAR(100),
    commission_percentage DECIMAL(5, 2) DEFAULT 5.00,
    target_amount DECIMAL(12, 2),
    current_sales DECIMAL(12, 2) DEFAULT 0,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'suspended')),
    joined_date DATE DEFAULT CURRENT_DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Orders Table
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    agent_id BIGINT REFERENCES agents(id),
    order_number VARCHAR(50) UNIQUE NOT NULL,
    channel VARCHAR(50) DEFAULT 'website', -- 'website', 'marketplace', 'agent', 'retail'
    order_type VARCHAR(20) DEFAULT 'standard', -- 'standard', 'bulk', 'custom'
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'confirmed', 'processing', 'packed', 'shipped', 'delivered', 'cancelled', 'returned')),
    payment_status VARCHAR(20) DEFAULT 'pending' CHECK (payment_status IN ('pending', 'paid', 'partial', 'refunded', 'failed')),
    subtotal DECIMAL(12, 2) NOT NULL,
    tax_amount DECIMAL(10, 2) DEFAULT 0,
    shipping_cost DECIMAL(10, 2) DEFAULT 0,
    discount_amount DECIMAL(10, 2) DEFAULT 0,
    discount_code VARCHAR(50),
    total_amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'INR',
    notes TEXT,
    internal_notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Order Items Table
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    variant_id BIGINT NOT NULL REFERENCES product_variants(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10, 2) NOT NULL,
    discount_amount DECIMAL(10, 2) DEFAULT 0,
    tax_rate DECIMAL(5, 2) DEFAULT 0,
    tax_amount DECIMAL(10, 2) DEFAULT 0,
    total_amount DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Order Addresses Table
CREATE TABLE order_addresses (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    address_type VARCHAR(20) CHECK (address_type IN ('billing', 'shipping')),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(255),
    address_line1 TEXT NOT NULL,
    address_line2 TEXT,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(100) NOT NULL,
    country VARCHAR(100) NOT NULL,
    pincode VARCHAR(20) NOT NULL,
    landmark VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Order Status History Table
CREATE TABLE order_status_history (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    status VARCHAR(20) NOT NULL,
    notes TEXT,
    changed_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Shipments Table
CREATE TABLE shipments (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    warehouse_id INTEGER REFERENCES warehouses(id),
    tracking_number VARCHAR(100) UNIQUE,
    carrier VARCHAR(100), -- 'FedEx', 'DHL', 'Delhivery'
    carrier_code VARCHAR(50),
    service_type VARCHAR(50), -- 'standard', 'express', 'overnight'
    weight DECIMAL(8, 2),
    dimensions VARCHAR(50),
    shipped_at TIMESTAMP,
    estimated_delivery TIMESTAMP,
    delivered_at TIMESTAMP,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'picked', 'in_transit', 'out_for_delivery', 'delivered', 'failed', 'returned')),
    label_url VARCHAR(500),
    tracking_url VARCHAR(500),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Returns Table
CREATE TABLE returns (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    return_number VARCHAR(50) UNIQUE NOT NULL,
    reason VARCHAR(255) NOT NULL,
    reason_category VARCHAR(50), -- 'defective', 'wrong_item', 'size_issue', 'not_needed'
    status VARCHAR(20) DEFAULT 'requested' CHECK (status IN ('requested', 'approved', 'rejected', 'received', 'refunded', 'exchanged')),
    refund_amount DECIMAL(10, 2),
    refund_method VARCHAR(50), -- 'original_payment', 'store_credit', 'bank_transfer'
    approved_by BIGINT REFERENCES users(id),
    approved_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Return Items Table
CREATE TABLE return_items (
    id BIGSERIAL PRIMARY KEY,
    return_id BIGINT NOT NULL REFERENCES returns(id) ON DELETE CASCADE,
    order_item_id BIGINT NOT NULL REFERENCES order_items(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    condition VARCHAR(50), -- 'unused', 'used', 'damaged'
    action VARCHAR(20) CHECK (action IN ('refund', 'replace', 'repair', 'discard')),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.6 Payment & Finance Tables

```sql
-- Payments Table
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT REFERENCES orders(id),
    customer_id BIGINT REFERENCES customers(id),
    payment_id VARCHAR(100) UNIQUE NOT NULL,
    gateway VARCHAR(50) NOT NULL, -- 'stripe', 'razorpay', 'paypal'
    method VARCHAR(50), -- 'card', 'upi', 'netbanking', 'wallet', 'cod'
    amount DECIMAL(12, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'INR',
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded', 'cancelled')),
    transaction_id VARCHAR(255),
    gateway_response JSON,
    failure_reason TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Refunds Table
CREATE TABLE refunds (
    id BIGSERIAL PRIMARY KEY,
    payment_id BIGINT NOT NULL REFERENCES payments(id),
    order_id BIGINT REFERENCES orders(id),
    refund_id VARCHAR(100) UNIQUE NOT NULL,
    amount DECIMAL(12, 2) NOT NULL,
    reason TEXT,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    gateway_response JSON,
    processed_by BIGINT REFERENCES users(id),
    processed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Invoices Table
CREATE TABLE invoices (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT UNIQUE REFERENCES orders(id),
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    invoice_number VARCHAR(50) UNIQUE NOT NULL,
    invoice_date DATE NOT NULL DEFAULT CURRENT_DATE,
    due_date DATE,
    subtotal DECIMAL(12, 2) NOT NULL,
    tax_amount DECIMAL(10, 2) DEFAULT 0,
    discount_amount DECIMAL(10, 2) DEFAULT 0,
    shipping_cost DECIMAL(10, 2) DEFAULT 0,
    total_amount DECIMAL(12, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'sent', 'paid', 'overdue', 'cancelled')),
    pdf_url VARCHAR(500),
    paid_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Invoice Items Table
CREATE TABLE invoice_items (
    id BIGSERIAL PRIMARY KEY,
    invoice_id BIGINT NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description TEXT NOT NULL,
    quantity DECIMAL(10, 2) NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    tax_rate DECIMAL(5, 2) DEFAULT 0,
    tax_amount DECIMAL(10, 2) DEFAULT 0,
    discount_amount DECIMAL(10, 2) DEFAULT 0,
    total_amount DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Ledger Table (for credit management)
CREATE TABLE ledger (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    reference_type VARCHAR(50) NOT NULL, -- 'invoice', 'payment', 'credit_note', 'adjustment'
    reference_id BIGINT,
    entry_type VARCHAR(10) CHECK (entry_type IN ('debit', 'credit')),
    amount DECIMAL(12, 2) NOT NULL,
    balance DECIMAL(12, 2) NOT NULL,
    description TEXT,
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Expenses Table
CREATE TABLE expenses (
    id BIGSERIAL PRIMARY KEY,
    category VARCHAR(100) NOT NULL, -- 'materials', 'labor', 'utilities', 'rent', 'marketing'
    subcategory VARCHAR(100),
    description TEXT NOT NULL,
    amount DECIMAL(12, 2) NOT NULL,
    expense_date DATE NOT NULL DEFAULT CURRENT_DATE,
    vendor VARCHAR(255),
    payment_method VARCHAR(50),
    receipt_url VARCHAR(500),
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'approved', 'paid', 'rejected')),
    approved_by BIGINT REFERENCES users(id),
    approved_at TIMESTAMP,
    created_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.7 CRM Tables

```sql
-- Customer Segments Table
CREATE TABLE customer_segments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    criteria JSON, -- Store segment rules as JSON
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Customer Segment Mapping Table
CREATE TABLE customer_segment_mapping (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    segment_id INTEGER NOT NULL REFERENCES customer_segments(id) ON DELETE CASCADE,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(customer_id, segment_id)
);

-- Tickets Table
CREATE TABLE tickets (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    assigned_to BIGINT REFERENCES users(id),
    ticket_number VARCHAR(50) UNIQUE NOT NULL,
    subject VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    priority VARCHAR(20) DEFAULT 'medium' CHECK (priority IN ('low', 'medium', 'high', 'urgent')),
    status VARCHAR(20) DEFAULT 'open' CHECK (status IN ('open', 'in_progress', 'waiting', 'resolved', 'closed')),
    category VARCHAR(100), -- 'order_issue', 'product_inquiry', 'complaint', 'feedback'
    source VARCHAR(50), -- 'email', 'phone', 'chat', 'web'
    first_response_at TIMESTAMP,
    resolved_at TIMESTAMP,
    closed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Ticket Messages Table
CREATE TABLE ticket_messages (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    user_id BIGINT REFERENCES users(id),
    message TEXT NOT NULL,
    is_internal BOOLEAN DEFAULT FALSE,
    attachments JSON, -- Array of attachment URLs
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Commissions Table
CREATE TABLE commissions (
    id BIGSERIAL PRIMARY KEY,
    agent_id BIGINT NOT NULL REFERENCES agents(id),
    order_id BIGINT NOT NULL REFERENCES orders(id),
    amount DECIMAL(10, 2) NOT NULL,
    percentage DECIMAL(5, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'approved', 'paid', 'cancelled')),
    approved_by BIGINT REFERENCES users(id),
    approved_at TIMESTAMP,
    paid_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Loyalty Points Table
CREATE TABLE loyalty_points (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    points INTEGER NOT NULL,
    transaction_type VARCHAR(20) CHECK (transaction_type IN ('earned', 'redeemed', 'expired', 'adjusted')),
    earned_from VARCHAR(50), -- 'purchase', 'referral', 'bonus'
    reference_id BIGINT,
    description TEXT,
    expires_at DATE,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'expired', 'used')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.8 Marketplace Integration Tables

```sql
-- Marketplace Listings Table
CREATE TABLE marketplace_listings (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id),
    variant_id BIGINT REFERENCES product_variants(id),
    marketplace VARCHAR(50) NOT NULL, -- 'amazon', 'flipkart', 'myntra'
    listing_id VARCHAR(255),
    sku VARCHAR(100),
    title VARCHAR(500),
    price DECIMAL(10, 2),
    compare_at_price DECIMAL(10, 2),
    inventory_sync BOOLEAN DEFAULT TRUE,
    price_sync BOOLEAN DEFAULT TRUE,
    status VARCHAR(20) DEFAULT 'draft' CHECK (status IN ('draft', 'pending', 'active', 'inactive', 'rejected')),
    rejection_reason TEXT,
    last_synced_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(product_id, marketplace)
);

-- Sync Logs Table
CREATE TABLE sync_logs (
    id BIGSERIAL PRIMARY KEY,
    listing_id BIGINT REFERENCES marketplace_listings(id),
    sync_type VARCHAR(50) NOT NULL, -- 'product', 'inventory', 'price', 'order'
    marketplace VARCHAR(50) NOT NULL,
    status VARCHAR(20) CHECK (status IN ('success', 'failed', 'partial')),
    error_message TEXT,
    request_data JSON,
    response_data JSON,
    retry_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Webhooks Table
CREATE TABLE webhooks (
    id SERIAL PRIMARY KEY,
    url VARCHAR(500) NOT NULL,
    event_type VARCHAR(100) NOT NULL, -- 'order.created', 'order.updated', 'payment.completed'
    secret VARCHAR(255),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Webhook Logs Table
CREATE TABLE webhook_logs (
    id BIGSERIAL PRIMARY KEY,
    webhook_id INTEGER REFERENCES webhooks(id),
    event_type VARCHAR(100) NOT NULL,
    payload JSON NOT NULL,
    response_code INTEGER,
    response_body TEXT,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    status VARCHAR(20) CHECK (status IN ('pending', 'success', 'failed')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP
);
```

### 3.9 System & Audit Tables

```sql
-- Audit Logs Table
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    action VARCHAR(100) NOT NULL, -- 'create', 'update', 'delete', 'login', 'logout'
    resource_type VARCHAR(100) NOT NULL, -- 'product', 'order', 'user'
    resource_id BIGINT,
    old_values JSON,
    new_values JSON,
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Notifications Table
CREATE TABLE notifications (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    type VARCHAR(50) NOT NULL, -- 'order', 'payment', 'system', 'marketing'
    channel VARCHAR(20) CHECK (channel IN ('email', 'sms', 'push', 'in_app')),
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    data JSON, -- Additional data
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'delivered', 'failed', 'read')),
    sent_at TIMESTAMP,
    read_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Email Logs Table
CREATE TABLE email_logs (
    id BIGSERIAL PRIMARY KEY,
    to_email VARCHAR(255) NOT NULL,
    cc_emails TEXT,
    bcc_emails TEXT,
    subject VARCHAR(255) NOT NULL,
    template VARCHAR(100),
    variables JSON,
    status VARCHAR(20) CHECK (status IN ('pending', 'sent', 'delivered', 'bounced', 'failed')),
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    opened_at TIMESTAMP,
    clicked_at TIMESTAMP,
    error TEXT,
    provider_id VARCHAR(255), -- SES Message ID
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Settings Table
CREATE TABLE settings (
    id SERIAL PRIMARY KEY,
    key VARCHAR(100) UNIQUE NOT NULL,
    value TEXT NOT NULL,
    type VARCHAR(20) DEFAULT 'string' CHECK (type IN ('string', 'number', 'boolean', 'json')),
    description TEXT,
    is_public BOOLEAN DEFAULT FALSE, -- Can be accessed without authentication
    group_name VARCHAR(50), -- 'general', 'email', 'payment', 'shipping'
    updated_by BIGINT REFERENCES users(id),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 3.10 Analytics Tables

```sql
-- Daily Metrics Table
CREATE TABLE daily_metrics (
    id SERIAL PRIMARY KEY,
    date DATE UNIQUE NOT NULL,
    total_orders INTEGER DEFAULT 0,
    total_revenue DECIMAL(12, 2) DEFAULT 0,
    avg_order_value DECIMAL(10, 2) DEFAULT 0,
    new_customers INTEGER DEFAULT 0,
    returning_customers INTEGER DEFAULT 0,
    total_units_sold INTEGER DEFAULT 0,
    conversion_rate DECIMAL(5, 2) DEFAULT 0,
    cart_abandonment_rate DECIMAL(5, 2) DEFAULT 0,
    refund_rate DECIMAL(5, 2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Product Views Table
CREATE TABLE product_views (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL REFERENCES products(id),
    variant_id BIGINT REFERENCES product_variants(id),
    user_id BIGINT REFERENCES users(id),
    session_id VARCHAR(255),
    ip_address VARCHAR(45),
    referrer VARCHAR(500),
    user_agent TEXT,
    duration INTEGER, -- Time spent in seconds
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Search Logs Table
CREATE TABLE search_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    session_id VARCHAR(255),
    query VARCHAR(255) NOT NULL,
    filters JSON, -- Applied filters
    results_count INTEGER DEFAULT 0,
    clicked_product_id BIGINT REFERENCES products(id),
    click_position INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Cart Abandonment Table
CREATE TABLE cart_abandonment (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    session_id VARCHAR(255) NOT NULL,
    cart_data JSON NOT NULL,
    cart_value DECIMAL(10, 2),
    recovery_email_sent BOOLEAN DEFAULT FALSE,
    recovered_order_id BIGINT REFERENCES orders(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    recovered_at TIMESTAMP
);
```

---

## 4. Indexing Strategy

### 4.1 Primary Indexes

```sql
-- Users & Auth
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_token ON sessions(token);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);

-- Products
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_slug ON products(slug);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_is_featured ON products(is_featured);
CREATE INDEX idx_product_variants_product_id ON product_variants(product_id);
CREATE INDEX idx_product_variants_sku ON product_variants(sku);
CREATE INDEX idx_product_variants_barcode ON product_variants(barcode);
CREATE INDEX idx_product_images_product_id ON product_images(product_id);

-- Inventory
CREATE INDEX idx_inventory_variant_id ON inventory(variant_id);
CREATE INDEX idx_inventory_warehouse_id ON inventory(warehouse_id);
CREATE INDEX idx_inventory_bin_location_id ON inventory(bin_location_id);
CREATE INDEX idx_stock_movements_variant_id ON stock_movements(variant_id);
CREATE INDEX idx_stock_movements_created_at ON stock_movements(created_at DESC);
CREATE INDEX idx_stock_movements_reference ON stock_movements(reference_type, reference_id);

-- Manufacturing
CREATE INDEX idx_work_orders_product_id ON work_orders(product_id);
CREATE INDEX idx_work_orders_status ON work_orders(status);
CREATE INDEX idx_work_orders_start_date ON work_orders(start_date);
CREATE INDEX idx_production_stages_wo_id ON production_stages(work_order_id);
CREATE INDEX idx_production_stages_status ON production_stages(status);
CREATE INDEX idx_purchase_orders_supplier_id ON purchase_orders(supplier_id);
CREATE INDEX idx_purchase_orders_status ON purchase_orders(status);

-- Orders
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_agent_id ON orders(agent_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_orders_order_number ON orders(order_number);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_variant_id ON order_items(variant_id);
CREATE INDEX idx_shipments_order_id ON shipments(order_id);
CREATE INDEX idx_shipments_tracking_no ON shipments(tracking_number);
CREATE INDEX idx_returns_order_id ON returns(order_id);
CREATE INDEX idx_returns_status ON returns(status);

-- Payments
CREATE INDEX idx_payments_order_id ON payments(order_id);
CREATE INDEX idx_payments_customer_id ON payments(customer_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);
CREATE INDEX idx_invoices_order_id ON invoices(order_id);
CREATE INDEX idx_invoices_customer_id ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(status);

-- CRM
CREATE INDEX idx_tickets_customer_id ON tickets(customer_id);
CREATE INDEX idx_tickets_assigned_to ON tickets(assigned_to);
CREATE INDEX idx_tickets_status ON tickets(status);
CREATE INDEX idx_tickets_created_at ON tickets(created_at DESC);
CREATE INDEX idx_ticket_messages_ticket_id ON ticket_messages(ticket_id);
CREATE INDEX idx_commissions_agent_id ON commissions(agent_id);
CREATE INDEX idx_commissions_order_id ON commissions(order_id);

-- System
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs(resource_type, resource_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_status ON notifications(status);
```

### 4.2 Composite Indexes

```sql
-- Performance-critical composite indexes
CREATE INDEX idx_orders_status_date ON orders(status, created_at DESC);
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
CREATE INDEX idx_inventory_variant_warehouse ON inventory(variant_id, warehouse_id);
CREATE INDEX idx_stock_movements_variant_date ON stock_movements(variant_id, created_at DESC);
CREATE INDEX idx_product_variants_product_active ON product_variants(product_id, is_active);
CREATE INDEX idx_order_items_order_variant ON order_items(order_id, variant_id);
CREATE INDEX idx_payments_order_status ON payments(order_id, status);
CREATE INDEX idx_work_orders_status_date ON work_orders(status, start_date);
```

### 4.3 Full-Text Search Indexes

```sql
-- Full-text search for products
CREATE INDEX idx_products_fulltext ON products 
    USING GIN(to_tsvector('english', name || ' ' || COALESCE(description, '')));

-- Full-text search for customers
CREATE INDEX idx_customers_fulltext ON customers 
    USING GIN(to_tsvector('english', COALESCE(company_name, '') || ' ' || COALESCE(notes, '')));

-- Full-text search for tickets
CREATE INDEX idx_tickets_fulltext ON tickets 
    USING GIN(to_tsvector('english', subject || ' ' || description));
```

---

## 5. Data Integrity Rules

### 5.1 Referential Integrity

```sql
-- Cascading deletes where appropriate
-- Order deletion cascades to order_items and order_addresses
-- Product deletion cascades to product_variants and product_images
-- User deletion preserves audit trail (SET NULL)

-- Check constraints for data validation
ALTER TABLE users 
    ADD CONSTRAINT chk_email_format 
    CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

ALTER TABLE product_variants 
    ADD CONSTRAINT chk_positive_price 
    CHECK (retail_price >= 0 AND wholesale_price >= 0);

ALTER TABLE inventory 
    ADD CONSTRAINT chk_reserved_quantity 
    CHECK (reserved_quantity <= quantity);

ALTER TABLE payments 
    ADD CONSTRAINT chk_positive_amount 
    CHECK (amount > 0);
```

### 5.2 Business Rules

```sql
-- Trigger: Update inventory reserved quantity when order is placed
CREATE OR REPLACE FUNCTION update_inventory_reserved()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE inventory 
    SET reserved_quantity = reserved_quantity + NEW.quantity
    WHERE variant_id = NEW.variant_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_order_item_reserve_inventory
AFTER INSERT ON order_items
FOR EACH ROW
EXECUTE FUNCTION update_inventory_reserved();

-- Trigger: Update customer lifetime value
CREATE OR REPLACE FUNCTION update_customer_lifetime_value()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.status = 'delivered' THEN
        UPDATE customers 
        SET lifetime_value = lifetime_value + NEW.total_amount,
            total_orders = total_orders + 1
        WHERE id = NEW.customer_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_customer_ltv
AFTER UPDATE OF status ON orders
FOR EACH ROW
WHEN (NEW.status = 'delivered' AND OLD.status != 'delivered')
EXECUTE FUNCTION update_customer_lifetime_value();

-- Trigger: Auto-update timestamps
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to all tables with updated_at column
CREATE TRIGGER trg_users_timestamp BEFORE UPDATE ON users 
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER trg_products_timestamp BEFORE UPDATE ON products 
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

CREATE TRIGGER trg_orders_timestamp BEFORE UPDATE ON orders 
    FOR EACH ROW EXECUTE FUNCTION update_timestamp();

-- (Apply to other tables as needed)
```

---

## 6. Performance Optimization

### 6.1 Partitioning Strategy

```sql
-- Partition large tables by date for better performance
-- Orders table partitioning by month
CREATE TABLE orders_2026_03 PARTITION OF orders
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

CREATE TABLE orders_2026_04 PARTITION OF orders
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

-- Audit logs partitioning by month
CREATE TABLE audit_logs_2026_03 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

-- Stock movements partitioning by quarter
CREATE TABLE stock_movements_2026_q1 PARTITION OF stock_movements
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
```

### 6.2 Materialized Views

```sql
-- Materialized view for product analytics
CREATE MATERIALIZED VIEW mv_product_analytics AS
SELECT 
    p.id,
    p.name,
    p.category_id,
    COUNT(DISTINCT oi.order_id) as total_orders,
    SUM(oi.quantity) as total_units_sold,
    SUM(oi.total_amount) as total_revenue,
    AVG(oi.unit_price) as avg_selling_price,
    COUNT(DISTINCT pv.id) as total_views
FROM products p
LEFT JOIN product_variants pv_variant ON pv_variant.product_id = p.id
LEFT JOIN order_items oi ON oi.variant_id = pv_variant.id
LEFT JOIN product_views pv ON pv.product_id = p.id
GROUP BY p.id, p.name, p.category_id;

CREATE UNIQUE INDEX ON mv_product_analytics(id);

-- Materialized view for customer analytics
CREATE MATERIALIZED VIEW mv_customer_analytics AS
SELECT 
    c.id,
    c.customer_type,
    c.segment,
    COUNT(DISTINCT o.id) as total_orders,
    SUM(o.total_amount) as total_spent,
    AVG(o.total_amount) as avg_order_value,
    MAX(o.created_at) as last_order_date,
    DATE_PART('day', CURRENT_TIMESTAMP - MAX(o.created_at)) as days_since_last_order
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status IN ('delivered', 'shipped')
GROUP BY c.id, c.customer_type, c.segment;

CREATE UNIQUE INDEX ON mv_customer_analytics(id);

-- Refresh materialized views (schedule this via cron job)
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_product_analytics;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_customer_analytics;
```

### 6.3 Query Optimization Tips

```sql
-- Use EXPLAIN ANALYZE to understand query performance
EXPLAIN ANALYZE
SELECT * FROM orders 
WHERE customer_id = 123 
AND status = 'delivered' 
ORDER BY created_at DESC 
LIMIT 10;

-- Use covering indexes for frequently accessed columns
CREATE INDEX idx_orders_covering ON orders(customer_id, status, created_at, total_amount);

-- Avoid SELECT *, specify only needed columns
SELECT id, order_number, status, total_amount, created_at
FROM orders
WHERE customer_id = 123;

-- Use pagination with offset/limit or cursor-based
-- Cursor-based pagination (better for large datasets)
SELECT * FROM orders 
WHERE id > last_seen_id 
ORDER BY id 
LIMIT 20;

-- Use connection pooling (handled at application level)
-- Recommended pool size: 20-50 connections
```

---

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Next Review:** April 26, 2026  
**Status:** ✅ Production-Ready

---

[← Back to Index](./00_MASTER_INDEX.md) | [Next: Backend Architecture →](./03_BACKEND_ARCHITECTURE.md)
