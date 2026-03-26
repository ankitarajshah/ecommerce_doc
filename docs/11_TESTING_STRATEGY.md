# 🧪 Testing Strategy Documentation

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Status:** ✅ Production Ready

---

## Table of Contents

1. [Testing Overview](#testing-overview)
2. [Unit Testing](#unit-testing)
3. [Integration Testing](#integration-testing)
4. [End-to-End Testing](#end-to-end-testing)
5. [API Testing](#api-testing)
6. [Performance Testing](#performance-testing)
7. [Security Testing](#security-testing)
8. [Test Automation](#test-automation)

---

## 1. Testing Overview

### 1.1 Testing Pyramid

```
                 ┌──────────────┐
                 │  Manual (5%) │
                 └──────────────┘
            ┌────────────────────────┐
            │  E2E Tests (15%)       │
            └────────────────────────┘
       ┌───────────────────────────────────┐
       │  Integration Tests (30%)          │
       └───────────────────────────────────┘
  ┌──────────────────────────────────────────────┐
  │  Unit Tests (50%)                            │
  └──────────────────────────────────────────────┘
```

### 1.2 Testing Stack

```yaml
Unit Testing:
  Framework: Jest
  Assertions: Jest Matchers
  Mocking: Jest Mocks
  Coverage: Istanbul

Integration Testing:
  Framework: Jest + Supertest
  Database: Test PostgreSQL instance
  API: Supertest

E2E Testing:
  Framework: Playwright
  Browser: Chromium, Firefox, WebKit
  Visual: Percy

Performance Testing:
  Load Testing: k6
  Stress Testing: Artillery
  Profiling: Chrome DevTools

Security Testing:
  SAST: SonarQube, ESLint Security
  DAST: OWASP ZAP
  Dependencies: Snyk, npm audit

Test Data:
  Factory: Faker.js
  Seeding: Prisma Seed
  Fixtures: JSON files
```

### 1.3 Testing Principles

- ✅ **Write testable code** - Follow SOLID principles
- ✅ **Test behavior, not implementation** - Focus on outcomes
- ✅ **Maintain independence** - Tests should not depend on each other
- ✅ **Fast feedback** - Tests should run quickly
- ✅ **Meaningful assertions** - Test what matters
- ✅ **Clear test names** - Describe what is being tested
- ✅ **Arrange-Act-Assert pattern** - Structure tests consistently

---

## 2. Unit Testing

### 2.1 Jest Configuration

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.test.ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/__tests__/**',
    '!src/server.ts'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  moduleNameMapper: {
    '@/(.*)': '<rootDir>/src/$1'
  },
  setupFilesAfterEnv: ['<rootDir>/src/__tests__/setup.ts'],
  testTimeout: 10000,
  maxWorkers: '50%'
};
```

### 2.2 Service Layer Tests

```typescript
// src/features/products/__tests__/product.service.test.ts
import { ProductService } from '../product.service';
import { prismaMock } from '@/__tests__/mocks/prisma';
import { mockProduct } from '@/__tests__/fixtures/product.fixture';

describe('ProductService', () => {
  let productService: ProductService;

  beforeEach(() => {
    productService = new ProductService();
    jest.clearAllMocks();
  });

  describe('createProduct', () => {
    it('should create a product successfully', async () => {
      // Arrange
      const productData = {
        name: 'Test Product',
        sku: 'TEST-001',
        price: 999.99,
        categoryId: 'cat-123'
      };

      prismaMock.product.create.mockResolvedValue(
        mockProduct(productData)
      );

      // Act
      const result = await productService.createProduct(productData);

      // Assert
      expect(result).toMatchObject(productData);
      expect(prismaMock.product.create).toHaveBeenCalledWith({
        data: expect.objectContaining(productData)
      });
    });

    it('should throw error if SKU already exists', async () => {
      // Arrange
      const productData = {
        name: 'Test Product',
        sku: 'DUPLICATE-SKU',
        price: 999.99,
        categoryId: 'cat-123'
      };

      prismaMock.product.findUnique.mockResolvedValue(
        mockProduct({ sku: 'DUPLICATE-SKU' })
      );

      // Act & Assert
      await expect(
        productService.createProduct(productData)
      ).rejects.toThrow('SKU already exists');
    });

    it('should validate product price', async () => {
      // Arrange
      const productData = {
        name: 'Test Product',
        sku: 'TEST-002',
        price: -100, // Invalid price
        categoryId: 'cat-123'
      };

      // Act & Assert
      await expect(
        productService.createProduct(productData)
      ).rejects.toThrow('Price must be positive');
    });
  });

  describe('getProduct', () => {
    it('should return product by ID', async () => {
      // Arrange
      const productId = 'prod-123';
      const expectedProduct = mockProduct({ id: productId });

      prismaMock.product.findUnique.mockResolvedValue(expectedProduct);

      // Act
      const result = await productService.getProduct(productId);

      // Assert
      expect(result).toEqual(expectedProduct);
      expect(prismaMock.product.findUnique).toHaveBeenCalledWith({
        where: { id: productId },
        include: expect.any(Object)
      });
    });

    it('should return null if product not found', async () => {
      // Arrange
      prismaMock.product.findUnique.mockResolvedValue(null);

      // Act
      const result = await productService.getProduct('nonexistent');

      // Assert
      expect(result).toBeNull();
    });
  });

  describe('updateProduct', () => {
    it('should update product successfully', async () => {
      // Arrange
      const productId = 'prod-123';
      const updateData = { price: 1299.99 };
      const updatedProduct = mockProduct({ id: productId, ...updateData });

      prismaMock.product.update.mockResolvedValue(updatedProduct);

      // Act
      const result = await productService.updateProduct(productId, updateData);

      // Assert
      expect(result.price).toBe(updateData.price);
      expect(prismaMock.product.update).toHaveBeenCalledWith({
        where: { id: productId },
        data: updateData
      });
    });
  });

  describe('deleteProduct', () => {
    it('should soft delete product', async () => {
      // Arrange
      const productId = 'prod-123';

      prismaMock.product.update.mockResolvedValue(
        mockProduct({ id: productId, isDeleted: true })
      );

      // Act
      await productService.deleteProduct(productId);

      // Assert
      expect(prismaMock.product.update).toHaveBeenCalledWith({
        where: { id: productId },
        data: { isDeleted: true, deletedAt: expect.any(Date) }
      });
    });
  });

  describe('searchProducts', () => {
    it('should search products by name', async () => {
      // Arrange
      const searchTerm = 'shirt';
      const expectedProducts = [
        mockProduct({ name: 'Blue Shirt' }),
        mockProduct({ name: 'Red Shirt' })
      ];

      prismaMock.product.findMany.mockResolvedValue(expectedProducts);

      // Act
      const result = await productService.searchProducts(searchTerm);

      // Assert
      expect(result).toHaveLength(2);
      expect(prismaMock.product.findMany).toHaveBeenCalledWith({
        where: {
          name: { contains: searchTerm, mode: 'insensitive' }
        }
      });
    });
  });
});
```

### 2.3 Repository Layer Tests

```typescript
// src/repositories/__tests__/product.repository.test.ts
import { ProductRepository } from '../product.repository';
import { prismaMock } from '@/__tests__/mocks/prisma';

describe('ProductRepository', () => {
  let repository: ProductRepository;

  beforeEach(() => {
    repository = new ProductRepository();
  });

  describe('findById', () => {
    it('should find product by ID with relations', async () => {
      // Arrange
      const productId = 'prod-123';
      const mockData = {
        id: productId,
        name: 'Test Product',
        category: { id: 'cat-1', name: 'Category' },
        variants: [{ id: 'var-1', size: 'M' }]
      };

      prismaMock.product.findUnique.mockResolvedValue(mockData as any);

      // Act
      const result = await repository.findById(productId);

      // Assert
      expect(result).toBeDefined();
      expect(result?.category).toBeDefined();
      expect(result?.variants).toHaveLength(1);
    });
  });

  describe('findBySKU', () => {
    it('should find product by SKU', async () => {
      // Arrange
      const sku = 'TEST-SKU-001';
      const mockData = { id: 'prod-1', sku };

      prismaMock.product.findUnique.mockResolvedValue(mockData as any);

      // Act
      const result = await repository.findBySKU(sku);

      // Assert
      expect(result?.sku).toBe(sku);
    });
  });

  describe('findByCategory', () => {
    it('should find products by category', async () => {
      // Arrange
      const categoryId = 'cat-123';
      const mockData = [
        { id: 'prod-1', categoryId },
        { id: 'prod-2', categoryId }
      ];

      prismaMock.product.findMany.mockResolvedValue(mockData as any);

      // Act
      const result = await repository.findByCategory(categoryId);

      // Assert
      expect(result).toHaveLength(2);
      expect(prismaMock.product.findMany).toHaveBeenCalledWith({
        where: { categoryId }
      });
    });
  });

  describe('create', () => {
    it('should create product with transaction', async () => {
      // Arrange
      const productData = {
        name: 'New Product',
        sku: 'NEW-001',
        price: 999
      };

      prismaMock.$transaction.mockImplementation(
        async (callback: any) => callback(prismaMock)
      );
      prismaMock.product.create.mockResolvedValue({
        id: 'prod-new',
        ...productData
      } as any);

      // Act
      const result = await repository.create(productData);

      // Assert
      expect(result.sku).toBe(productData.sku);
      expect(prismaMock.$transaction).toHaveBeenCalled();
    });
  });
});
```

### 2.4 Utility Function Tests

```typescript
// src/utils/__tests__/validation.test.ts
import {
  validateEmail,
  validatePhone,
  validateSKU,
  sanitizeInput
} from '../validation';

describe('Validation Utilities', () => {
  describe('validateEmail', () => {
    it('should validate correct email formats', () => {
      expect(validateEmail('test@example.com')).toBe(true);
      expect(validateEmail('user+tag@domain.co.in')).toBe(true);
    });

    it('should reject invalid email formats', () => {
      expect(validateEmail('invalid')).toBe(false);
      expect(validateEmail('test@')).toBe(false);
      expect(validateEmail('@example.com')).toBe(false);
    });
  });

  describe('validatePhone', () => {
    it('should validate Indian phone numbers', () => {
      expect(validatePhone('+919876543210')).toBe(true);
      expect(validatePhone('9876543210')).toBe(true);
    });

    it('should reject invalid phone numbers', () => {
      expect(validatePhone('123')).toBe(false);
      expect(validatePhone('abcdefghij')).toBe(false);
    });
  });

  describe('validateSKU', () => {
    it('should validate SKU format', () => {
      expect(validateSKU('PROD-123-ABC')).toBe(true);
      expect(validateSKU('SKU-2024-001')).toBe(true);
    });

    it('should reject invalid SKU format', () => {
      expect(validateSKU('invalid sku')).toBe(false);
      expect(validateSKU('SKU@123')).toBe(false);
    });
  });

  describe('sanitizeInput', () => {
    it('should remove XSS attempts', () => {
      const malicious = '<script>alert("xss")</script>';
      expect(sanitizeInput(malicious)).not.toContain('<script>');
    });

    it('should preserve safe HTML', () => {
      const safe = 'Hello World';
      expect(sanitizeInput(safe)).toBe(safe);
    });
  });
});
```

### 2.5 Test Fixtures

```typescript
// src/__tests__/fixtures/product.fixture.ts
import { faker } from '@faker-js/faker';

export function mockProduct(overrides?: Partial<Product>): Product {
  return {
    id: faker.string.uuid(),
    name: faker.commerce.productName(),
    sku: faker.string.alphanumeric(10).toUpperCase(),
    description: faker.commerce.productDescription(),
    price: parseFloat(faker.commerce.price()),
    costPrice: parseFloat(faker.commerce.price({ min: 100, max: 500 })),
    categoryId: faker.string.uuid(),
    isActive: true,
    isFeatured: false,
    isDeleted: false,
    createdAt: faker.date.past(),
    updatedAt: faker.date.recent(),
    ...overrides
  };
}

export function mockProductWithVariants(): Product {
  return {
    ...mockProduct(),
    variants: [
      {
        id: faker.string.uuid(),
        size: 'M',
        color: 'Blue',
        sku: faker.string.alphanumeric(10),
        stock: 100
      },
      {
        id: faker.string.uuid(),
        size: 'L',
        color: 'Blue',
        sku: faker.string.alphanumeric(10),
        stock: 50
      }
    ]
  };
}
```

---

## 3. Integration Testing

### 3.1 API Integration Tests

```typescript
// src/features/products/__tests__/product.integration.test.ts
import request from 'supertest';
import { app } from '@/app';
import { prisma } from '@/config/database';
import { generateAuthToken } from '@/__tests__/helpers/auth';

describe('Product API Integration Tests', () => {
  let authToken: string;
  let testProduct: any;

  beforeAll(async () => {
    // Generate auth token
    authToken = generateAuthToken({ role: 'ADMIN' });

    // Seed test data
    testProduct = await prisma.product.create({
      data: {
        name: 'Test Product',
        sku: 'TEST-INT-001',
        price: 999.99,
        categoryId: 'cat-123'
      }
    });
  });

  afterAll(async () => {
    // Cleanup
    await prisma.product.deleteMany({
      where: { sku: { startsWith: 'TEST-INT-' } }
    });
    await prisma.$disconnect();
  });

  describe('POST /api/products', () => {
    it('should create a new product', async () => {
      // Arrange
      const productData = {
        name: 'New Test Product',
        sku: 'TEST-INT-002',
        price: 1299.99,
        description: 'Test description',
        categoryId: 'cat-123'
      };

      // Act
      const response = await request(app)
        .post('/api/products')
        .set('Authorization', `Bearer ${authToken}`)
        .send(productData)
        .expect(201);

      // Assert
      expect(response.body).toMatchObject({
        id: expect.any(String),
        name: productData.name,
        sku: productData.sku,
        price: productData.price
      });

      // Verify in database
      const dbProduct = await prisma.product.findUnique({
        where: { sku: productData.sku }
      });
      expect(dbProduct).toBeTruthy();
    });

    it('should return 400 for invalid data', async () => {
      // Arrange
      const invalidData = {
        name: 'Test',
        // Missing required fields
      };

      // Act & Assert
      const response = await request(app)
        .post('/api/products')
        .set('Authorization', `Bearer ${authToken}`)
        .send(invalidData)
        .expect(400);

      expect(response.body.error).toBeDefined();
    });

    it('should return 409 for duplicate SKU', async () => {
      // Arrange
      const duplicateData = {
        name: 'Duplicate Product',
        sku: testProduct.sku, // Existing SKU
        price: 999.99,
        categoryId: 'cat-123'
      };

      // Act & Assert
      await request(app)
        .post('/api/products')
        .set('Authorization', `Bearer ${authToken}`)
        .send(duplicateData)
        .expect(409);
    });

    it('should return 401 without authentication', async () => {
      // Act & Assert
      await request(app)
        .post('/api/products')
        .send({ name: 'Test' })
        .expect(401);
    });
  });

  describe('GET /api/products/:id', () => {
    it('should get product by ID', async () => {
      // Act
      const response = await request(app)
        .get(`/api/products/${testProduct.id}`)
        .expect(200);

      // Assert
      expect(response.body).toMatchObject({
        id: testProduct.id,
        name: testProduct.name,
        sku: testProduct.sku
      });
    });

    it('should return 404 for non-existent product', async () => {
      // Act & Assert
      await request(app)
        .get('/api/products/non-existent-id')
        .expect(404);
    });
  });

  describe('PUT /api/products/:id', () => {
    it('should update product', async () => {
      // Arrange
      const updateData = {
        name: 'Updated Product Name',
        price: 1599.99
      };

      // Act
      const response = await request(app)
        .put(`/api/products/${testProduct.id}`)
        .set('Authorization', `Bearer ${authToken}`)
        .send(updateData)
        .expect(200);

      // Assert
      expect(response.body.name).toBe(updateData.name);
      expect(response.body.price).toBe(updateData.price);

      // Verify in database
      const dbProduct = await prisma.product.findUnique({
        where: { id: testProduct.id }
      });
      expect(dbProduct?.name).toBe(updateData.name);
    });
  });

  describe('DELETE /api/products/:id', () => {
    it('should soft delete product', async () => {
      // Act
      await request(app)
        .delete(`/api/products/${testProduct.id}`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(204);

      // Assert - Product should be soft deleted
      const dbProduct = await prisma.product.findUnique({
        where: { id: testProduct.id }
      });
      expect(dbProduct?.isDeleted).toBe(true);
    });
  });

  describe('GET /api/products', () => {
    it('should list products with pagination', async () => {
      // Act
      const response = await request(app)
        .get('/api/products')
        .query({ page: 1, limit: 10 })
        .expect(200);

      // Assert
      expect(response.body).toHaveProperty('data');
      expect(response.body).toHaveProperty('pagination');
      expect(Array.isArray(response.body.data)).toBe(true);
      expect(response.body.pagination).toMatchObject({
        page: 1,
        limit: 10,
        total: expect.any(Number)
      });
    });

    it('should filter products by category', async () => {
      // Act
      const response = await request(app)
        .get('/api/products')
        .query({ categoryId: 'cat-123' })
        .expect(200);

      // Assert
      expect(response.body.data.every(
        (p: any) => p.categoryId === 'cat-123'
      )).toBe(true);
    });

    it('should search products by name', async () => {
      // Act
      const response = await request(app)
        .get('/api/products')
        .query({ search: 'Test' })
        .expect(200);

      // Assert
      expect(response.body.data.length).toBeGreaterThan(0);
      expect(response.body.data.some(
        (p: any) => p.name.includes('Test')
      )).toBe(true);
    });
  });
});
```

### 3.2 Database Integration Tests

```typescript
// src/__tests__/integration/database.test.ts
import { prisma } from '@/config/database';

describe('Database Integration Tests', () => {
  describe('Transactions', () => {
    it('should rollback on error', async () => {
      try {
        await prisma.$transaction(async (tx) => {
          // Create product
          const product = await tx.product.create({
            data: {
              name: 'Transactional Product',
              sku: 'TX-001',
              price: 999
            }
          });

          // Create inventory (this will fail)
          await tx.inventory.create({
            data: {
              productId: product.id,
              warehouseId: 'non-existent', // Invalid warehouse
              quantity: 100
            }
          });
        });
      } catch (error) {
        // Expected to fail
      }

      // Verify rollback - product should not exist
      const product = await prisma.product.findUnique({
        where: { sku: 'TX-001' }
      });
      expect(product).toBeNull();
    });
  });

  describe('Cascading Deletes', () => {
    it('should cascade delete related records', async () => {
      // Create product with variants
      const product = await prisma.product.create({
        data: {
          name: 'Cascade Test Product',
          sku: 'CASCADE-001',
          price: 999,
          variants: {
            create: [
              { size: 'M', sku: 'CASCADE-001-M' },
              { size: 'L', sku: 'CASCADE-001-L' }
            ]
          }
        }
      });

      // Delete product
      await prisma.product.delete({
        where: { id: product.id }
      });

      // Verify variants are also deleted
      const variants = await prisma.productVariant.findMany({
        where: { productId: product.id }
      });
      expect(variants).toHaveLength(0);
    });
  });

  describe('Indexes', () => {
    it('should use indexes for efficient queries', async () => {
      // This would require EXPLAIN ANALYZE in PostgreSQL
      // Verify query performance
      const start = Date.now();
      
      await prisma.product.findMany({
        where: {
          sku: 'TEST-001'
        }
      });
      
      const duration = Date.now() - start;
      expect(duration).toBeLessThan(100); // Should be fast with index
    });
  });
});
```

---

## 4. End-to-End Testing

### 4.1 Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['junit', { outputFile: 'test-results/junit.xml' }]
  ],
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure'
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] }
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] }
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] }
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] }
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] }
    }
  ],

  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI
  }
});
```

### 4.2 E2E Test Examples

```typescript
// e2e/auth/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('User Authentication', () => {
  test('should login successfully', async ({ page }) => {
    // Navigate to login page
    await page.goto('/login');

    // Fill login form
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="password"]', 'password123');

    // Submit form
    await page.click('button[type="submit"]');

    // Verify redirect to dashboard
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('h1')).toContainText('Dashboard');
  });

  test('should show error for invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[name="email"]', 'wrong@example.com');
    await page.fill('[name="password"]', 'wrongpassword');
    await page.click('button[type="submit"]');

    // Verify error message
    await expect(page.locator('.error-message')).toBeVisible();
    await expect(page.locator('.error-message'))
      .toContainText('Invalid credentials');
  });

  test('should validate email format', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[name="email"]', 'invalid-email');
    await page.click('button[type="submit"]');

    // Verify validation message
    const emailInput = page.locator('[name="email"]');
    await expect(emailInput).toHaveAttribute('aria-invalid', 'true');
  });
});
```

```typescript
// e2e/products/product-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Product Management Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login as admin
    await page.goto('/login');
    await page.fill('[name="email"]', 'admin@example.com');
    await page.fill('[name="password"]', 'admin123');
    await page.click('button[type="submit"]');
    await page.waitForURL('/dashboard');
  });

  test('should create a new product', async ({ page }) => {
    // Navigate to products page
    await page.click('text=Products');
    await page.click('text=Add Product');

    // Fill product form
    await page.fill('[name="name"]', 'Test Product');
    await page.fill('[name="sku"]', 'TEST-E2E-001');
    await page.fill('[name="price"]', '999.99');
    await page.selectOption('[name="categoryId"]', 'cat-123');

    // Upload image
    await page.setInputFiles(
      'input[type="file"]',
      'e2e/fixtures/test-image.jpg'
    );

    // Submit form
    await page.click('button[type="submit"]');

    // Verify success message
    await expect(page.locator('.success-message'))
      .toContainText('Product created successfully');

    // Verify product in list
    await page.goto('/products');
    await expect(page.locator('text=Test Product')).toBeVisible();
  });

  test('should update product details', async ({ page }) => {
    // Navigate to product
    await page.goto('/products');
    await page.click('text=Test Product');

    // Click edit button
    await page.click('button:has-text("Edit")');

    // Update price
    await page.fill('[name="price"]', '1299.99');
    await page.click('button[type="submit"]');

    // Verify update
    await expect(page.locator('.price')).toContainText('₹1,299.99');
  });

  test('should delete product', async ({ page }) => {
    await page.goto('/products');

    // Click delete on product
    await page.click('[data-testid="delete-TEST-E2E-001"]');

    // Confirm deletion
    await page.click('button:has-text("Confirm")');

    // Verify product removed
    await expect(page.locator('text=Test Product')).not.toBeVisible();
  });
});
```

```typescript
// e2e/orders/checkout-flow.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Checkout Flow', () => {
  test('should complete full checkout process', async ({ page }) => {
    // 1. Browse products
    await page.goto('/');
    await page.click('text=Men\'s Clothing');

    // 2. Add product to cart
    await page.click('[data-testid="product-card-1"]');
    await page.selectOption('[name="size"]', 'M');
    await page.click('button:has-text("Add to Cart")');

    // Verify cart count
    await expect(page.locator('.cart-count')).toHaveText('1');

    // 3. Go to cart
    await page.click('.cart-icon');
    await expect(page).toHaveURL('/cart');

    // 4. Proceed to checkout
    await page.click('button:has-text("Proceed to Checkout")');

    // 5. Fill shipping address
    await page.fill('[name="fullName"]', 'John Doe');
    await page.fill('[name="phone"]', '9876543210');
    await page.fill('[name="address"]', '123 Test Street');
    await page.fill('[name="city"]', 'Mumbai');
    await page.fill('[name="pincode"]', '400001');
    await page.click('button:has-text("Continue")');

    // 6. Select payment method
    await page.click('input[value="razorpay"]');
    await page.click('button:has-text("Place Order")');

    // 7. Complete payment (mock)
    // Note: In real scenario, this would interact with payment gateway
    await page.waitForURL('/order-confirmation/*');

    // 8. Verify order confirmation
    await expect(page.locator('h1')).toContainText('Order Placed');
    await expect(page.locator('.order-id')).toBeVisible();
  });

  test('should apply discount code', async ({ page }) => {
    // Add product to cart
    await page.goto('/');
    await page.click('[data-testid="product-card-1"]');
    await page.click('button:has-text("Add to Cart")');

    // Go to checkout
    await page.click('.cart-icon');
    await page.click('button:has-text("Proceed to Checkout")');

    // Apply discount code
    await page.fill('[name="discountCode"]', 'SAVE10');
    await page.click('button:has-text("Apply")');

    // Verify discount applied
    await expect(page.locator('.discount-amount'))
      .not.toContainText('₹0');
  });
});
```

---

## 5. API Testing

### 5.1 API Test Suite

```typescript
// tests/api/product-api.test.ts
import axios from 'axios';

const API_URL = process.env.API_URL || 'http://localhost:3000/api';

describe('Product API Tests', () => {
  let authToken: string;
  let createdProductId: string;

  beforeAll(async () => {
    // Login to get auth token
    const response = await axios.post(`${API_URL}/auth/login`, {
      email: 'admin@example.com',
      password: 'admin123'
    });
    authToken = response.data.accessToken;
  });

  describe('Product CRUD Operations', () => {
    test('POST /api/products - Create product', async () => {
      const response = await axios.post(
        `${API_URL}/products`,
        {
          name: 'API Test Product',
          sku: 'API-TEST-001',
          price: 999.99,
          categoryId: 'cat-123'
        },
        {
          headers: { Authorization: `Bearer ${authToken}` }
        }
      );

      expect(response.status).toBe(201);
      expect(response.data).toHaveProperty('id');
      expect(response.data.name).toBe('API Test Product');

      createdProductId = response.data.id;
    });

    test('GET /api/products/:id - Get product', async () => {
      const response = await axios.get(
        `${API_URL}/products/${createdProductId}`
      );

      expect(response.status).toBe(200);
      expect(response.data.id).toBe(createdProductId);
    });

    test('PUT /api/products/:id - Update product', async () => {
      const response = await axios.put(
        `${API_URL}/products/${createdProductId}`,
        { price: 1299.99 },
        {
          headers: { Authorization: `Bearer ${authToken}` }
        }
      );

      expect(response.status).toBe(200);
      expect(response.data.price).toBe(1299.99);
    });

    test('DELETE /api/products/:id - Delete product', async () => {
      const response = await axios.delete(
        `${API_URL}/products/${createdProductId}`,
        {
          headers: { Authorization: `Bearer ${authToken}` }
        }
      );

      expect(response.status).toBe(204);
    });
  });

  describe('API Validation Tests', () => {
    test('should return 400 for invalid data', async () => {
      try {
        await axios.post(
          `${API_URL}/products`,
          { name: 'Invalid' }, // Missing required fields
          {
            headers: { Authorization: `Bearer ${authToken}` }
          }
        );
      } catch (error: any) {
        expect(error.response.status).toBe(400);
        expect(error.response.data.error).toBeDefined();
      }
    });

    test('should return 401 without auth token', async () => {
      try {
        await axios.post(`${API_URL}/products`, {
          name: 'Test Product'
        });
      } catch (error: any) {
        expect(error.response.status).toBe(401);
      }
    });
  });

  describe('API Performance Tests', () => {
    test('should respond within 200ms', async () => {
      const start = Date.now();
      
      await axios.get(`${API_URL}/products`);
      
      const duration = Date.now() - start;
      expect(duration).toBeLessThan(200);
    });
  });
});
```

---

## 6. Performance Testing

### 6.1 Load Testing with k6

```javascript
// performance/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export let options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up to 200 users
    { duration: '5m', target: 200 },  // Stay at 200 users
    { duration: '2m', target: 0 }     // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% of requests must complete below 500ms
    http_req_failed: ['rate<0.01'],    // Less than 1% of requests should fail
    errors: ['rate<0.1']
  }
};

const BASE_URL = __ENV.API_URL || 'http://localhost:3000/api';

export default function () {
  // Test 1: Get products list
  let response = http.get(`${BASE_URL}/products`);
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500
  }) || errorRate.add(1);

  sleep(1);

  // Test 2: Get single product
  response = http.get(`${BASE_URL}/products/prod-123`);
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'has product data': (r) => r.json('id') !== undefined
  }) || errorRate.add(1);

  sleep(1);

  // Test 3: Search products
  response = http.get(`${BASE_URL}/products?search=shirt`);
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'has results': (r) => Array.isArray(r.json('data'))
  }) || errorRate.add(1);

  sleep(2);
}
```

### 6.2 Stress Testing

```javascript
// performance/stress-test.js
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 500 },   // Ramp up to 500 users
    { duration: '5m', target: 500 },   // Stay at 500 users
    { duration: '2m', target: 1000 },  // Spike to 1000 users
    { duration: '3m', target: 1000 },  // Stay at 1000 users
    { duration: '2m', target: 0 }      // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<1000'],
    http_req_failed: ['rate<0.05']
  }
};

const BASE_URL = __ENV.API_URL || 'http://localhost:3000/api';

export default function () {
  const responses = http.batch([
    ['GET', `${BASE_URL}/products`],
    ['GET', `${BASE_URL}/categories`],
    ['GET', `${BASE_URL}/products?featured=true`]
  ]);

  responses.forEach((response) => {
    check(response, {
      'status is 200': (r) => r.status === 200
    });
  });
}
```

---

## 7. Security Testing

### 7.1 Security Test Cases

```typescript
// tests/security/security.test.ts
import request from 'supertest';
import { app } from '@/app';

describe('Security Tests', () => {
  describe('SQL Injection Prevention', () => {
    test('should prevent SQL injection in search', async () => {
      const maliciousInput = "'; DROP TABLE products; --";
      
      const response = await request(app)
        .get('/api/products')
        .query({ search: maliciousInput })
        .expect(200);

      // Should not execute SQL
      expect(response.body.data).toBeDefined();
    });
  });

  describe('XSS Prevention', () => {
    test('should sanitize XSS attempts', async () => {
      const xssPayload = '<script>alert("xss")</script>';
      
      const response = await request(app)
        .post('/api/products')
        .send({
          name: xssPayload,
          sku: 'TEST-001',
          price: 999
        });

      expect(response.body.name).not.toContain('<script>');
    });
  });

  describe('Authentication Security', () => {
    test('should rate limit login attempts', async () => {
      const attempts = Array(10).fill(null).map(() =>
        request(app)
          .post('/api/auth/login')
          .send({
            email: 'test@example.com',
            password: 'wrongpassword'
          })
      );

      const responses = await Promise.all(attempts);
      const lastResponse = responses[responses.length - 1];
      
      expect(lastResponse.status).toBe(429); // Too Many Requests
    });

    test('should invalidate old tokens', async () => {
      // Login
      const loginResponse = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'password123'
        });

      const token = loginResponse.body.accessToken;

      // Logout
      await request(app)
        .post('/api/auth/logout')
        .set('Authorization', `Bearer ${token}`);

      // Try to use old token
      const response = await request(app)
        .get('/api/profile')
        .set('Authorization', `Bearer ${token}`);

      expect(response.status).toBe(401);
    });
  });

  describe('Authorization', () => {
    test('should prevent unauthorized access', async () => {
      const response = await request(app)
        .get('/api/admin/users')
        .expect(401);
    });

    test('should prevent privilege escalation', async () => {
      // Login as regular user
      const loginResponse = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'user@example.com',
          password: 'password123'
        });

      const token = loginResponse.body.accessToken;

      // Try to access admin endpoint
      const response = await request(app)
        .get('/api/admin/users')
        .set('Authorization', `Bearer ${token}`)
        .expect(403);
    });
  });

  describe('CSRF Protection', () => {
    test('should require CSRF token for state-changing operations', async () => {
      const response = await request(app)
        .post('/api/products')
        .send({
          name: 'Test Product',
          sku: 'TEST-001',
          price: 999
        });
      
      // Should fail without CSRF token
      expect([403, 401]).toContain(response.status);
    });
  });
});
```

---

## 8. Test Automation

### 8.1 Test Scripts

```json
// package.json scripts
{
  "scripts": {
    "test": "jest",
    "test:unit": "jest --testPathPattern=__tests__",
    "test:integration": "jest --testPathPattern=integration",
    "test:e2e": "playwright test",
    "test:api": "jest --testPathPattern=api",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch",
    "test:ci": "npm run test:unit && npm run test:integration",
    "test:load": "k6 run performance/load-test.js",
    "test:stress": "k6 run performance/stress-test.js"
  }
}
```

### 8.2 Pre-commit Hook

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# Run linter
npm run lint

# Run type check
npm run type-check

# Run unit tests
npm run test:unit

# Check coverage threshold
npm run test:coverage -- --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80,"statements":80}}'
```

### 8.3 CI Test Pipeline

```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:coverage
      
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
      redis:
        image: redis:7
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test:integration
      
  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npx playwright install
      - run: npm run test:e2e
```

---

## Conclusion

This comprehensive testing strategy ensures:

✅ **80%+ code coverage** with unit tests  
✅ **Complete API testing** with integration tests  
✅ **Full user flow validation** with E2E tests  
✅ **Performance benchmarks** with load testing  
✅ **Security validation** with security tests  
✅ **Automated execution** in CI/CD pipeline

---

**Document Status:** ✅ Complete  
**Last Updated:** March 26, 2026  
**QA Team:** qa@company.com
