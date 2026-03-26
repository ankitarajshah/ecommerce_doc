# Enterprise Men's Clothing Manufacturing & Sales Platform

[![Status](https://img.shields.io/badge/Status-Production%20Ready-green)]()
[![Version](https://img.shields.io/badge/Version-2.0-blue)]()
[![Documentation](https://img.shields.io/badge/Documentation-Complete-brightgreen)]()

## 📋 Overview

A comprehensive, scalable, enterprise-grade web platform for men's clothing manufacturing and sales business. This platform handles end-to-end operations including manufacturing, inventory management, multi-channel sales, marketplace integrations, CRM, and financial management.

## 🚀 Technology Stack

### Backend
- **Runtime:** Node.js 20 LTS
- **Framework:** Express.js 4.x
- **Language:** TypeScript 5.x
- **Database:** PostgreSQL 15+ (Neon.tech)
- **ORM:** Prisma 5.x
- **Cache:** Redis 7+ (Upstash)
- **Search:** PostgreSQL Full-Text Search
- **Queue:** Graphile Worker
- **Scheduler:** node-cron
- **Hosting:** Render.com (Free 750hrs/month)

### Frontend
- **Framework:** React.js 18+
- **Build Tool:** Vite 5.x
- **Language:** TypeScript
- **State Management:** Redux Toolkit + React Query
- **UI Library:** Headless UI + Tailwind CSS
- **Forms:** React Hook Form + Zod
- **Hosting:** Vercel (Free for projects)

### Infrastructure (Free Tier)
- **Backend Hosting:** Render.com (750 hours/month)
- **Database:** Neon.tech (10GB + 100hrs compute)
- **Cache:** Upstash Redis (10K commands/day)
- **File Storage:** Cloudinary (25GB storage + 25GB bandwidth)
- **Email Service:** Resend (3,000 emails/month)
- **Error Tracking:** Sentry (5,000 errors/month)
- **CI/CD:** GitHub Actions (Free for public repos)
- **Authentication:** JWT (Access + Refresh tokens)

## 📚 Documentation

Complete technical documentation with **12,250+ lines** across **9 comprehensive guides**:

| Document | Size | Description |
|----------|------|-------------|
| [00_MASTER_INDEX.md](docs/00_MASTER_INDEX.md) | 7.4KB | Central documentation hub |
| [01_SYSTEM_ARCHITECTURE.md](docs/01_SYSTEM_ARCHITECTURE.md) | 48KB | High-level system design |
| [02_DATABASE_SCHEMA.md](docs/02_DATABASE_SCHEMA.md) | 86KB | Complete database design |
| [03_BACKEND_ARCHITECTURE.md](docs/03_BACKEND_ARCHITECTURE.md) | 71KB | Node.js/Express implementation |
| [04_FRONTEND_ARCHITECTURE.md](docs/04_FRONTEND_ARCHITECTURE.md) | 45KB | React.js structure |
| [05_API_CONTRACTS.md](docs/05_API_CONTRACTS.md) | 29KB | REST API specifications |
| [06_PROJECT_FLOW.md](docs/06_PROJECT_FLOW.md) | 58KB | Business workflows |
| [07_SOP_ENGINE.md](docs/07_SOP_ENGINE.md) | 44KB | Approval workflows |
| [08_BUSINESS_LOGIC.md](docs/08_BUSINESS_LOGIC.md) | 37KB | Domain rules & calculations |
| [09_FREE_TIER_DEPLOYMENT.md](docs/09_FREE_TIER_DEPLOYMENT.md) | 35KB | Free tier deployment guide |

**Total Documentation:** 460KB

## 🎯 Key Features

### Manufacturing Operations
- ✅ Production planning & scheduling
- ✅ Bill of Materials (BOM) management
- ✅ Work order tracking
- ✅ Quality control & defect management
- ✅ Raw material procurement
- ✅ Multi-stage production workflow

### Inventory Management
- ✅ Multi-warehouse support
- ✅ Real-time stock tracking
- ✅ Automatic reorder points
- ✅ FIFO/LIFO stock allocation
- ✅ Batch & lot tracking
- ✅ Stock movement history

### Order Management
- ✅ Multi-channel order processing
- ✅ Order fulfillment workflow
- ✅ Payment gateway integration (Razorpay)
- ✅ Shipping integration (Delhivery, Blue Dart)
- ✅ Return & refund management
- ✅ Order tracking & notifications

### Marketplace Integration
- ✅ Amazon Seller Central
- ✅ Flipkart API
- ✅ Meesho API
- ✅ Automatic inventory sync
- ✅ Order import automation
- ✅ Price & product sync

### CRM & Customer Management
- ✅ Customer segmentation (Regular, VIP, Premium)
- ✅ Loyalty points system
- ✅ Credit limit management
- ✅ Support ticket system
- ✅ Multi-level agent commission
- ✅ Customer lifetime value tracking

### Analytics & Reporting
- ✅ Sales analytics dashboard
- ✅ Inventory reports
- ✅ Manufacturing metrics
- ✅ Agent performance tracking
- ✅ Financial reporting
- ✅ Custom report builder

## 🏗️ System Architecture (Free Tier)

```
┌─────────────────────────────────────────────────────────────┐
│              Client Layer (Vercel CDN - Free)                │
│  React.js + TypeScript + Tailwind CSS                       │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│          Application Layer (Render.com - Free)               │
│  Node.js + Express.js + TypeScript                          │
│  750 hours/month (24/7 for 1 month)                         │
└────────────────────┬────────────────────────────────────────┘
                     │
         ┌───────────┼───────────┬─────────────┐
         │           │           │             │
         ▼           ▼           ▼             ▼
┌──────────────┐ ┌──────────┐ ┌─────────┐ ┌────────────┐
│  Neon.tech   │ │  Upstash │ │Cloudinary│ │   Resend   │
│  PostgreSQL  │ │  Redis   │ │  Images  │ │   Email    │
│  10GB Free   │ │ 10K/day  │ │  25GB    │ │ 3K/month   │
└──────────────┘ └──────────┘ └─────────┘ └────────────┘
         │
         ▼
┌──────────────────────────────────┐
│  Sentry Error Tracking (Free)    │
│  5,000 errors/month              │
└──────────────────────────────────┘
```

## 🔐 Security Features

- JWT-based authentication with refresh tokens
- Role-based access control (RBAC)
- Permission-based authorization
- Rate limiting (100 req/15min)
- HTTPS enforcement
- SQL injection prevention (Prisma ORM)
- XSS protection
- CSRF protection
- Input validation & sanitization
- Secure password hashing (bcrypt)

## 📊 Database Design

**60+ tables** covering:
- Authentication & Authorization
- Product Catalog Management
- Inventory & Warehouse Management
- Manufacturing Operations
- Order & Payment Processing
- CRM & Customer Management
- Marketplace Integration
- Analytics & Reporting
- Audit Logs & Notifications

## 🔄 Key Workflows

### Order Fulfillment Flow
```
Customer Order → Payment → Inventory Check → Picking → 
Packing → Quality Check → Shipping → Delivery → 
Customer Review → Commission Calculation
```

### Manufacturing Workflow
```
Sales Forecast → Production Planning → Material Procurement → 
Work Order Creation → Cutting → Stitching → Quality Check → 
Finishing → Packaging → Inventory Update
```

### Agent Commission Flow
```
Order Delivered → Commission Calculation (Tier-based) → 
15-day Lock-in Period → Approval → Monthly Payout
```

## 🚦 Getting Started

### Prerequisites
- Node.js 20 LTS
- Git
- Free accounts on:
  - Render.com (Backend hosting)
  - Neon.tech (PostgreSQL database)
  - Vercel (Frontend hosting)
  - Cloudinary (Image storage)
  - Upstash (Redis cache)
  - Resend (Email service)
  - Sentry (Error tracking)

### Installation

```bash
# Clone repository
git clone https://github.com/yourcompany/ecommerce-platform.git
cd ecommerce-platform

# Install backend dependencies
npm install

# Install frontend dependencies
cd frontend
npm install

# Setup environment variables
cp .env.example .env
# Edit .env with your free service credentials

# Run database migrations
npx prisma migrate deploy

# Seed initial data
npm run seed

# Start development server
npm run dev
```

### Environment Variables (Free Tier)

```env
# Database (Neon.tech)
DATABASE_URL=postgresql://user:password@ep-xxx.neon.tech/ecommerce?sslmode=require

# Redis (Upstash)
REDIS_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-token

# JWT
JWT_SECRET=your-secret-key
JWT_REFRESH_SECRET=your-refresh-secret

# File Storage (Cloudinary)
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret

# Payment Gateway
RAZORPAY_KEY_ID=your-key-id
RAZORPAY_KEY_SECRET=your-key-secret

# Email (Resend)
RESEND_API_KEY=re_xxxxxxxxxxxx
FROM_EMAIL=noreply@yourdomain.com

# Error Tracking (Sentry)
SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx

# SMS (Optional)
SMS_API_KEY=your-sms-api-key
SMS_SENDER_ID=your-sender-id
```

## 🧪 Testing

```bash
# Run unit tests
npm test

# Run integration tests
npm run test:integration

# Run E2E tests
npm run test:e2e

# Test coverage
npm run test:coverage
```

## 📦 Deployment

### Free Tier Deployment (Render + Vercel + Neon)

**Step 1: Deploy Database (Neon.tech)**
```bash
# Sign up at neon.tech
# Create a new project
# Copy the connection string
# Update DATABASE_URL in your environment
```

**Step 2: Deploy Backend (Render.com)**
```bash
# Sign up at render.com
# Create new Web Service
# Connect your GitHub repository
# Set environment variables
# Deploy automatically on push
```

**Step 3: Deploy Frontend (Vercel)**
```bash
# Install Vercel CLI
npm i -g vercel

# Deploy from frontend directory
cd frontend
vercel

# Or connect GitHub repo on vercel.com
```

**Step 4: Setup Additional Services**
```bash
# Cloudinary: Sign up and get API keys
# Upstash: Create Redis database
# Resend: Add and verify domain
# Sentry: Create project and get DSN
```

### Docker Deployment (Local Development)

```bash
# Build Docker image
docker build -t ecommerce-platform .

# Run container
docker run -p 3000:3000 ecommerce-platform
```

### Production Scaling (When Ready to Pay)

See [09_FREE_TIER_DEPLOYMENT.md](docs/09_FREE_TIER_DEPLOYMENT.md) for:
- Migration from free to paid tiers
- AWS deployment guide
- Load balancing setup
- CDN configuration

## 📈 Performance Targets

**Free Tier (0-1000 users):**
- **Page Load Time:** < 3 seconds
- **API Response Time (p95):** < 500ms
- **Concurrent Users:** 100-500
- **Uptime:** 99%+ (with cold starts on Render)
- **Database Queries:** < 100ms (p95)

**Paid Tier (1000+ users):**
- **Page Load Time:** < 2 seconds
- **API Response Time (p95):** < 200ms
- **Concurrent Users:** 10,000+
- **Uptime:** 99.9%
- **Database Queries:** < 50ms (p95)

**Free Tier Limitations:**
- Render.com: 15-minute inactivity sleep (30s cold start)
- Neon.tech: Auto-pause after 5 minutes idle
- Upstash: 10,000 Redis commands/day limit
- Cloudinary: 25GB bandwidth/month
- Resend: 100 emails/day, 3,000/month

## 🔧 Development Guidelines

### Code Structure
```
src/
├── features/          # Feature modules
├── components/        # Shared components
├── services/          # Business logic
├── utils/             # Utility functions
├── types/             # TypeScript types
└── config/            # Configuration
```

### Commit Convention
```
feat: Add new feature
fix: Bug fix
docs: Documentation update
style: Code style changes
refactor: Code refactoring
test: Add/update tests
chore: Build process/tooling changes
```

## 📞 Support & Contact

- **Documentation:** [docs/00_MASTER_INDEX.md](docs/00_MASTER_INDEX.md)
- **Issues:** GitHub Issues
- **Email:** support@yourcompany.com
- **Slack:** #ecommerce-platform

## 📝 License

Copyright © 2026 Your Company. All rights reserved.

---

**Built with ❤️ for enterprise-grade e-commerce operations**

**Version:** 2.0  
**Last Updated:** March 26, 2026  
**Status:** ✅ Production Ready
