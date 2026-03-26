# 05. API Contracts
## Enterprise Men's Clothing Manufacturing & Sales Platform

**Version:** 3.0 (OpenAPI 3.0)  
**Last Updated:** March 26, 2026  
**Base URL:** `https://api.yourcompany.com/v1`  
**Authentication:** JWT Bearer Token  
**Status:** ✅ Production-Ready

---

## 📋 Table of Contents

1. [Authentication & Authorization](#1-authentication--authorization)
2. [Product Management](#2-product-management)
3. [Order Management](#3-order-management)
4. [Inventory Management](#4-inventory-management)
5. [Manufacturing Operations](#5-manufacturing-operations)
6. [CRM & Customer Management](#6-crm--customer-management)
7. [Analytics & Reporting](#7-analytics--reporting)
8. [Marketplace Integration](#8-marketplace-integration)

---

## Global Conventions

### Headers

```yaml
Common Request Headers:
  Authorization: Bearer <access_token>
  Content-Type: application/json
  Accept: application/json
  X-Request-ID: <unique-request-id>

Common Response Headers:
  Content-Type: application/json
  X-Request-ID: <unique-request-id>
  X-Rate-Limit-Limit: 100
  X-Rate-Limit-Remaining: 95
  X-Rate-Limit-Reset: 1678901234
```

### Response Format

```typescript
// Success Response
{
  "success": true,
  "data": { /* response data */ },
  "message": "Operation successful",
  "pagination": { /* if applicable */ }
}

// Error Response
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "email",
        "message": "Email is required"
      }
    ]
  }
}
```

### Pagination

```typescript
// Query Parameters
?page=1&limit=20&sortBy=createdAt&sortOrder=desc

// Response Pagination Object
{
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

### Error Codes

```yaml
HTTP Status Codes:
  200: OK - Success
  201: Created - Resource created
  204: No Content - Success with no response body
  400: Bad Request - Invalid input
  401: Unauthorized - Authentication required
  403: Forbidden - Insufficient permissions
  404: Not Found - Resource not found
  409: Conflict - Resource conflict
  422: Unprocessable Entity - Validation error
  429: Too Many Requests - Rate limit exceeded
  500: Internal Server Error - Server error
  503: Service Unavailable - Maintenance

Application Error Codes:
  VALIDATION_ERROR: Input validation failed
  UNAUTHORIZED: Authentication failed
  FORBIDDEN: Access denied
  NOT_FOUND: Resource not found
  DUPLICATE_ENTRY: Resource already exists
  INSUFFICIENT_STOCK: Not enough inventory
  PAYMENT_FAILED: Payment processing failed
  INVALID_CREDENTIALS: Wrong email/password
  TOKEN_EXPIRED: JWT token expired
```

---

## 1. Authentication & Authorization

### 1.1 Register User

```http
POST /auth/register
```

**Request Body:**

```json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "password": "SecurePass123!",
  "phone": "+919876543210",
  "role": "customer"
}
```

**Response: `201 Created`**

```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "firstName": "John",
      "lastName": "Doe",
      "email": "john.doe@example.com",
      "phone": "+919876543210",
      "roles": ["customer"],
      "status": "active",
      "createdAt": "2026-03-26T10:00:00Z"
    },
    "tokens": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expiresIn": 900,
      "tokenType": "Bearer"
    }
  },
  "message": "User registered successfully"
}
```

### 1.2 Login

```http
POST /auth/login
```

**Request Body:**

```json
{
  "email": "john.doe@example.com",
  "password": "SecurePass123!"
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "firstName": "John",
      "lastName": "Doe",
      "email": "john.doe@example.com",
      "roles": ["customer"],
      "permissions": ["orders:read", "orders:create"]
    },
    "tokens": {
      "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "expiresIn": 900,
      "tokenType": "Bearer"
    }
  },
  "message": "Login successful"
}
```

### 1.3 Refresh Token

```http
POST /auth/refresh
```

**Request Body:**

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expiresIn": 900,
    "tokenType": "Bearer"
  }
}
```

### 1.4 Logout

```http
POST /auth/logout
Authorization: Bearer <access_token>
```

**Response: `200 OK`**

```json
{
  "success": true,
  "message": "Logged out successfully"
}
```

### 1.5 Get Current User

```http
GET /auth/me
Authorization: Bearer <access_token>
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "phone": "+919876543210",
    "roles": ["customer"],
    "permissions": ["orders:read", "orders:create"],
    "status": "active",
    "avatar": "https://cdn.example.com/avatars/1.jpg",
    "createdAt": "2026-01-15T10:00:00Z",
    "lastLoginAt": "2026-03-26T09:30:00Z"
  }
}
```

---

## 2. Product Management

### 2.1 Get Products

```http
GET /products?page=1&limit=20&category=shirts&status=active&search=formal
```

**Query Parameters:**

```typescript
{
  page?: number;          // Default: 1
  limit?: number;         // Default: 20, Max: 100
  category?: string;      // Filter by category slug
  status?: 'active' | 'inactive' | 'draft';
  search?: string;        // Search in name, description, SKU
  minPrice?: number;      // Minimum price filter
  maxPrice?: number;      // Maximum price filter
  sortBy?: string;        // createdAt, name, price
  sortOrder?: 'asc' | 'desc';
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "Premium Formal Shirt",
      "slug": "premium-formal-shirt",
      "sku": "SHIRT-001",
      "description": "High-quality formal shirt made from 100% cotton",
      "category": {
        "id": 5,
        "name": "Formal Shirts",
        "slug": "formal-shirts"
      },
      "images": [
        {
          "id": 1,
          "url": "https://cdn.example.com/products/shirt-001-main.jpg",
          "alt": "Premium Formal Shirt - Front View",
          "position": 0,
          "isMain": true
        }
      ],
      "variants": [
        {
          "id": 1,
          "sku": "SHIRT-001-M-BLUE",
          "size": "M",
          "color": "Blue",
          "retailPrice": 1299.00,
          "wholesalePrice": 999.00,
          "compareAtPrice": 1799.00,
          "stockQuantity": 45,
          "isAvailable": true
        }
      ],
      "attributes": {
        "material": "100% Cotton",
        "fit": "Slim Fit",
        "collar": "Spread Collar",
        "occasion": "Formal"
      },
      "status": "active",
      "isFeatured": true,
      "views": 1250,
      "createdAt": "2026-02-01T10:00:00Z",
      "updatedAt": "2026-03-20T14:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNextPage": true,
    "hasPrevPage": false
  }
}
```

### 2.2 Get Product by ID

```http
GET /products/:id
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "Premium Formal Shirt",
    "slug": "premium-formal-shirt",
    "sku": "SHIRT-001",
    "description": "High-quality formal shirt made from 100% cotton",
    "longDescription": "Experience luxury with our premium formal shirt...",
    "category": {
      "id": 5,
      "name": "Formal Shirts",
      "slug": "formal-shirts",
      "path": ["Men", "Shirts", "Formal Shirts"]
    },
    "images": [
      {
        "id": 1,
        "url": "https://cdn.example.com/products/shirt-001-main.jpg",
        "thumbnailUrl": "https://cdn.example.com/products/shirt-001-main-thumb.jpg",
        "alt": "Premium Formal Shirt - Front View",
        "position": 0,
        "isMain": true
      },
      {
        "id": 2,
        "url": "https://cdn.example.com/products/shirt-001-back.jpg",
        "thumbnailUrl": "https://cdn.example.com/products/shirt-001-back-thumb.jpg",
        "alt": "Premium Formal Shirt - Back View",
        "position": 1,
        "isMain": false
      }
    ],
    "variants": [
      {
        "id": 1,
        "sku": "SHIRT-001-M-BLUE",
        "size": "M",
        "color": "Blue",
        "colorCode": "#1E3A8A",
        "retailPrice": 1299.00,
        "wholesalePrice": 999.00,
        "compareAtPrice": 1799.00,
        "cost": 650.00,
        "stockQuantity": 45,
        "reservedQuantity": 3,
        "availableQuantity": 42,
        "weight": 250,
        "isAvailable": true,
        "barcode": "8901234567890"
      }
    ],
    "attributes": {
      "material": "100% Cotton",
      "fit": "Slim Fit",
      "collar": "Spread Collar",
      "sleeves": "Full Sleeves",
      "pattern": "Solid",
      "occasion": "Formal",
      "careInstructions": "Machine wash cold"
    },
    "seo": {
      "metaTitle": "Premium Formal Shirt - Best Quality | YourBrand",
      "metaDescription": "Shop our premium formal shirt made from 100% cotton...",
      "keywords": ["formal shirt", "cotton shirt", "office wear"]
    },
    "status": "active",
    "isFeatured": true,
    "views": 1250,
    "salesCount": 87,
    "averageRating": 4.7,
    "reviewCount": 45,
    "tags": ["formal", "cotton", "premium", "bestseller"],
    "relatedProducts": [2, 5, 8],
    "createdAt": "2026-02-01T10:00:00Z",
    "updatedAt": "2026-03-20T14:30:00Z"
  }
}
```

### 2.3 Create Product

```http
POST /products
Authorization: Bearer <access_token>
Content-Type: application/json
```

**Request Body:**

```json
{
  "name": "Premium Formal Shirt",
  "slug": "premium-formal-shirt",
  "sku": "SHIRT-001",
  "description": "High-quality formal shirt",
  "longDescription": "Experience luxury with our premium formal shirt...",
  "categoryId": 5,
  "status": "active",
  "isFeatured": true,
  "attributes": {
    "material": "100% Cotton",
    "fit": "Slim Fit"
  },
  "variants": [
    {
      "sku": "SHIRT-001-M-BLUE",
      "size": "M",
      "color": "Blue",
      "retailPrice": 1299.00,
      "wholesalePrice": 999.00,
      "cost": 650.00,
      "stockQuantity": 50
    }
  ],
  "seo": {
    "metaTitle": "Premium Formal Shirt",
    "metaDescription": "Shop premium formal shirt",
    "keywords": ["formal", "shirt"]
  }
}
```

**Response: `201 Created`**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "Premium Formal Shirt",
    "slug": "premium-formal-shirt",
    "status": "active",
    "createdAt": "2026-03-26T10:00:00Z"
  },
  "message": "Product created successfully"
}
```

### 2.4 Update Product

```http
PUT /products/:id
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "name": "Premium Formal Shirt - Updated",
  "status": "active",
  "isFeatured": false,
  "variants": [
    {
      "id": 1,
      "retailPrice": 1399.00
    }
  ]
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "Premium Formal Shirt - Updated",
    "updatedAt": "2026-03-26T11:00:00Z"
  },
  "message": "Product updated successfully"
}
```

### 2.5 Delete Product

```http
DELETE /products/:id
Authorization: Bearer <access_token>
```

**Response: `204 No Content`**

### 2.6 Upload Product Images

```http
POST /products/:id/images
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

