# 03. Backend Architecture
## Enterprise Men's Clothing Manufacturing & Sales Platform

**Version:** 2.0  
**Last Updated:** March 26, 2026  
**Stack:** Node.js + Express.js + TypeScript  
**Status:** ✅ Production-Ready

---

> **💡 FREE TIER CONFIGURATION:**  
> This document includes Elasticsearch examples, but for **free tier deployment**:
> - Use **PostgreSQL Full-Text Search** instead of Elasticsearch (saves $30-50/month)
> - Use **Cloudinary SDK** instead of AWS S3 SDK (free tier: 25GB)
> - Use **Upstash Redis** with REST API (free tier: 10K/day)
> - All code examples are adaptable - see notes in each section
>
> See [09_FREE_TIER_DEPLOYMENT.md](./09_FREE_TIER_DEPLOYMENT.md) for free tier setup.

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Project Structure](#project-structure)
3. [Layer Architecture](#layer-architecture)
4. [Module Details](#module-details)
5. [Graphile Worker Integration](#graphile-worker-integration)
6. [Cron Jobs](#cron-jobs)
7. [Error Handling](#error-handling)
8. [Security Implementation](#security-implementation)

---

## 1. Architecture Overview

### 1.1 Architecture Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│                    (React.js Frontend)                           │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTP/HTTPS
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Presentation Layer                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Express.js Middleware                        │  │
│  │  • CORS  • Helmet  • Rate Limiter  • Body Parser         │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Routes / Controllers                    │  │
│  │  • Input Validation  • Request Handling  • Response      │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Business Layer                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Service Layer                          │  │
│  │  • Business Logic  • Validation  • Orchestration         │  │
│  │  • Transaction Management  • Cache Management            │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Data Access Layer                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                 Repository Pattern                        │  │
│  │  • Database Queries  • ORM Operations  • Data Mapping    │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Database Layer                              │
│         PostgreSQL + Redis + Elasticsearch                       │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Design Principles

```yaml
SOLID Principles:
  S: Single Responsibility - Each class has one reason to change
  O: Open/Closed - Open for extension, closed for modification
  L: Liskov Substitution - Derived classes are substitutable
  I: Interface Segregation - Many specific interfaces
  D: Dependency Inversion - Depend on abstractions

Additional Principles:
  - DRY (Don't Repeat Yourself)
  - KISS (Keep It Simple, Stupid)
  - YAGNI (You Aren't Gonna Need It)
  - Separation of Concerns
  - Dependency Injection
  - Repository Pattern
  - Service Layer Pattern
```

---

## 2. Project Structure

### 2.1 Directory Structure

```
backend/
├── src/
│   ├── config/                    # Configuration files
│   │   ├── database.ts           # Database configuration
│   │   ├── redis.ts              # Redis configuration
│   │   ├── elasticsearch.ts      # Elasticsearch configuration
│   │   ├── aws.ts                # AWS SDK configuration
│   │   ├── constants.ts          # Application constants
│   │   └── index.ts              # Export all configs
│   │
│   ├── modules/                   # Feature modules
│   │   ├── auth/                 # Authentication module
│   │   │   ├── controllers/      # Route controllers
│   │   │   │   ├── auth.controller.ts
│   │   │   │   └── index.ts
│   │   │   ├── services/         # Business logic
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── jwt.service.ts
│   │   │   │   └── index.ts
│   │   │   ├── repositories/     # Data access
│   │   │   │   ├── user.repository.ts
│   │   │   │   ├── session.repository.ts
│   │   │   │   └── index.ts
│   │   │   ├── validators/       # Input validation
│   │   │   │   ├── auth.validator.ts
│   │   │   │   └── index.ts
│   │   │   ├── routes/           # Route definitions
│   │   │   │   ├── auth.routes.ts
│   │   │   │   └── index.ts
│   │   │   ├── types/            # TypeScript types
│   │   │   │   ├── auth.types.ts
│   │   │   │   └── index.ts
│   │   │   └── index.ts          # Module exports
│   │   │
│   │   ├── products/             # Product management
│   │   │   ├── controllers/
│   │   │   │   ├── product.controller.ts
│   │   │   │   ├── category.controller.ts
│   │   │   │   └── variant.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── product.service.ts
│   │   │   │   ├── category.service.ts
│   │   │   │   ├── variant.service.ts
│   │   │   │   └── pricing.service.ts
│   │   │   ├── repositories/
│   │   │   │   ├── product.repository.ts
│   │   │   │   ├── category.repository.ts
│   │   │   │   └── variant.repository.ts
│   │   │   ├── validators/
│   │   │   ├── routes/
│   │   │   ├── types/
│   │   │   └── index.ts
│   │   │
│   │   ├── inventory/            # Inventory management
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── repositories/
│   │   │   ├── validators/
│   │   │   ├── routes/
│   │   │   └── index.ts
│   │   │
│   │   ├── orders/               # Order management
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── repositories/
│   │   │   ├── validators/
│   │   │   ├── routes/
│   │   │   └── index.ts
│   │   │
│   │   ├── payments/             # Payment processing
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── repositories/
│   │   │   ├── gateways/        # Payment gateway integrations
│   │   │   │   ├── stripe.gateway.ts
│   │   │   │   ├── razorpay.gateway.ts
│   │   │   │   └── index.ts
│   │   │   ├── validators/
│   │   │   ├── routes/
│   │   │   └── index.ts
│   │   │
│   │   ├── manufacturing/        # Manufacturing module
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── repositories/
│   │   │   ├── validators/
│   │   │   ├── routes/
│   │   │   └── index.ts
│   │   │
│   │   ├── crm/                  # Customer relationship
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── repositories/
│   │   │   ├── validators/
│   │   │   ├── routes/
│   │   │   └── index.ts
│   │   │
│   │   ├── analytics/            # Analytics & reporting
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── repositories/
│   │   │   ├── validators/
│   │   │   ├── routes/
│   │   │   └── index.ts
│   │   │
│   │   ├── notifications/        # Notification service
│   │   │   ├── controllers/
│   │   │   ├── services/
│   │   │   ├── channels/        # Email, SMS, Push
│   │   │   │   ├── email.channel.ts
│   │   │   │   ├── sms.channel.ts
│   │   │   │   ├── push.channel.ts
│   │   │   │   └── index.ts
│   │   │   ├── templates/       # Email templates
│   │   │   ├── validators/
│   │   │   ├── routes/
│   │   │   └── index.ts
│   │   │
│   │   └── marketplace/          # Marketplace integrations
│   │       ├── controllers/
│   │       ├── services/
│   │       ├── repositories/
│   │       ├── integrations/    # Amazon, Flipkart, etc.
│   │       │   ├── amazon.integration.ts
│   │       │   ├── flipkart.integration.ts
│   │       │   └── index.ts
│   │       ├── validators/
│   │       ├── routes/
│   │       └── index.ts
│   │
│   ├── shared/                   # Shared utilities
│   │   ├── middleware/          # Express middleware
│   │   │   ├── auth.middleware.ts
│   │   │   ├── error.middleware.ts
│   │   │   ├── validation.middleware.ts
│   │   │   ├── ratelimit.middleware.ts
│   │   │   ├── logging.middleware.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── utils/               # Utility functions
│   │   │   ├── encryption.util.ts
│   │   │   ├── date.util.ts
│   │   │   ├── file.util.ts
│   │   │   ├── pagination.util.ts
│   │   │   ├── response.util.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── helpers/             # Helper functions
│   │   │   ├── query.helper.ts
│   │   │   ├── cache.helper.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── types/               # Global types
│   │   │   ├── express.d.ts     # Express type extensions
│   │   │   ├── common.types.ts
│   │   │   └── index.ts
│   │   │
│   │   ├── enums/               # Enumerations
│   │   │   ├── status.enum.ts
│   │   │   ├── roles.enum.ts
│   │   │   └── index.ts
│   │   │
│   │   └── errors/              # Custom error classes
│   │       ├── app.error.ts
│   │       ├── validation.error.ts
│   │       ├── not-found.error.ts
│   │       └── index.ts
│   │
│   ├── workers/                  # Graphile Worker jobs
│   │   ├── tasks/               # Background tasks
│   │   │   ├── order-processing.task.ts
│   │   │   ├── email-notification.task.ts
│   │   │   ├── sms-notification.task.ts
│   │   │   ├── marketplace-sync.task.ts
│   │   │   ├── image-processing.task.ts
│   │   │   ├── report-generation.task.ts
│   │   │   └── index.ts
│   │   ├── crons/               # Scheduled jobs
│   │   │   ├── daily-metrics.cron.ts
│   │   │   ├── inventory-check.cron.ts
│   │   │   ├── abandoned-cart.cron.ts
│   │   │   ├── subscription-renewal.cron.ts
│   │   │   └── index.ts
│   │   ├── worker.ts            # Worker initialization
│   │   └── index.ts
│   │
│   ├── database/                 # Database related
│   │   ├── migrations/          # Prisma migrations
│   │   ├── seeds/               # Database seeders
│   │   │   ├── roles.seed.ts
│   │   │   ├── permissions.seed.ts
│   │   │   ├── users.seed.ts
│   │   │   └── index.ts
│   │   ├── schema.prisma        # Prisma schema
│   │   └── client.ts            # Prisma client instance
│   │
│   ├── docs/                     # API documentation
│   │   ├── swagger.ts           # Swagger configuration
│   │   └── api-spec.yaml        # OpenAPI specification
│   │
│   ├── app.ts                    # Express app setup
│   ├── server.ts                 # Server entry point
│   └── index.ts                  # Main entry point
│
├── tests/                        # Test files
│   ├── unit/                    # Unit tests
│   │   ├── services/
│   │   ├── repositories/
│   │   └── utils/
│   ├── integration/             # Integration tests
│   │   ├── api/
│   │   └── database/
│   ├── e2e/                     # End-to-end tests
│   │   └── scenarios/
│   ├── fixtures/                # Test data
│   │   └── data/
│   └── setup.ts                 # Test setup
│
├── scripts/                      # Utility scripts
│   ├── seed-db.ts               # Database seeding
│   ├── generate-keys.ts         # Generate JWT keys
│   └── backup.ts                # Backup script
│
├── .env.example                  # Environment variables template
├── .env                          # Environment variables (gitignored)
├── .gitignore                    # Git ignore rules
├── .eslintrc.js                  # ESLint configuration
├── .prettierrc                   # Prettier configuration
├── tsconfig.json                 # TypeScript configuration
├── package.json                  # Dependencies
├── docker-compose.yml            # Docker compose for local dev
├── Dockerfile                    # Docker configuration
└── README.md                     # Project documentation
```

---

## 3. Layer Architecture

### 3.1 Controller Layer

**Responsibility:** Handle HTTP requests and responses

```typescript
// src/modules/products/controllers/product.controller.ts

import { Request, Response, NextFunction } from 'express';
import { ProductService } from '../services/product.service';
import { ApiResponse } from '../../../shared/utils/response.util';
import { ProductCreateDto, ProductUpdateDto } from '../types/product.types';

export class ProductController {
  constructor(private productService: ProductService) {}

  /**
   * Get all products with pagination and filters
   */
  async getProducts(req: Request, res: Response, next: NextFunction) {
    try {
      const { page = 1, limit = 20, category, status, search } = req.query;

      const result = await this.productService.getProducts({
        page: Number(page),
        limit: Number(limit),
        category: category as string,
        status: status as string,
        search: search as string,
      });

      return ApiResponse.success(res, result, 'Products retrieved successfully');
    } catch (error) {
      next(error);
    }
  }

  /**
   * Get single product by ID
   */
  async getProductById(req: Request, res: Response, next: NextFunction) {
    try {
      const { id } = req.params;
      
      const product = await this.productService.getProductById(Number(id));

      return ApiResponse.success(res, product, 'Product retrieved successfully');
    } catch (error) {
      next(error);
    }
  }

  /**
   * Create new product
   */
  async createProduct(req: Request, res: Response, next: NextFunction) {
    try {
      const productData: ProductCreateDto = req.body;
      const userId = req.user!.id;

      const product = await this.productService.createProduct(productData, userId);

      return ApiResponse.created(res, product, 'Product created successfully');
    } catch (error) {
      next(error);
    }
  }

  /**
   * Update product
   */
  async updateProduct(req: Request, res: Response, next: NextFunction) {
    try {
      const { id } = req.params;
      const productData: ProductUpdateDto = req.body;
      const userId = req.user!.id;

      const product = await this.productService.updateProduct(
        Number(id),
        productData,
        userId
      );

      return ApiResponse.success(res, product, 'Product updated successfully');
    } catch (error) {
      next(error);
    }
  }

  /**
   * Delete product
   */
  async deleteProduct(req: Request, res: Response, next: NextFunction) {
    try {
      const { id } = req.params;
      const userId = req.user!.id;

      await this.productService.deleteProduct(Number(id), userId);

      return ApiResponse.success(res, null, 'Product deleted successfully');
    } catch (error) {
      next(error);
    }
  }

  /**
   * Bulk upload products
   */
  async bulkUpload(req: Request, res: Response, next: NextFunction) {
    try {
      const file = req.file;
      const userId = req.user!.id;

      if (!file) {
        throw new ValidationError('File is required');
      }

      const result = await this.productService.bulkUpload(file, userId);

      return ApiResponse.success(res, result, 'Products uploaded successfully');
    } catch (error) {
      next(error);
    }
  }
}
```

### 3.2 Service Layer

**Responsibility:** Business logic and orchestration

```typescript
// src/modules/products/services/product.service.ts

import { ProductRepository } from '../repositories/product.repository';
import { CategoryRepository } from '../repositories/category.repository';
import { InventoryRepository } from '../../inventory/repositories/inventory.repository';
import { CacheHelper } from '../../../shared/helpers/cache.helper';
import { ElasticsearchService } from '../../../shared/services/elasticsearch.service';
import { NotFoundError, ValidationError } from '../../../shared/errors';
import { ProductCreateDto, ProductUpdateDto, ProductFilterDto } from '../types/product.types';
import { AuditLogService } from '../../../shared/services/audit-log.service';
import { GraphileWorker } from '../../../workers';

export class ProductService {
  constructor(
    private productRepository: ProductRepository,
    private categoryRepository: CategoryRepository,
    private inventoryRepository: InventoryRepository,
    private cacheHelper: CacheHelper,
    private elasticsearchService: ElasticsearchService,
    private auditLogService: AuditLogService,
    private worker: GraphileWorker
  ) {}

  /**
   * Get products with pagination and filters
   */
  async getProducts(filters: ProductFilterDto) {
    const cacheKey = `products:${JSON.stringify(filters)}`;
    
    // Try to get from cache
    const cached = await this.cacheHelper.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // If search query exists, use Elasticsearch
    if (filters.search) {
      const result = await this.elasticsearchService.searchProducts(filters);
      await this.cacheHelper.set(cacheKey, JSON.stringify(result), 300); // 5 min
      return result;
    }

    // Otherwise, use database
    const result = await this.productRepository.findMany(filters);
    
    // Cache the result
    await this.cacheHelper.set(cacheKey, JSON.stringify(result), 300);
    
    return result;
  }

  /**
   * Get product by ID
   */
  async getProductById(id: number) {
    const cacheKey = `product:${id}`;
    
    // Try cache first
    const cached = await this.cacheHelper.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    const product = await this.productRepository.findById(id);
    
    if (!product) {
      throw new NotFoundError(`Product with ID ${id} not found`);
    }

    // Get related data
    const [variants, images, inventory] = await Promise.all([
      this.productRepository.getVariants(id),
      this.productRepository.getImages(id),
      this.inventoryRepository.getByProductId(id),
    ]);

    const result = {
      ...product,
      variants,
      images,
      inventory,
    };

    // Cache for 1 hour
    await this.cacheHelper.set(cacheKey, JSON.stringify(result), 3600);
    
    return result;
  }

  /**
   * Create new product
   */
  async createProduct(data: ProductCreateDto, userId: number) {
    // Validate category exists
    const category = await this.categoryRepository.findById(data.categoryId);
    if (!category) {
      throw new ValidationError('Invalid category');
    }

    // Check if SKU already exists
    const existingProduct = await this.productRepository.findBySku(data.sku);
    if (existingProduct) {
      throw new ValidationError('Product with this SKU already exists');
    }

    // Start transaction
    const product = await this.productRepository.create({
      ...data,
      createdBy: userId,
    });

    // Create variants if provided
    if (data.variants && data.variants.length > 0) {
      await this.productRepository.createVariants(product.id, data.variants);
    }

    // Index in Elasticsearch
    await this.elasticsearchService.indexProduct(product);

    // Create audit log
    await this.auditLogService.log({
      userId,
      action: 'create',
      resourceType: 'product',
      resourceId: product.id,
      newValues: product,
    });

    // Queue image processing job if images provided
    if (data.images && data.images.length > 0) {
      await this.worker.addJob('process-product-images', {
        productId: product.id,
        images: data.images,
      });
    }

    // Clear related caches
    await this.cacheHelper.delete('products:*');

    return product;
  }

  /**
   * Update product
   */
  async updateProduct(id: number, data: ProductUpdateDto, userId: number) {
    const existingProduct = await this.productRepository.findById(id);
    
    if (!existingProduct) {
      throw new NotFoundError(`Product with ID ${id} not found`);
    }

    // If category is being changed, validate it
    if (data.categoryId && data.categoryId !== existingProduct.categoryId) {
      const category = await this.categoryRepository.findById(data.categoryId);
      if (!category) {
        throw new ValidationError('Invalid category');
      }
    }

    // Update product
    const updatedProduct = await this.productRepository.update(id, data);

    // Update Elasticsearch index
    await this.elasticsearchService.updateProduct(id, updatedProduct);

    // Create audit log
    await this.auditLogService.log({
      userId,
      action: 'update',
      resourceType: 'product',
      resourceId: id,
      oldValues: existingProduct,
      newValues: updatedProduct,
    });

    // Clear caches
    await this.cacheHelper.delete(`product:${id}`);
    await this.cacheHelper.delete('products:*');

    return updatedProduct;
  }

  /**
   * Delete product
   */
  async deleteProduct(id: number, userId: number) {
    const product = await this.productRepository.findById(id);
    
    if (!product) {
      throw new NotFoundError(`Product with ID ${id} not found`);
    }

    // Check if product has active orders
    const hasActiveOrders = await this.productRepository.hasActiveOrders(id);
    if (hasActiveOrders) {
      throw new ValidationError('Cannot delete product with active orders');
    }

    // Soft delete
    await this.productRepository.softDelete(id);

    // Remove from Elasticsearch
    await this.elasticsearchService.deleteProduct(id);

    // Create audit log
    await this.auditLogService.log({
      userId,
      action: 'delete',
      resourceType: 'product',
      resourceId: id,
      oldValues: product,
    });

    // Clear caches
    await this.cacheHelper.delete(`product:${id}`);
    await this.cacheHelper.delete('products:*');
  }

  /**
   * Bulk upload products from CSV
   */
  async bulkUpload(file: Express.Multer.File, userId: number) {
    // Queue bulk upload job
    const job = await this.worker.addJob('bulk-product-upload', {
      fileUrl: file.path,
      userId,
    });

    return {
      jobId: job.id,
      message: 'Bulk upload started. You will be notified upon completion.',
    };
  }

  /**
   * Get product analytics
   */
  async getProductAnalytics(productId: number, dateRange: { from: Date; to: Date }) {
    return this.productRepository.getAnalytics(productId, dateRange);
  }
}
```

### 3.3 Repository Layer

**Responsibility:** Data access and database operations

```typescript
// src/modules/products/repositories/product.repository.ts

import { PrismaClient } from '@prisma/client';
import { ProductFilterDto } from '../types/product.types';

export class ProductRepository {
  constructor(private prisma: PrismaClient) {}

  /**
   * Find many products with filters
   */
  async findMany(filters: ProductFilterDto) {
    const { page = 1, limit = 20, category, status, search } = filters;
    const skip = (page - 1) * limit;

    const where: any = {};

    if (category) {
      where.categoryId = Number(category);
    }

    if (status) {
      where.status = status;
    }

    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { sku: { contains: search, mode: 'insensitive' } },
        { description: { contains: search, mode: 'insensitive' } },
      ];
    }

    const [products, total] = await Promise.all([
      this.prisma.product.findMany({
        where,
        skip,
        take: limit,
        include: {
          category: true,
          variants: {
            select: {
              id: true,
              sku: true,
              size: true,
              color: true,
              retailPrice: true,
              isActive: true,
            },
          },
          images: {
            where: { isPrimary: true },
            take: 1,
          },
        },
        orderBy: { createdAt: 'desc' },
      }),
      this.prisma.product.count({ where }),
    ]);

    return {
      data: products,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
        hasNext: page * limit < total,
        hasPrev: page > 1,
      },
    };
  }

  /**
   * Find product by ID
   */
  async findById(id: number) {
    return this.prisma.product.findUnique({
      where: { id },
      include: {
        category: true,
        createdBy: {
          select: {
            id: true,
            firstName: true,
            lastName: true,
            email: true,
          },
        },
      },
    });
  }

  /**
   * Find product by SKU
   */
  async findBySku(sku: string) {
    return this.prisma.product.findUnique({
      where: { sku },
    });
  }

  /**
   * Create product
   */
  async create(data: any) {
    return this.prisma.product.create({
      data,
      include: {
        category: true,
      },
    });
  }

  /**
   * Update product
   */
  async update(id: number, data: any) {
    return this.prisma.product.update({
      where: { id },
      data: {
        ...data,
        updatedAt: new Date(),
      },
      include: {
        category: true,
      },
    });
  }

  /**
   * Soft delete product
   */
  async softDelete(id: number) {
    return this.prisma.product.update({
      where: { id },
      data: {
        status: 'discontinued',
        updatedAt: new Date(),
      },
    });
  }

  /**
   * Get product variants
   */
  async getVariants(productId: number) {
    return this.prisma.productVariant.findMany({
      where: { productId },
      include: {
        inventory: {
          select: {
            warehouseId: true,
            quantity: true,
            reservedQuantity: true,
          },
        },
      },
    });
  }

  /**
   * Create product variants
   */
  async createVariants(productId: number, variants: any[]) {
    const data = variants.map((variant) => ({
      ...variant,
      productId,
    }));

    return this.prisma.productVariant.createMany({
      data,
    });
  }

  /**
   * Get product images
   */
  async getImages(productId: number) {
    return this.prisma.productImage.findMany({
      where: { productId },
      orderBy: [{ isPrimary: 'desc' }, { sortOrder: 'asc' }],
    });
  }

  /**
   * Check if product has active orders
   */
  async hasActiveOrders(productId: number): Promise<boolean> {
    const count = await this.prisma.orderItem.count({
      where: {
        variant: {
          productId,
        },
        order: {
          status: {
            in: ['pending', 'confirmed', 'processing', 'packed', 'shipped'],
          },
        },
      },
    });

    return count > 0;
  }

  /**
   * Get product analytics
   */
  async getAnalytics(productId: number, dateRange: { from: Date; to: Date }) {
    const [totalOrders, totalRevenue, totalUnits] = await Promise.all([
      // Total orders
      this.prisma.orderItem.count({
        where: {
          variant: { productId },
          order: {
            createdAt: {
              gte: dateRange.from,
              lte: dateRange.to,
            },
            status: 'delivered',
          },
        },
      }),

      // Total revenue
      this.prisma.orderItem.aggregate({
        where: {
          variant: { productId },
          order: {
            createdAt: {
              gte: dateRange.from,
              lte: dateRange.to,
            },
            status: 'delivered',
          },
        },
        _sum: {
          totalAmount: true,
        },
      }),

      // Total units sold
      this.prisma.orderItem.aggregate({
        where: {
          variant: { productId },
          order: {
            createdAt: {
              gte: dateRange.from,
              lte: dateRange.to,
            },
            status: 'delivered',
          },
        },
        _sum: {
          quantity: true,
        },
      }),
    ]);

    return {
      totalOrders,
      totalRevenue: totalRevenue._sum.totalAmount || 0,
      totalUnits: totalUnits._sum.quantity || 0,
      avgOrderValue: totalOrders > 0 
        ? (totalRevenue._sum.totalAmount || 0) / totalOrders 
        : 0,
    };
  }
}
```

---

## 4. Module Details

### 4.1 Authentication Module

```typescript
// src/modules/auth/services/auth.service.ts

import bcrypt from 'bcrypt';
import { UserRepository } from '../repositories/user.repository';
import { SessionRepository } from '../repositories/session.repository';
import { JwtService } from './jwt.service';
import { UnauthorizedError, ValidationError } from '../../../shared/errors';
import { LoginDto, RegisterDto, TokenPayload } from '../types/auth.types';

export class AuthService {
  constructor(
    private userRepository: UserRepository,
    private sessionRepository: SessionRepository,
    private jwtService: JwtService
  ) {}

  /**
   * Register new user
   */
  async register(data: RegisterDto) {
    // Check if email already exists
    const existingUser = await this.userRepository.findByEmail(data.email);
    if (existingUser) {
      throw new ValidationError('Email already registered');
    }

    // Hash password
    const passwordHash = await bcrypt.hash(data.password, 10);

    // Create user
    const user = await this.userRepository.create({
      ...data,
      passwordHash,
    });

    // Assign default role
    await this.userRepository.assignRole(user.id, 'customer');

    // Generate tokens
    const { accessToken, refreshToken } = await this.generateTokens(user);

    return {
      user: this.sanitizeUser(user),
      accessToken,
      refreshToken,
    };
  }

  /**
   * Login user
   */
  async login(data: LoginDto, deviceInfo: any) {
    // Find user
    const user = await this.userRepository.findByEmail(data.email);
    if (!user) {
      throw new UnauthorizedError('Invalid credentials');
    }

    // Check account status
    if (user.status !== 'active') {
      throw new UnauthorizedError('Account is not active');
    }

    // Check if account is locked
    if (user.lockedUntil && user.lockedUntil > new Date()) {
      throw new UnauthorizedError('Account is temporarily locked. Please try again later.');
    }

    // Verify password
    const isPasswordValid = await bcrypt.compare(data.password, user.passwordHash);
    
    if (!isPasswordValid) {
      // Increment failed login attempts
      await this.userRepository.incrementFailedLoginAttempts(user.id);
      
      // Lock account if too many failed attempts
      if (user.failedLoginAttempts >= 4) {
        await this.userRepository.lockAccount(user.id, 30); // Lock for 30 minutes
      }
      
      throw new UnauthorizedError('Invalid credentials');
    }

    // Reset failed login attempts
    await this.userRepository.resetFailedLoginAttempts(user.id);

    // Update last login
    await this.userRepository.updateLastLogin(user.id);

    // Generate tokens
    const { accessToken, refreshToken } = await this.generateTokens(user, deviceInfo);

    return {
      user: this.sanitizeUser(user),
      accessToken,
      refreshToken,
    };
  }

  /**
   * Refresh access token
   */
  async refreshToken(refreshToken: string) {
    // Verify refresh token
    const payload = await this.jwtService.verifyRefreshToken(refreshToken);
    
    // Check if session exists
    const session = await this.sessionRepository.findByToken(refreshToken);
    if (!session || session.expiresAt < new Date()) {
      throw new UnauthorizedError('Invalid or expired refresh token');
    }

    // Get user
    const user = await this.userRepository.findById(payload.userId);
    if (!user || user.status !== 'active') {
      throw new UnauthorizedError('User not found or inactive');
    }

    // Generate new access token
    const accessToken = await this.jwtService.generateAccessToken({
      userId: user.id,
      email: user.email,
      roles: payload.roles,
    });

    return { accessToken };
  }

  /**
   * Logout user
   */
  async logout(refreshToken: string) {
    await this.sessionRepository.deleteByToken(refreshToken);
  }

  /**
   * Generate tokens
   */
  private async generateTokens(user: any, deviceInfo?: any) {
    // Get user roles and permissions
    const roles = await this.userRepository.getRoles(user.id);
    const permissions = await this.userRepository.getPermissions(user.id);

    const payload: TokenPayload = {
      userId: user.id,
      email: user.email,
      roles: roles.map((r) => r.name),
      permissions: permissions.map((p) => `${p.resource}:${p.action}`),
    };

    // Generate tokens
    const accessToken = await this.jwtService.generateAccessToken(payload);
    const refreshToken = await this.jwtService.generateRefreshToken(payload);

    // Store session
    await this.sessionRepository.create({
      userId: user.id,
      token: refreshToken,
      refreshToken,
      deviceInfo: deviceInfo ? JSON.stringify(deviceInfo) : null,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
    });

    return { accessToken, refreshToken };
  }

  /**
   * Remove sensitive data from user object
   */
  private sanitizeUser(user: any) {
    const { passwordHash, failedLoginAttempts, lockedUntil, ...sanitized } = user;
    return sanitized;
  }
}
```

### 4.2 Order Processing Module

```typescript
// src/modules/orders/services/order.service.ts

import { OrderRepository } from '../repositories/order.repository';
import { InventoryService } from '../../inventory/services/inventory.service';
import { PaymentService } from '../../payments/services/payment.service';
import { NotificationService } from '../../notifications/services/notification.service';
import { GraphileWorker } from '../../../workers';
import { ValidationError, InsufficientInventoryError } from '../../../shared/errors';
import { OrderCreateDto } from '../types/order.types';

export class OrderService {
  constructor(
    private orderRepository: OrderRepository,
    private inventoryService: InventoryService,
    private paymentService: PaymentService,
    private notificationService: NotificationService,
    private worker: GraphileWorker
  ) {}

  /**
   * Create new order
   */
  async createOrder(data: OrderCreateDto, customerId: number) {
    // Validate customer
    const customer = await this.orderRepository.getCustomer(customerId);
    if (!customer) {
      throw new ValidationError('Invalid customer');
    }

    // Check credit limit for wholesale customers
    if (customer.customerType === 'wholesale') {
      const availableCredit = customer.creditLimit - customer.creditUsed;
      if (data.totalAmount > availableCredit) {
        throw new ValidationError('Insufficient credit limit');
      }
    }

    // Validate inventory for all items
    for (const item of data.items) {
      const available = await this.inventoryService.checkAvailability(
        item.variantId,
        item.quantity
      );

      if (!available) {
        throw new InsufficientInventoryError(
          `Insufficient inventory for variant ${item.variantId}`
        );
      }
    }

    // Calculate order totals
    const subtotal = data.items.reduce((sum, item) => sum + item.total, 0);
    const taxAmount = subtotal * 0.18; // 18% GST
    const totalAmount = subtotal + taxAmount + (data.shippingCost || 0) - (data.discountAmount || 0);

    // Start database transaction
    const order = await this.orderRepository.transaction(async (tx) => {
      // Create order
      const newOrder = await tx.order.create({
        data: {
          customerId,
          orderNumber: this.generateOrderNumber(),
          channel: data.channel || 'website',
          status: 'pending',
          paymentStatus: 'pending',
          subtotal,
          taxAmount,
          shippingCost: data.shippingCost || 0,
          discountAmount: data.discountAmount || 0,
          discountCode: data.discountCode,
          totalAmount,
          currency: 'INR',
          notes: data.notes,
        },
      });

      // Create order items
      await tx.orderItem.createMany({
        data: data.items.map((item) => ({
          orderId: newOrder.id,
          variantId: item.variantId,
          quantity: item.quantity,
          unitPrice: item.unitPrice,
          discountAmount: item.discountAmount || 0,
          taxRate: 18,
          taxAmount: item.total * 0.18,
          totalAmount: item.total,
        })),
      });

      // Create order addresses
      await tx.orderAddress.createMany({
        data: [
          { ...data.billingAddress, orderId: newOrder.id, addressType: 'billing' },
          { ...data.shippingAddress, orderId: newOrder.id, addressType: 'shipping' },
        ],
      });

      // Reserve inventory
      for (const item of data.items) {
        await this.inventoryService.reserveStock(item.variantId, item.quantity, tx);
      }

      // Create order status history
      await tx.orderStatusHistory.create({
        data: {
          orderId: newOrder.id,
          status: 'pending',
          notes: 'Order created',
        },
      });

      return newOrder;
    });

    // Queue background jobs
    await Promise.all([
      // Send order confirmation email
      this.worker.addJob('send-order-confirmation', {
        orderId: order.id,
        customerId,
      }),

      // Process payment (if not COD)
      data.paymentMethod !== 'cod' &&
        this.worker.addJob('process-payment', {
          orderId: order.id,
          amount: totalAmount,
          method: data.paymentMethod,
        }),

      // Generate invoice
      this.worker.addJob('generate-invoice', {
        orderId: order.id,
      }),
    ]);

    return order;
  }

  /**
   * Update order status
   */
  async updateOrderStatus(
    orderId: number,
    status: string,
    userId: number,
    notes?: string
  ) {
    const order = await this.orderRepository.findById(orderId);
    
    if (!order) {
      throw new NotFoundError(`Order ${orderId} not found`);
    }

    // Validate status transition
    this.validateStatusTransition(order.status, status);

    // Update order status
    await this.orderRepository.updateStatus(orderId, status);

    // Add status history
    await this.orderRepository.addStatusHistory(orderId, status, userId, notes);

    // Handle status-specific logic
    await this.handleStatusChange(order, status);

    return this.orderRepository.findById(orderId);
  }

  /**
   * Cancel order
   */
  async cancelOrder(orderId: number, userId: number, reason: string) {
    const order = await this.orderRepository.findById(orderId);
    
    if (!order) {
      throw new NotFoundError(`Order ${orderId} not found`);
    }

    // Can only cancel pending or confirmed orders
    if (!['pending', 'confirmed'].includes(order.status)) {
      throw new ValidationError('Order cannot be cancelled at this stage');
    }

    await this.orderRepository.transaction(async (tx) => {
      // Update order status
      await tx.order.update({
        where: { id: orderId },
        data: { status: 'cancelled' },
      });

      // Release reserved inventory
      const items = await tx.orderItem.findMany({
        where: { orderId },
      });

      for (const item of items) {
        await this.inventoryService.releaseStock(item.variantId, item.quantity, tx);
      }

      // Refund payment if already paid
      if (order.paymentStatus === 'paid') {
        await this.worker.addJob('process-refund', {
          orderId,
          amount: order.totalAmount,
          reason,
        });
      }

      // Add status history
      await tx.orderStatusHistory.create({
        data: {
          orderId,
          status: 'cancelled',
          changedBy: userId,
          notes: reason,
        },
      });
    });

    // Send cancellation notification
    await this.worker.addJob('send-order-cancellation', {
      orderId,
      customerId: order.customerId,
    });
  }

  /**
   * Generate unique order number
   */
  private generateOrderNumber(): string {
    const timestamp = Date.now().toString(36).toUpperCase();
    const random = Math.random().toString(36).substring(2, 6).toUpperCase();
    return `ORD-${timestamp}-${random}`;
  }

  /**
   * Validate status transition
   */
  private validateStatusTransition(currentStatus: string, newStatus: string) {
    const validTransitions: Record<string, string[]> = {
      pending: ['confirmed', 'cancelled'],
      confirmed: ['processing', 'cancelled'],
      processing: ['packed', 'cancelled'],
      packed: ['shipped'],
      shipped: ['delivered', 'returned'],
      delivered: ['returned'],
      cancelled: [],
      returned: [],
    };

    if (!validTransitions[currentStatus]?.includes(newStatus)) {
      throw new ValidationError(
        `Invalid status transition from ${currentStatus} to ${newStatus}`
      );
    }
  }

  /**
   * Handle status change side effects
   */
  private async handleStatusChange(order: any, newStatus: string) {
    switch (newStatus) {
      case 'confirmed':
        // Queue for processing
        await this.worker.addJob('process-order', { orderId: order.id });
        break;

      case 'packed':
        // Create shipment
        await this.worker.addJob('create-shipment', { orderId: order.id });
        break;

      case 'shipped':
        // Send tracking notification
        await this.worker.addJob('send-shipping-notification', {
          orderId: order.id,
          customerId: order.customerId,
        });
        break;

      case 'delivered':
        // Update customer lifetime value
        // Request feedback
        await Promise.all([
          this.worker.addJob('update-customer-metrics', {
            customerId: order.customerId,
            orderValue: order.totalAmount,
          }),
          this.worker.addJob('request-feedback', {
            orderId: order.id,
            customerId: order.customerId,
          }),
        ]);
        break;
    }
  }
}
```

---

## 5. Graphile Worker Integration

### 5.1 Worker Setup

```typescript
// src/workers/worker.ts

import { run, quickAddJob, JobHelpers } from 'graphile-worker';
import { PrismaClient } from '@prisma/client';
import { logger } from '../shared/utils/logger.util';
import { config } from '../config';

// Import all task handlers
import * as tasks from './tasks';

export class GraphileWorker {
  private runner: any;
  private prisma: PrismaClient;

  constructor() {
    this.prisma = new PrismaClient();
  }

  /**
   * Initialize and start worker
   */
  async start() {
    try {
      this.runner = await run({
        connectionString: config.database.url,
        concurrency: 4, // Number of concurrent jobs
        pollInterval: 1000, // Check for jobs every 1 second
        taskList: {
          // Order processing
          'process-order': tasks.processOrder,
          'process-payment': tasks.processPayment,
          'process-refund': tasks.processRefund,
          'create-shipment': tasks.createShipment,

          // Notifications
          'send-order-confirmation': tasks.sendOrderConfirmation,
          'send-shipping-notification': tasks.sendShippingNotification,
          'send-order-cancellation': tasks.sendOrderCancellation,
          'send-email': tasks.sendEmail,
          'send-sms': tasks.sendSMS,
          'send-push-notification': tasks.sendPushNotification,

          // Marketplace
          'sync-marketplace-inventory': tasks.syncMarketplaceInventory,
          'sync-marketplace-price': tasks.syncMarketplacePrice,
          'import-marketplace-order': tasks.importMarketplaceOrder,

          // Product
          'process-product-images': tasks.processProductImages,
          'bulk-product-upload': tasks.bulkProductUpload,

          // Analytics
          'generate-report': tasks.generateReport,
          'update-customer-metrics': tasks.updateCustomerMetrics,

          // Inventory
          'check-low-stock': tasks.checkLowStock,
          'generate-reorder-alerts': tasks.generateReorderAlerts,

          // Customer
          'request-feedback': tasks.requestFeedback,
          'send-abandoned-cart-reminder': tasks.sendAbandonedCartReminder,
        },
      });

      logger.info('Graphile Worker started successfully');

      // Handle graceful shutdown
      process.on('SIGTERM', () => this.stop());
      process.on('SIGINT', () => this.stop());
    } catch (error) {
      logger.error('Failed to start Graphile Worker:', error);
      throw error;
    }
  }

  /**
   * Stop worker
   */
  async stop() {
    if (this.runner) {
      await this.runner.stop();
      logger.info('Graphile Worker stopped');
    }
    await this.prisma.$disconnect();
  }

  /**
   * Add job to queue
   */
  async addJob(taskIdentifier: string, payload: any, options: any = {}) {
    try {
      const job = await quickAddJob(
        {
          connectionString: config.database.url,
        },
        taskIdentifier,
        payload,
        {
          maxAttempts: options.maxAttempts || 3,
          runAt: options.runAt, // Schedule for later
          priority: options.priority || 0,
        }
      );

      logger.info(`Job queued: ${taskIdentifier}`, { jobId: job.id });
      return job;
    } catch (error) {
      logger.error(`Failed to queue job: ${taskIdentifier}`, error);
      throw error;
    }
  }
}

// Export singleton instance
export const worker = new GraphileWorker();
```

### 5.2 Task Handlers

```typescript
// src/workers/tasks/order-processing.task.ts

import { JobHelpers } from 'graphile-worker';
import { PrismaClient } from '@prisma/client';
import { logger } from '../../shared/utils/logger.util';
import { OrderService } from '../../modules/orders/services/order.service';

const prisma = new PrismaClient();

/**
 * Process order - Pick items, update inventory
 */
export async function processOrder(payload: any, helpers: JobHelpers) {
  const { orderId } = payload;

  try {
    logger.info(`Processing order ${orderId}`);

    // Get order details
    const order = await prisma.order.findUnique({
      where: { id: orderId },
      include: {
        items: {
          include: {
            variant: true,
          },
        },
      },
    });

    if (!order) {
      throw new Error(`Order ${orderId} not found`);
    }

    // Pick items from warehouse
    for (const item of order.items) {
      await prisma.stockMovement.create({
        data: {
          variantId: item.variantId,
          fromWarehouseId: 1, // Main warehouse
          movementType: 'OUT',
          quantity: item.quantity,
          referenceType: 'order',
          referenceId: orderId,
          notes: `Picked for order ${order.orderNumber}`,
        },
      });

      // Update inventory
      await prisma.inventory.update({
        where: {
          variantId_warehouseId_binLocationId: {
            variantId: item.variantId,
            warehouseId: 1,
            binLocationId: null,
          },
        },
        data: {
          quantity: { decrement: item.quantity },
          reservedQuantity: { decrement: item.quantity },
        },
      });
    }

    // Update order status
    await prisma.order.update({
      where: { id: orderId },
      data: { status: 'packed' },
    });

    // Add status history
    await prisma.orderStatusHistory.create({
      data: {
        orderId,
        status: 'packed',
        notes: 'Order packed and ready for shipment',
      },
    });

    logger.info(`Order ${orderId} processed successfully`);
  } catch (error) {
    logger.error(`Failed to process order ${orderId}:`, error);
    throw error;
  }
}

/**
 * Process payment
 */
export async function processPayment(payload: any, helpers: JobHelpers) {
  const { orderId, amount, method } = payload;

  try {
    logger.info(`Processing payment for order ${orderId}`);

    // Get order
    const order = await prisma.order.findUnique({
      where: { id: orderId },
      include: { customer: true },
    });

    if (!order) {
      throw new Error(`Order ${orderId} not found`);
    }

    // Initialize payment gateway (Stripe/Razorpay)
    // This is a simplified example
    const paymentGateway = method === 'stripe' 
      ? new StripeGateway() 
      : new RazorpayGateway();

    const result = await paymentGateway.charge({
      amount,
      currency: order.currency,
      description: `Payment for order ${order.orderNumber}`,
      customer: order.customer,
    });

    // Create payment record
    await prisma.payment.create({
      data: {
        orderId,
        customerId: order.customerId,
        paymentId: result.id,
        gateway: method,
        method: result.method,
        amount,
        currency: order.currency,
        status: result.status,
        transactionId: result.transactionId,
        gatewayResponse: result.rawResponse,
      },
    });

    // Update order payment status
    if (result.status === 'completed') {
      await prisma.order.update({
        where: { id: orderId },
        data: { paymentStatus: 'paid' },
      });

      // Queue order confirmation
      await helpers.addJob('send-order-confirmation', { orderId });
    }

    logger.info(`Payment processed for order ${orderId}`);
  } catch (error) {
    logger.error(`Payment failed for order ${orderId}:`, error);
    
    // Update order payment status
    await prisma.order.update({
      where: { id: orderId },
      data: { paymentStatus: 'failed' },
    });

    throw error;
  }
}
```

```typescript
// src/workers/tasks/email-notification.task.ts

import { JobHelpers } from 'graphile-worker';
import { logger } from '../../shared/utils/logger.util';
import { EmailService } from '../../modules/notifications/services/email.service';

const emailService = new EmailService();

/**
 * Send order confirmation email
 */
export async function sendOrderConfirmation(payload: any, helpers: JobHelpers) {
  const { orderId, customerId } = payload;

  try {
    logger.info(`Sending order confirmation for order ${orderId}`);

    // Get order and customer details
    const order = await prisma.order.findUnique({
      where: { id: orderId },
      include: {
        customer: { include: { user: true } },
        items: { include: { variant: { include: { product: true } } } },
        addresses: true,
      },
    });

    if (!order) {
      throw new Error(`Order ${orderId} not found`);
    }

    // Send email
    await emailService.send({
      to: order.customer.user.email,
      subject: `Order Confirmation - ${order.orderNumber}`,
      template: 'order-confirmation',
      variables: {
        customerName: `${order.customer.user.firstName} ${order.customer.user.lastName}`,
        orderNumber: order.orderNumber,
        orderDate: order.createdAt,
        items: order.items,
        subtotal: order.subtotal,
        tax: order.taxAmount,
        shipping: order.shippingCost,
        total: order.totalAmount,
        shippingAddress: order.addresses.find((a) => a.addressType === 'shipping'),
      },
    });

    logger.info(`Order confirmation sent for order ${orderId}`);
  } catch (error) {
    logger.error(`Failed to send order confirmation for order ${orderId}:`, error);
    throw error;
  }
}

/**
 * Send generic email
 */
export async function sendEmail(payload: any, helpers: JobHelpers) {
  const { to, subject, template, variables } = payload;

  try {
    await emailService.send({ to, subject, template, variables });
    
    // Log email
    await prisma.emailLog.create({
      data: {
        toEmail: to,
        subject,
        template,
        variables,
        status: 'sent',
        sentAt: new Date(),
      },
    });

    logger.info(`Email sent to ${to}`);
  } catch (error) {
    logger.error(`Failed to send email to ${to}:`, error);
    
    // Log failed email
    await prisma.emailLog.create({
      data: {
        toEmail: to,
        subject,
        template,
        variables,
        status: 'failed',
        error: error.message,
      },
    });

    throw error;
  }
}
```

---

## 6. Cron Jobs

### 6.1 Cron Job Setup

```typescript
// src/workers/crons/index.ts

import cron from 'node-cron';
import { logger } from '../../shared/utils/logger.util';
import { worker } from '../worker';

/**
 * Initialize all cron jobs
 */
export function initializeCronJobs() {
  logger.info('Initializing cron jobs...');

  // Daily metrics calculation - Run at 1:00 AM every day
  cron.schedule('0 1 * * *', async () => {
    logger.info('Running daily metrics calculation');
    await worker.addJob('calculate-daily-metrics', {
      date: new Date().toISOString().split('T')[0],
    });
  });

  // Low stock check - Run every 2 hours
  cron.schedule('0 */2 * * *', async () => {
    logger.info('Running low stock check');
    await worker.addJob('check-low-stock', {});
  });

  // Abandoned cart reminders - Run every 4 hours
  cron.schedule('0 */4 * * *', async () => {
    logger.info('Running abandoned cart reminder');
    await worker.addJob('send-abandoned-cart-reminder', {});
  });

  // Marketplace inventory sync - Run every 30 minutes
  cron.schedule('*/30 * * * *', async () => {
    logger.info('Running marketplace inventory sync');
    await worker.addJob('sync-marketplace-inventory', {});
  });

  // Refresh materialized views - Run at 2:00 AM every day
  cron.schedule('0 2 * * *', async () => {
    logger.info('Refreshing materialized views');
    await worker.addJob('refresh-materialized-views', {});
  });

  // Cleanup old sessions - Run at 3:00 AM every day
  cron.schedule('0 3 * * *', async () => {
    logger.info('Cleaning up old sessions');
    await worker.addJob('cleanup-old-sessions', {});
  });

  // Backup database - Run at 4:00 AM every day
  cron.schedule('0 4 * * *', async () => {
    logger.info('Starting database backup');
    await worker.addJob('backup-database', {});
  });

  // Generate weekly reports - Run at 6:00 AM every Monday
  cron.schedule('0 6 * * 1', async () => {
    logger.info('Generating weekly reports');
    await worker.addJob('generate-weekly-report', {});
  });

  logger.info('Cron jobs initialized successfully');
}
```

### 6.2 Cron Task Handlers

```typescript
// src/workers/tasks/daily-metrics.task.ts

import { JobHelpers } from 'graphile-worker';
import { PrismaClient } from '@prisma/client';
import { logger } from '../../shared/utils/logger.util';

const prisma = new PrismaClient();

/**
 * Calculate daily metrics
 */
export async function calculateDailyMetrics(payload: any, helpers: JobHelpers) {
  const { date } = payload;
  const targetDate = new Date(date);
  const nextDate = new Date(targetDate);
  nextDate.setDate(nextDate.getDate() + 1);

  try {
    logger.info(`Calculating daily metrics for ${date}`);

    // Get orders for the day
    const orders = await prisma.order.findMany({
      where: {
        createdAt: {
          gte: targetDate,
          lt: nextDate,
        },
        status: { not: 'cancelled' },
      },
    });

    // Calculate metrics
    const totalOrders = orders.length;
    const totalRevenue = orders.reduce((sum, order) => sum + Number(order.totalAmount), 0);
    const avgOrderValue = totalOrders > 0 ? totalRevenue / totalOrders : 0;

    // Get new customers
    const newCustomers = await prisma.customer.count({
      where: {
        createdAt: {
          gte: targetDate,
          lt: nextDate,
        },
      },
    });

    // Get returning customers
    const returningCustomers = await prisma.$queryRaw`
      SELECT COUNT(DISTINCT customer_id) as count
      FROM orders
      WHERE DATE(created_at) = ${date}
      AND customer_id IN (
        SELECT customer_id
        FROM orders
        WHERE DATE(created_at) < ${date}
        GROUP BY customer_id
        HAVING COUNT(*) > 1
      )
    `;

    // Get total units sold
    const unitsResult = await prisma.orderItem.aggregate({
      where: {
        order: {
          createdAt: {
            gte: targetDate,
            lt: nextDate,
          },
          status: { not: 'cancelled' },
        },
      },
      _sum: {
        quantity: true,
      },
    });

    const totalUnitsSold = unitsResult._sum.quantity || 0;

    // Calculate conversion rate (simplified - would need session data)
    const sessions = 1000; // This should come from analytics
    const conversionRate = (totalOrders / sessions) * 100;

    // Insert or update daily metrics
    await prisma.dailyMetrics.upsert({
      where: { date: targetDate },
      update: {
        totalOrders,
        totalRevenue,
        avgOrderValue,
        newCustomers,
        returningCustomers: returningCustomers[0].count,
        totalUnitsSold,
        conversionRate,
      },
      create: {
        date: targetDate,
        totalOrders,
        totalRevenue,
        avgOrderValue,
        newCustomers,
        returningCustomers: returningCustomers[0].count,
        totalUnitsSold,
        conversionRate,
      },
    });

    logger.info(`Daily metrics calculated for ${date}`);
  } catch (error) {
    logger.error(`Failed to calculate daily metrics for ${date}:`, error);
    throw error;
  }
}

/**
 * Check low stock and generate alerts
 */
export async function checkLowStock(payload: any, helpers: JobHelpers) {
  try {
    logger.info('Checking for low stock items');

    // Find items below reorder point
    const lowStockItems = await prisma.inventory.findMany({
      where: {
        quantity: {
          lte: prisma.inventory.fields.reorderPoint,
        },
      },
      include: {
        variant: {
          include: {
            product: true,
          },
        },
        warehouse: true,
      },
    });

    if (lowStockItems.length === 0) {
      logger.info('No low stock items found');
      return;
    }

    logger.info(`Found ${lowStockItems.length} low stock items`);

    // Send notifications to inventory managers
    const managers = await prisma.user.findMany({
      where: {
        userRoles: {
          some: {
            role: {
              slug: 'inventory-manager',
            },
          },
        },
      },
    });

    for (const manager of managers) {
      await helpers.addJob('send-email', {
        to: manager.email,
        subject: 'Low Stock Alert',
        template: 'low-stock-alert',
        variables: {
          items: lowStockItems,
          date: new Date().toISOString(),
        },
      });
    }

    // Create notifications in database
    for (const item of lowStockItems) {
      await prisma.notification.create({
        data: {
          type: 'inventory',
          channel: 'in_app',
          title: 'Low Stock Alert',
          message: `${item.variant.product.name} (${item.variant.sku}) is running low. Current stock: ${item.quantity}, Reorder point: ${item.reorderPoint}`,
          data: JSON.stringify({ variantId: item.variantId, warehouseId: item.warehouseId }),
          status: 'pending',
        },
      });
    }

    logger.info('Low stock alerts sent successfully');
  } catch (error) {
    logger.error('Failed to check low stock:', error);
    throw error;
  }
}
```

---

## 7. Error Handling

### 7.1 Custom Error Classes

```typescript
// src/shared/errors/app.error.ts

export class AppError extends Error {
  statusCode: number;
  isOperational: boolean;

  constructor(message: string, statusCode: number = 500) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;

    Error.captureStackTrace(this, this.constructor);
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message: string = 'Unauthorized') {
    super(message, 401);
  }
}

export class ForbiddenError extends AppError {
  constructor(message: string = 'Forbidden') {
    super(message, 403);
  }
}

export class NotFoundError extends AppError {
  constructor(message: string) {
    super(message, 404);
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409);
  }
}

export class InsufficientInventoryError extends AppError {
  constructor(message: string) {
    super(message, 422);
  }
}

export class ExternalServiceError extends AppError {
  constructor(message: string, service: string) {
    super(`${service}: ${message}`, 502);
  }
}
```

### 7.2 Error Middleware

```typescript
// src/shared/middleware/error.middleware.ts

import { Request, Response, NextFunction } from 'express';
import { AppError } from '../errors/app.error';
import { logger } from '../utils/logger.util';
import { PrismaClientKnownRequestError } from '@prisma/client/runtime';

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  // Log error
  logger.error('Error occurred:', {
    message: error.message,
    stack: error.stack,
    url: req.url,
    method: req.method,
    body: req.body,
    user: req.user?.id,
  });

  // Prisma errors
  if (error instanceof PrismaClientKnownRequestError) {
    return handlePrismaError(error, res);
  }

  // Application errors
  if (error instanceof AppError) {
    return res.status(error.statusCode).json({
      success: false,
      message: error.message,
      ...(process.env.NODE_ENV === 'development' && { stack: error.stack }),
    });
  }

  // Validation errors (from express-validator)
  if (error.name === 'ValidationError') {
    return res.status(400).json({
      success: false,
      message: 'Validation failed',
      errors: error.message,
    });
  }

  // JWT errors
  if (error.name === 'JsonWebTokenError') {
    return res.status(401).json({
      success: false,
      message: 'Invalid token',
    });
  }

  if (error.name === 'TokenExpiredError') {
    return res.status(401).json({
      success: false,
      message: 'Token expired',
    });
  }

  // Default error
  return res.status(500).json({
    success: false,
    message: 'Internal server error',
    ...(process.env.NODE_ENV === 'development' && {
      error: error.message,
      stack: error.stack,
    }),
  });
}

function handlePrismaError(error: PrismaClientKnownRequestError, res: Response) {
  switch (error.code) {
    case 'P2002':
      return res.status(409).json({
        success: false,
        message: 'A record with this value already exists',
        field: error.meta?.target,
      });

    case 'P2025':
      return res.status(404).json({
        success: false,
        message: 'Record not found',
      });

    case 'P2003':
      return res.status(400).json({
        success: false,
        message: 'Invalid reference',
      });

    default:
      return res.status(500).json({
        success: false,
        message: 'Database error',
      });
  }
}
```

---

## 8. Security Implementation

### 8.1 Authentication Middleware

```typescript
// src/shared/middleware/auth.middleware.ts

import { Request, Response, NextFunction } from 'express';
import { JwtService } from '../../modules/auth/services/jwt.service';
import { UnauthorizedError, ForbiddenError } from '../errors';
import { UserRepository } from '../../modules/auth/repositories/user.repository';

const jwtService = new JwtService();
const userRepository = new UserRepository();

/**
 * Authenticate user from JWT token
 */
export async function authenticate(req: Request, res: Response, next: NextFunction) {
  try {
    const token = extractToken(req);

    if (!token) {
      throw new UnauthorizedError('No token provided');
    }

    // Verify token
    const payload = await jwtService.verifyAccessToken(token);

    // Get user
    const user = await userRepository.findById(payload.userId);

    if (!user || user.status !== 'active') {
      throw new UnauthorizedError('Invalid token');
    }

    // Attach user to request
    req.user = {
      id: user.id,
      email: user.email,
      roles: payload.roles,
      permissions: payload.permissions,
    };

    next();
  } catch (error) {
    next(error);
  }
}

/**
 * Check if user has required role
 */
export function authorize(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return next(new UnauthorizedError('Not authenticated'));
    }

    const hasRole = roles.some((role) => req.user!.roles.includes(role));

    if (!hasRole) {
      return next(new ForbiddenError('Insufficient permissions'));
    }

    next();
  };
}

/**
 * Check if user has required permission
 */
export function checkPermission(resource: string, action: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return next(new UnauthorizedError('Not authenticated'));
    }

    const permission = `${resource}:${action}`;
    const hasPermission = req.user.permissions.includes(permission);

    if (!hasPermission) {
      return next(new ForbiddenError(`Permission denied: ${permission}`));
    }

    next();
  };
}

/**
 * Extract token from request
 */
function extractToken(req: Request): string | null {
  // Check Authorization header
  if (req.headers.authorization?.startsWith('Bearer ')) {
    return req.headers.authorization.substring(7);
  }

  // Check cookie
  if (req.cookies?.accessToken) {
    return req.cookies.accessToken;
  }

  return null;
}
```

### 8.2 Rate Limiting

```typescript
// src/shared/middleware/ratelimit.middleware.ts

import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../../config/redis';

/**
 * General API rate limiter
 */
export const apiLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:api:',
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Max 100 requests per windowMs
  message: 'Too many requests from this IP, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
});

/**
 * Auth endpoints rate limiter (stricter)
 */
export const authLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:auth:',
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // Max 5 requests per windowMs
  message: 'Too many login attempts, please try again later',
  skipSuccessfulRequests: true, // Don't count successful logins
});

/**
 * Create order rate limiter
 */
export const orderLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:order:',
  }),
  windowMs: 60 * 1000, // 1 minute
  max: 10, // Max 10 orders per minute
  message: 'Too many orders, please slow down',
});
```

---

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Next Review:** April 26, 2026  
**Status:** ✅ Production-Ready

---

[← Back to Index](./00_MASTER_INDEX.md) | [Next: Frontend Architecture →](./04_FRONTEND_ARCHITECTURE.md)