**Request Body (Form Data):**

```
images: [File, File, File]
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "images": [
      {
        "id": 1,
        "url": "https://cdn.example.com/products/shirt-001-1.jpg",
        "thumbnailUrl": "https://cdn.example.com/products/shirt-001-1-thumb.jpg",
        "position": 0
      }
    ]
  },
  "message": "Images uploaded successfully"
}
```

---

## 3. Order Management

### 3.1 Create Order

```http
POST /orders
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "customerId": 10,
  "items": [
    {
      "variantId": 1,
      "quantity": 2,
      "price": 1299.00
    },
    {
      "variantId": 5,
      "quantity": 1,
      "price": 899.00
    }
  ],
  "shippingAddress": {
    "fullName": "John Doe",
    "phone": "+919876543210",
    "addressLine1": "123 Main Street",
    "addressLine2": "Apartment 4B",
    "city": "Mumbai",
    "state": "Maharashtra",
    "pincode": "400001",
    "country": "India"
  },
  "billingAddress": {
    "fullName": "John Doe",
    "phone": "+919876543210",
    "addressLine1": "123 Main Street",
    "city": "Mumbai",
    "state": "Maharashtra",
    "pincode": "400001",
    "country": "India"
  },
  "paymentMethod": "razorpay",
  "notes": "Please deliver before 6 PM"
}
```

**Response: `201 Created`**

```json
{
  "success": true,
  "data": {
    "id": 1001,
    "orderNumber": "ORD-2026-001001",
    "customerId": 10,
    "customer": {
      "id": 10,
      "name": "John Doe",
      "email": "john@example.com"
    },
    "items": [
      {
        "id": 1,
        "variantId": 1,
        "productName": "Premium Formal Shirt",
        "sku": "SHIRT-001-M-BLUE",
        "quantity": 2,
        "price": 1299.00,
        "total": 2598.00
      }
    ],
    "subtotal": 3497.00,
    "discount": 0.00,
    "tax": 629.46,
    "shippingCost": 100.00,
    "total": 4226.46,
    "status": "pending",
    "paymentStatus": "pending",
    "shippingAddress": {
      "fullName": "John Doe",
      "addressLine1": "123 Main Street",
      "city": "Mumbai",
      "state": "Maharashtra",
      "pincode": "400001",
      "country": "India"
    },
    "createdAt": "2026-03-26T10:00:00Z"
  },
  "message": "Order created successfully"
}
```

### 3.2 Get Orders

```http
GET /orders?page=1&limit=20&status=pending&customerId=10
```

**Query Parameters:**

```typescript
{
  page?: number;
  limit?: number;
  status?: 'pending' | 'confirmed' | 'processing' | 'shipped' | 'delivered' | 'cancelled';
  paymentStatus?: 'pending' | 'paid' | 'failed' | 'refunded';
  customerId?: number;
  agentId?: number;
  dateFrom?: string;    // YYYY-MM-DD
  dateTo?: string;      // YYYY-MM-DD
  search?: string;      // Search by order number, customer name
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": [
    {
      "id": 1001,
      "orderNumber": "ORD-2026-001001",
      "customer": {
        "id": 10,
        "name": "John Doe",
        "email": "john@example.com"
      },
      "total": 4226.46,
      "status": "pending",
      "paymentStatus": "pending",
      "itemCount": 2,
      "createdAt": "2026-03-26T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 50,
    "totalPages": 3
  }
}
```

### 3.3 Get Order by ID

```http
GET /orders/:id
Authorization: Bearer <access_token>
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "id": 1001,
    "orderNumber": "ORD-2026-001001",
    "customer": {
      "id": 10,
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com",
      "phone": "+919876543210"
    },
    "agent": {
      "id": 5,
      "name": "Agent Smith",
      "commission": 5.00
    },
    "items": [
      {
        "id": 1,
        "variantId": 1,
        "product": {
          "id": 1,
          "name": "Premium Formal Shirt",
          "image": "https://cdn.example.com/products/shirt-001.jpg"
        },
        "sku": "SHIRT-001-M-BLUE",
        "size": "M",
        "color": "Blue",
        "quantity": 2,
        "price": 1299.00,
        "discount": 0.00,
        "tax": 233.82,
        "total": 2598.00
      }
    ],
    "subtotal": 3497.00,
    "discount": 0.00,
    "tax": 629.46,
    "shippingCost": 100.00,
    "total": 4226.46,
    "status": "pending",
    "paymentStatus": "pending",
    "paymentMethod": "razorpay",
    "shippingAddress": {
      "fullName": "John Doe",
      "phone": "+919876543210",
      "addressLine1": "123 Main Street",
      "addressLine2": "Apartment 4B",
      "city": "Mumbai",
      "state": "Maharashtra",
      "pincode": "400001",
      "country": "India"
    },
    "billingAddress": {
      "fullName": "John Doe",
      "addressLine1": "123 Main Street",
      "city": "Mumbai",
      "state": "Maharashtra",
      "pincode": "400001",
      "country": "India"
    },
    "shipment": null,
    "notes": "Please deliver before 6 PM",
    "statusHistory": [
      {
        "status": "pending",
        "note": "Order placed",
        "createdAt": "2026-03-26T10:00:00Z",
        "createdBy": "Customer"
      }
    ],
    "createdAt": "2026-03-26T10:00:00Z",
    "updatedAt": "2026-03-26T10:00:00Z"
  }
}
```

### 3.4 Update Order Status

```http
PATCH /orders/:id/status
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "status": "confirmed",
  "note": "Order confirmed and ready for processing"
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "id": 1001,
    "status": "confirmed",
    "updatedAt": "2026-03-26T10:30:00Z"
  },
  "message": "Order status updated successfully"
}
```

### 3.5 Cancel Order

```http
POST /orders/:id/cancel
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "reason": "Customer requested cancellation",
  "refund": true
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "id": 1001,
    "status": "cancelled",
    "refundAmount": 4226.46,
    "cancelledAt": "2026-03-26T11:00:00Z"
  },
  "message": "Order cancelled successfully"
}
```

---

## 4. Inventory Management

### 4.1 Get Inventory

```http
GET /inventory?warehouseId=1&productId=5&status=in_stock
```

**Query Parameters:**

```typescript
{
  warehouseId?: number;
  productId?: number;
  variantId?: number;
  status?: 'in_stock' | 'low_stock' | 'out_of_stock';
  page?: number;
  limit?: number;
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "warehouse": {
        "id": 1,
        "name": "Main Warehouse",
        "code": "WH-MAIN"
      },
      "variant": {
        "id": 1,
        "sku": "SHIRT-001-M-BLUE",
        "product": {
          "id": 1,
          "name": "Premium Formal Shirt"
        }
      },
      "quantity": 45,
      "reserved": 3,
      "available": 42,
      "minStock": 10,
      "maxStock": 100,
      "reorderPoint": 20,
      "status": "in_stock",
      "lastRestocked": "2026-03-20T10:00:00Z",
      "updatedAt": "2026-03-26T09:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 250
  }
}
```

### 4.2 Update Stock

```http
PATCH /inventory/:id
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "quantity": 50,
  "reason": "Stock replenishment",
  "reference": "PO-2026-0015"
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "quantity": 50,
    "available": 47,
    "updatedAt": "2026-03-26T10:00:00Z"
  },
  "message": "Stock updated successfully"
}
```

### 4.3 Stock Movement History

```http
GET /inventory/:id/movements?page=1&limit=20
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": [
    {
      "id": 101,
      "type": "in",
      "quantity": 50,
      "reason": "Purchase Order",
      "reference": "PO-2026-0015",
      "balanceBefore": 0,
      "balanceAfter": 50,
      "createdBy": {
        "id": 3,
        "name": "Admin User"
      },
      "createdAt": "2026-03-26T10:00:00Z"
    },
    {
      "id": 102,
      "type": "out",
      "quantity": 3,
      "reason": "Order Fulfillment",
      "reference": "ORD-2026-001001",
      "balanceBefore": 50,
      "balanceAfter": 47,
      "createdBy": {
        "id": 4,
        "name": "Warehouse Manager"
      },
      "createdAt": "2026-03-26T11:00:00Z"
    }
  ]
}
```

---

## 5. Manufacturing Operations

### 5.1 Create Work Order

```http
POST /manufacturing/work-orders
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "workOrderNumber": "WO-2026-0001",
  "productId": 1,
  "variantId": 1,
  "quantity": 100,
  "priority": "high",
  "startDate": "2026-03-27",
  "dueDate": "2026-04-10",
  "notes": "Rush order for premium formal shirts"
}
```

**Response: `201 Created`**

```json
{
  "success": true,
  "data": {
    "id": 1,
    "workOrderNumber": "WO-2026-0001",
    "product": {
      "id": 1,
      "name": "Premium Formal Shirt",
      "sku": "SHIRT-001-M-BLUE"
    },
    "quantity": 100,
    "producedQuantity": 0,
    "status": "pending",
    "priority": "high",
    "startDate": "2026-03-27",
    "dueDate": "2026-04-10",
    "createdAt": "2026-03-26T10:00:00Z"
  },
  "message": "Work order created successfully"
}
```

### 5.2 Get Work Orders

```http
GET /manufacturing/work-orders?status=in_progress&page=1
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "workOrderNumber": "WO-2026-0001",
      "product": {
        "id": 1,
        "name": "Premium Formal Shirt"
      },
      "quantity": 100,
      "producedQuantity": 35,
      "progress": 35,
      "status": "in_progress",
      "priority": "high",
      "dueDate": "2026-04-10",
      "currentStage": {
        "name": "Cutting",
        "progress": 85
      }
    }
  ]
}
```

### 5.3 Create Purchase Order

```http
POST /manufacturing/purchase-orders
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "supplierId": 5,
  "items": [
    {
      "materialId": 10,
      "quantity": 500,
      "unitPrice": 150.00
    }
  ],
  "deliveryDate": "2026-04-05",
  "notes": "Premium cotton fabric"
}
```

**Response: `201 Created`**

```json
{
  "success": true,
  "data": {
    "id": 15,
    "poNumber": "PO-2026-0015",
    "supplier": {
      "id": 5,
      "name": "Cotton Suppliers Ltd"
    },
    "items": [
      {
        "material": {
          "id": 10,
          "name": "Premium Cotton Fabric"
        },
        "quantity": 500,
        "unitPrice": 150.00,
        "total": 75000.00
      }
    ],
    "subtotal": 75000.00,
    "tax": 13500.00,
    "total": 88500.00,
    "status": "pending",
    "deliveryDate": "2026-04-05",
    "createdAt": "2026-03-26T10:00:00Z"
  }
}
```

---

## 6. CRM & Customer Management

### 6.1 Get Customers

```http
GET /customers?segment=vip&page=1&limit=20
Authorization: Bearer <access_token>
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": [
    {
      "id": 10,
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com",
      "phone": "+919876543210",
      "segment": "vip",
      "totalOrders": 25,
      "totalSpent": 125000.00,
      "lifetimeValue": 145000.00,
      "loyaltyPoints": 2500,
      "status": "active",
      "registeredAt": "2025-06-15T10:00:00Z",
      "lastOrderAt": "2026-03-20T14:30:00Z"
    }
  ]
}
```

### 6.2 Get Customer Details

```http
GET /customers/:id
Authorization: Bearer <access_token>
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "id": 10,
    "firstName": "John",
    "lastName": "Doe",
    "email": "john@example.com",
    "phone": "+919876543210",
    "dateOfBirth": "1990-05-15",
    "gender": "male",
    "segment": "vip",
    "addresses": [
      {
        "id": 1,
        "type": "home",
        "addressLine1": "123 Main Street",
        "city": "Mumbai",
        "state": "Maharashtra",
        "pincode": "400001",
        "isDefault": true
      }
    ],
    "stats": {
      "totalOrders": 25,
      "completedOrders": 23,
      "cancelledOrders": 2,
      "totalSpent": 125000.00,
      "averageOrderValue": 5000.00,
      "lifetimeValue": 145000.00,
      "loyaltyPoints": 2500
    },
    "preferences": {
      "newsletter": true,
      "smsNotifications": true,
      "preferredLanguage": "en"
    },
    "status": "active",
    "registeredAt": "2025-06-15T10:00:00Z",
    "lastOrderAt": "2026-03-20T14:30:00Z"
  }
}
```

### 6.3 Create Support Ticket

```http
POST /tickets
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "customerId": 10,
  "subject": "Product Quality Issue",
  "description": "The shirt I received has a stitching defect",
  "priority": "high",
  "category": "quality",
  "orderId": 1001
}
```

**Response: `201 Created`**

```json
{
  "success": true,
  "data": {
    "id": 501,
    "ticketNumber": "TKT-2026-0501",
    "customer": {
      "id": 10,
      "name": "John Doe"
    },
    "subject": "Product Quality Issue",
    "status": "open",
    "priority": "high",
    "assignedTo": null,
    "createdAt": "2026-03-26T10:00:00Z"
  },
  "message": "Ticket created successfully"
}
```

---

## 7. Analytics & Reporting

### 7.1 Dashboard Metrics

```http
GET /analytics/dashboard?dateFrom=2026-03-01&dateTo=2026-03-26
Authorization: Bearer <access_token>
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "sales": {
      "total": 2450000.00,
      "orders": 450,
      "averageOrderValue": 5444.44,
      "growth": 15.5
    },
    "revenue": {
      "total": 2450000.00,
      "online": 1850000.00,
      "offline": 600000.00
    },
    "customers": {
      "total": 1250,
      "new": 85,
      "returning": 365,
      "growth": 8.2
    },
    "inventory": {
      "totalProducts": 850,
      "lowStock": 45,
      "outOfStock": 12,
      "totalValue": 15500000.00
    },
    "manufacturing": {
      "activeWorkOrders": 25,
      "completed": 180,
      "pending": 15
    }
  }
}
```

### 7.2 Sales Report

```http
GET /analytics/sales?dateFrom=2026-03-01&dateTo=2026-03-26&groupBy=day
Authorization: Bearer <access_token>
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": [
    {
      "date": "2026-03-01",
      "orders": 15,
      "revenue": 75000.00,
      "profit": 22500.00,
      "customers": 12
    },
    {
      "date": "2026-03-02",
      "orders": 18,
      "revenue": 90000.00,
      "profit": 27000.00,
      "customers": 15
    }
  ],
  "summary": {
    "totalOrders": 450,
    "totalRevenue": 2450000.00,
    "totalProfit": 735000.00,
    "averageOrderValue": 5444.44
  }
}
```

---

## 8. Marketplace Integration

### 8.1 Sync Product to Marketplace

```http
POST /marketplace/sync
Authorization: Bearer <access_token>
```

**Request Body:**

```json
{
  "productId": 1,
  "marketplaces": ["amazon", "flipkart"],
  "autoSync": true
}
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": {
    "productId": 1,
    "synced": [
      {
        "marketplace": "amazon",
        "listingId": "AMZN-SHIRT-001",
        "status": "active",
        "syncedAt": "2026-03-26T10:00:00Z"
      },
      {
        "marketplace": "flipkart",
        "listingId": "FLPK-SHIRT-001",
        "status": "pending",
        "syncedAt": "2026-03-26T10:00:15Z"
      }
    ]
  },
  "message": "Product synced to marketplaces"
}
```

### 8.2 Get Marketplace Orders

```http
GET /marketplace/orders?marketplace=amazon&status=pending
Authorization: Bearer <access_token>
```

**Response: `200 OK`**

```json
{
  "success": true,
  "data": [
    {
      "id": 2001,
      "marketplace": "amazon",
      "marketplaceOrderId": "AMZ-ORD-123456",
      "status": "pending",
      "items": [
        {
          "listingId": "AMZN-SHIRT-001",
          "productName": "Premium Formal Shirt",
          "quantity": 2,
          "price": 1299.00
        }
      ],
      "total": 2698.00,
      "customer": {
        "name": "Marketplace Customer",
        "phone": "+919876543210"
      },
      "createdAt": "2026-03-26T09:00:00Z"
    }
  ]
}
```

---

## Rate Limiting

```yaml
Rate Limits:
  General Endpoints: 100 requests / 15 minutes
  Auth Endpoints: 5 requests / 15 minutes
  Order Creation: 10 requests / minute
  Search: 30 requests / minute

Headers:
  X-Rate-Limit-Limit: Maximum requests allowed
  X-Rate-Limit-Remaining: Remaining requests
  X-Rate-Limit-Reset: Unix timestamp when limit resets

Error Response (429):
  {
    "success": false,
    "error": {
      "code": "RATE_LIMIT_EXCEEDED",
      "message": "Too many requests, please try again later",
      "retryAfter": 300
    }
  }
```

---

## Webhooks

### Webhook Events

```yaml
Available Events:
  - order.created
  - order.updated
  - order.cancelled
  - payment.completed
  - payment.failed
  - inventory.low_stock
  - product.created
  - product.updated
  - shipment.created
  - shipment.delivered
```

### Webhook Payload

```json
{
  "id": "evt_123456",
  "event": "order.created",
  "timestamp": "2026-03-26T10:00:00Z",
  "data": {
    "orderId": 1001,
    "orderNumber": "ORD-2026-001001",
    "total": 4226.46,
    "status": "pending"
  }
}
```

---

**Document Version:** 3.0  
**Last Updated:** March 26, 2026  
**Next Review:** April 26, 2026  
**Status:** ✅ Production-Ready

---

[← Back to Index](./00_MASTER_INDEX.md) | [Next: Project Flow →](./06_PROJECT_FLOW.md)
