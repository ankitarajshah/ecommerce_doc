# 🚀 Free Tier Deployment Guide
## Enterprise E-Commerce Platform - Zero Cost Setup

**Document Version:** 1.0  
**Last Updated:** March 26, 2026  
**Target:** MVP Launch with 0-1000 Users

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Service Selection Rationale](#service-selection-rationale)
3. [Architecture Overview](#architecture-overview)
4. [Step-by-Step Deployment](#step-by-step-deployment)
5. [Environment Configuration](#environment-configuration)
6. [Database Setup](#database-setup)
7. [Backend Deployment](#backend-deployment)
8. [Frontend Deployment](#frontend-deployment)
9. [Service Integration](#service-integration)
10. [Monitoring & Error Tracking](#monitoring--error-tracking)
11. [Performance Optimization](#performance-optimization)
12. [Scaling Strategy](#scaling-strategy)
13. [Troubleshooting](#troubleshooting)

---

## 🎯 Overview

This guide provides a complete deployment strategy using **100% free services** for initial launch. The architecture supports 500-1000 active users with zero infrastructure costs.

### Free Tier Limits Summary

| Service | Free Tier | Monthly Limit | Annual Cost |
|---------|-----------|---------------|-------------|
| **Render.com** | 750 hours | 1 instance 24/7 | $0 |
| **Vercel** | Unlimited | Unlimited deployments | $0 |
| **Neon.tech** | 10GB + 100hrs compute | ~500K requests | $0 |
| **Upstash Redis** | 10K commands/day | 300K commands | $0 |
| **Cloudinary** | 25GB storage + 25GB bandwidth | ~5K images/day | $0 |
| **Resend** | 3,000 emails/month | 100 emails/day | $0 |
| **Sentry** | 5,000 errors/month | Real-time tracking | $0 |
| **GitHub Actions** | 2,000 minutes/month | Unlimited private repos | $0 |

**Total Monthly Cost:** $0  
**Estimated User Capacity:** 500-1000 active users

---

## 🔍 Service Selection Rationale

### Why NOT AWS Free Tier?

| Aspect | AWS Free Tier | Recommended Stack |
|--------|---------------|-------------------|
| **Duration** | ⚠️ 12 months only | ✅ Forever free |
| **Complexity** | ⚠️ High (VPC, IAM, etc.) | ✅ Simple setup |
| **Hidden Costs** | ⚠️ Easy to exceed | ✅ Hard limits |
| **Redis** | ❌ Not free | ✅ Free (Upstash) |
| **Elasticsearch** | ❌ $20-30/month | ✅ Free (PostgreSQL FTS) |
| **Setup Time** | ⚠️ 2-3 days | ✅ 2-3 hours |
| **Auto-scaling** | ⚠️ Manual | ✅ Automatic |

### Why These Services?

**Render.com vs AWS EC2:**
- ✅ No server management
- ✅ Auto-deploy from Git
- ✅ Free SSL certificates
- ✅ Built-in monitoring
- ✅ Zero DevOps knowledge needed

**Neon.tech vs AWS RDS:**
- ✅ Forever free (not 12 months)
- ✅ 10GB vs AWS's 20GB
- ✅ Serverless (auto-pause when idle)
- ✅ Instant branching for testing
- ✅ Connection pooling built-in

**Upstash vs AWS ElastiCache:**
- ✅ ElastiCache has NO free tier
- ✅ Upstash: 10K commands/day free
- ✅ Serverless (pay-per-use)
- ✅ Global edge caching

**Cloudinary vs AWS S3:**
- ✅ 25GB vs S3's 5GB
- ✅ Image transformation included
- ✅ CDN included
- ✅ No complex pricing

---

## 🏗️ Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│                    DNS (Cloudflare - Free)                  │
│              yourdomain.com → Vercel + Render              │
└────────────────────┬───────────────────────────────────────┘
                     │
         ┌───────────┴────────────┐
         │                        │
         ▼                        ▼
┌─────────────────┐     ┌──────────────────────┐
│  Vercel (Free)  │     │  Render.com (Free)   │
│  Frontend CDN   │────▶│  Backend API         │
│  React + Vite   │     │  Node.js + Express   │
│  Global Edge    │     │  750 hrs/month       │
└─────────────────┘     └──────────┬───────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
     ┌────────────────┐   ┌────────────────┐  ┌────────────────┐
     │  Neon.tech     │   │  Upstash       │  │  Cloudinary    │
     │  PostgreSQL    │   │  Redis Cache   │  │  Media Storage │
     │  10GB Storage  │   │  10K/day       │  │  25GB + CDN    │
     └────────────────┘   └────────────────┘  └────────────────┘
              │                    │                    │
              └────────────────────┴────────────────────┘
                                   │
                         ┌─────────┴──────────┐
                         │                    │
                         ▼                    ▼
                ┌────────────────┐   ┌────────────────┐
                │  Resend        │   │  Sentry        │
                │  Email Service │   │  Error Track   │
                │  3K/month      │   │  5K/month      │
                └────────────────┘   └────────────────┘
```

### Request Flow

```
User Request
    │
    ▼
Vercel CDN (React App)
    │
    ▼
API Call to Render.com
    │
    ├──▶ Check Redis Cache (Upstash)
    │    │
    │    ├──▶ Cache Hit → Return
    │    │
    │    └──▶ Cache Miss ──┐
    │                      ▼
    └──────────────▶ Query PostgreSQL (Neon)
                            │
                            ▼
                    Update Redis Cache
                            │
                            ▼
                    Return Response
```

---

## 📦 Step-by-Step Deployment

### Phase 1: Initial Setup (30 minutes)

#### 1. Sign Up for All Services

```bash
# Create accounts (no credit card needed except Render)
1. Render.com      → https://render.com
2. Vercel          → https://vercel.com
3. Neon.tech       → https://neon.tech
4. Upstash         → https://upstash.com
5. Cloudinary      → https://cloudinary.com
6. Resend          → https://resend.com
7. Sentry          → https://sentry.io
```

#### 2. Prepare Your Repository

```bash
# Clone your repository
git clone https://github.com/yourusername/clothing-ecom.git
cd clothing-ecom

# Create necessary files
touch .env.example
touch render.yaml
touch vercel.json

# Push to GitHub (all services deploy from Git)
git add .
git commit -m "Setup for free tier deployment"
git push origin main
```

---

### Phase 2: Database Setup (Neon.tech)

#### Step 1: Create Database

```bash
# 1. Go to https://console.neon.tech
# 2. Click "Create Project"
# 3. Project Name: clothing-ecom-prod
# 4. PostgreSQL Version: 15
# 5. Region: Choose closest to target users
#    - US East (Ohio) for US customers
#    - Europe (Frankfurt) for EU customers
#    - Asia Pacific (Singapore) for Asian customers
```

#### Step 2: Get Connection String

```bash
# Connection string format:
postgresql://[user]:[password]@[endpoint].neon.tech/[database]?sslmode=require

# Example:
postgresql://alex:AbC123xyz@ep-cool-darkness-123456.us-east-2.aws.neon.tech/clothing_ecom?sslmode=require
```

#### Step 3: Configure Connection Pooling

```bash
# For Prisma, use pooled connection:
postgresql://[user]:[password]@[endpoint]-pooler.neon.tech/[database]?sslmode=require&pgbouncer=true

# Benefits:
# - Handle 100+ concurrent connections
# - Auto-scaling with traffic
# - No "too many connections" errors
```

#### Step 4: Run Migrations

```bash
# Install Prisma CLI
npm install -g prisma

# Set DATABASE_URL
export DATABASE_URL="your-neon-connection-string"

# Run migrations
npx prisma migrate deploy

# Seed initial data
npm run seed

# Verify
npx prisma studio
```

#### Step 5: Configure Auto-Pause Settings

```bash
# In Neon Dashboard:
# Settings → Compute → Autosuspend Delay
# Recommended: 5 minutes (free tier default)

# What happens:
# - Database pauses after 5 min of inactivity
# - First query after pause takes ~500ms (cold start)
# - Subsequent queries are fast (~50ms)
```

---

### Phase 3: Redis Cache Setup (Upstash)

#### Step 1: Create Redis Database

```bash
# 1. Go to https://console.upstash.com
# 2. Click "Create Database"
# 3. Name: clothing-ecom-cache
# 4. Type: Regional (for free tier)
# 5. Region: Choose same as Neon database
# 6. Eviction: allkeys-lru (recommended)
```

#### Step 2: Get Credentials

```bash
# You'll get:
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=AxxxxxxxxxxxxxxxxxxxxxxxxxxxxQ

# For Node.js with ioredis:
REDIS_URL=rediss://default:[password]@xxx.upstash.io:6379
```

#### Step 3: Configure Redis Client

```typescript
// src/config/redis.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL, {
  maxRetriesPerRequest: 3,
  enableReadyCheck: false,
  tls: {
    rejectUnauthorized: false
  }
});

// Graceful error handling
redis.on('error', (error) => {
  console.error('Redis connection error:', error);
});

export default redis;
```

#### Step 4: Implement Caching Strategy

```typescript
// src/utils/cache.ts
import redis from '../config/redis';

export const cacheMiddleware = (duration: number = 300) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    const key = `cache:${req.originalUrl}`;
    
    try {
      const cachedData = await redis.get(key);
      
      if (cachedData) {
        return res.json(JSON.parse(cachedData));
      }
      
      // Store original send
      const originalSend = res.json;
      
      res.json = function(data: any) {
        // Cache the response
        redis.setex(key, duration, JSON.stringify(data));
        return originalSend.call(this, data);
      };
      
      next();
    } catch (error) {
      // If Redis fails, continue without caching
      console.error('Cache error:', error);
      next();
    }
  };
};

// Usage in routes:
router.get('/products', cacheMiddleware(600), getProducts);
router.get('/categories', cacheMiddleware(1800), getCategories);
```

---

### Phase 4: Backend Deployment (Render.com)

#### Step 1: Create Web Service

```bash
# 1. Go to https://dashboard.render.com
# 2. Click "New +" → "Web Service"
# 3. Connect your GitHub repository
# 4. Configure:

Name: clothing-ecom-api
Environment: Node
Region: Same as database
Branch: main
Build Command: npm install && npm run build
Start Command: npm start
Plan: Free (750 hours/month)
```

#### Step 2: Configure Environment Variables

```bash
# In Render Dashboard → Environment
# Add all variables:

NODE_ENV=production
PORT=3000

# Database (Neon)
DATABASE_URL=postgresql://...neon.tech/...?sslmode=require

# Redis (Upstash)
REDIS_URL=rediss://...upstash.io:6379

# JWT
JWT_SECRET=your-super-secret-key-min-32-chars
JWT_REFRESH_SECRET=another-secret-key-different-from-above
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# Cloudinary
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=123456789012345
CLOUDINARY_API_SECRET=abcdefghijklmnopqrstuvwxy

# Resend Email
RESEND_API_KEY=re_xxxxxxxxxxxx
FROM_EMAIL=noreply@yourdomain.com

# Sentry
SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx

# Razorpay (Payment)
RAZORPAY_KEY_ID=rzp_test_xxxxxxxxxxxx
RAZORPAY_KEY_SECRET=xxxxxxxxxxxxxxxxxxxx

# App Config
FRONTEND_URL=https://yourdomain.com
CORS_ORIGIN=https://yourdomain.com

# Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100
```

#### Step 3: Configure Health Check

```bash
# Render checks this endpoint to ensure app is running
# Create: src/routes/health.ts

import express from 'express';
const router = express.Router();

router.get('/health', async (req, res) => {
  try {
    // Check database connection
    await prisma.$queryRaw`SELECT 1`;
    
    // Check Redis connection
    await redis.ping();
    
    res.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      environment: process.env.NODE_ENV
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message
    });
  }
});

export default router;
```

#### Step 4: Configure Auto-Deploy

```bash
# In Render Dashboard → Settings → Build & Deploy
# Enable: Auto-Deploy → Yes

# Now every push to main branch will:
# 1. Trigger automatic build
# 2. Run tests (if configured)
# 3. Deploy new version
# 4. Zero downtime deployment
```

#### Step 5: Handle Cold Starts

```bash
# Render free tier sleeps after 15 minutes of inactivity
# First request after sleep takes ~30 seconds

# Solution 1: Keep-alive ping (external service)
# Use cron-job.org or UptimeRobot to ping every 14 minutes

# Solution 2: Warn users in frontend
# Show loading message: "Waking up server, please wait..."

# Solution 3: Upgrade to paid tier ($7/month - no sleep)
```

---

### Phase 5: Frontend Deployment (Vercel)

#### Step 1: Prepare Frontend

```bash
# Navigate to frontend directory
cd frontend

# Create vercel.json
cat > vercel.json << 'EOF'
{
  "version": 2,
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/static-build",
      "config": {
        "distDir": "dist"
      }
    }
  ],
  "routes": [
    {
      "src": "/assets/(.*)",
      "headers": {
        "cache-control": "public, max-age=31536000, immutable"
      }
    },
    {
      "src": "/(.*)",
      "dest": "/index.html"
    }
  ],
  "env": {
    "VITE_API_URL": "@api_url",
    "VITE_CLOUDINARY_CLOUD_NAME": "@cloudinary_name",
    "VITE_SENTRY_DSN": "@sentry_dsn"
  }
}
EOF
```

#### Step 2: Deploy to Vercel

```bash
# Method 1: Vercel CLI
npm i -g vercel
vercel login
vercel --prod

# Method 2: GitHub Integration (Recommended)
# 1. Go to https://vercel.com/new
# 2. Import Git Repository
# 3. Select your GitHub repo
# 4. Configure:
#    - Framework Preset: Vite
#    - Root Directory: frontend
#    - Build Command: npm run build
#    - Output Directory: dist
# 5. Add Environment Variables
# 6. Click "Deploy"
```

#### Step 3: Configure Environment Variables

```bash
# In Vercel Dashboard → Settings → Environment Variables

VITE_API_URL=https://your-app.onrender.com/api/v1
VITE_CLOUDINARY_CLOUD_NAME=your-cloud-name
VITE_SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx
VITE_RAZORPAY_KEY_ID=rzp_test_xxxxxxxxxxxx
VITE_APP_NAME=Clothing E-Commerce
VITE_APP_VERSION=1.0.0
```

#### Step 4: Configure Custom Domain

```bash
# In Vercel Dashboard → Settings → Domains
# Add your domain: yourdomain.com

# Update DNS (Cloudflare, GoDaddy, etc.):
# Type: CNAME
# Name: @
# Value: cname.vercel-dns.com

# For www subdomain:
# Type: CNAME
# Name: www
# Value: cname.vercel-dns.com

# SSL certificate is automatically provisioned (free)
```

---

### Phase 6: File Storage Setup (Cloudinary)

#### Step 1: Configure Cloudinary

```bash
# 1. Sign up at https://cloudinary.com
# 2. Dashboard → Settings → Upload
# 3. Configure:
#    - Upload Preset: Create unsigned preset
#    - Folder: clothing-ecom
#    - Allowed formats: jpg, png, webp
#    - Max file size: 10MB
```

#### Step 2: Create Upload API

```typescript
// src/routes/upload.ts
import { v2 as cloudinary } from 'cloudinary';
import multer from 'multer';
import { Readable } from 'stream';

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET
});

// Configure multer for memory storage
const upload = multer({
  storage: multer.memoryStorage(),
  limits: {
    fileSize: 10 * 1024 * 1024 // 10MB
  },
  fileFilter: (req, file, cb) => {
    const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
    if (allowedTypes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type. Only JPEG, PNG, and WebP are allowed.'));
    }
  }
});

// Upload endpoint
router.post('/upload', 
  authenticate, 
  upload.single('image'), 
  async (req, res) => {
    try {
      if (!req.file) {
        return res.status(400).json({ error: 'No file uploaded' });
      }

      // Convert buffer to stream
      const stream = Readable.from(req.file.buffer);

      // Upload to Cloudinary
      const result = await new Promise((resolve, reject) => {
        const uploadStream = cloudinary.uploader.upload_stream(
          {
            folder: 'clothing-ecom/products',
            transformation: [
              { width: 1000, height: 1000, crop: 'limit' },
              { quality: 'auto', fetch_format: 'auto' }
            ]
          },
          (error, result) => {
            if (error) reject(error);
            else resolve(result);
          }
        );

        stream.pipe(uploadStream);
      });

      res.json({
        url: result.secure_url,
        public_id: result.public_id,
        width: result.width,
        height: result.height
      });

    } catch (error) {
      console.error('Upload error:', error);
      res.status(500).json({ error: 'Upload failed' });
    }
  }
);

// Delete endpoint
router.delete('/upload/:publicId', 
  authenticate, 
  async (req, res) => {
    try {
      await cloudinary.uploader.destroy(req.params.publicId);
      res.json({ message: 'Image deleted successfully' });
    } catch (error) {
      res.status(500).json({ error: 'Delete failed' });
    }
  }
);
```

#### Step 3: Frontend Upload Component

```typescript
// frontend/src/components/ImageUpload.tsx
import { useState } from 'react';
import axios from 'axios';

export const ImageUpload = ({ onUpload }) => {
  const [uploading, setUploading] = useState(false);
  const [preview, setPreview] = useState<string | null>(null);

  const handleUpload = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    // Show preview
    const reader = new FileReader();
    reader.onload = () => setPreview(reader.result as string);
    reader.readAsDataURL(file);

    setUploading(true);

    try {
      const formData = new FormData();
      formData.append('image', file);

      const response = await axios.post('/api/v1/upload', formData, {
        headers: { 'Content-Type': 'multipart/form-data' }
      });

      onUpload(response.data.url);
    } catch (error) {
      console.error('Upload failed:', error);
      alert('Upload failed. Please try again.');
    } finally {
      setUploading(false);
    }
  };

  return (
    <div className="image-upload">
      <input
        type="file"
        accept="image/jpeg,image/png,image/webp"
        onChange={handleUpload}
        disabled={uploading}
      />
      {preview && (
        <img src={preview} alt="Preview" className="preview" />
      )}
      {uploading && <p>Uploading...</p>}
    </div>
  );
};
```

---

### Phase 7: Email Service Setup (Resend)

#### Step 1: Configure Resend

```bash
# 1. Sign up at https://resend.com
# 2. Add and verify your domain
# 3. Add DNS records:

# For sending from noreply@yourdomain.com:
Type: TXT
Name: @
Value: [Provided by Resend]

Type: CNAME
Name: resend._domainkey
Value: [Provided by Resend]

# Wait for DNS propagation (5-30 minutes)
```

#### Step 2: Create Email Templates

```typescript
// src/services/email.service.ts
import { Resend } from 'resend';

const resend = new Resend(process.env.RESEND_API_KEY);

export const emailService = {
  // Order confirmation
  async sendOrderConfirmation(order: Order) {
    return await resend.emails.send({
      from: 'orders@yourdomain.com',
      to: order.customer.email,
      subject: `Order Confirmation - #${order.orderNumber}`,
      html: `
        <h1>Thank you for your order!</h1>
        <p>Order Number: <strong>${order.orderNumber}</strong></p>
        <p>Total: ₹${order.total}</p>
        <h2>Items:</h2>
        <ul>
          ${order.items.map(item => `
            <li>${item.product.name} - ${item.quantity} x ₹${item.price}</li>
          `).join('')}
        </ul>
        <p>You can track your order at: 
          <a href="${process.env.FRONTEND_URL}/orders/${order.id}">
            Track Order
          </a>
        </p>
      `
    });
  },

  // Shipping notification
  async sendShippingNotification(shipment: Shipment) {
    return await resend.emails.send({
      from: 'shipping@yourdomain.com',
      to: shipment.order.customer.email,
      subject: `Your order has been shipped - #${shipment.order.orderNumber}`,
      html: `
        <h1>Your order is on the way!</h1>
        <p>Tracking Number: <strong>${shipment.trackingNumber}</strong></p>
        <p>Carrier: ${shipment.carrier}</p>
        <p>Track at: 
          <a href="${shipment.trackingUrl}">${shipment.trackingUrl}</a>
        </p>
      `
    });
  },

  // Password reset
  async sendPasswordReset(user: User, resetToken: string) {
    const resetUrl = `${process.env.FRONTEND_URL}/reset-password?token=${resetToken}`;
    
    return await resend.emails.send({
      from: 'noreply@yourdomain.com',
      to: user.email,
      subject: 'Password Reset Request',
      html: `
        <h1>Password Reset</h1>
        <p>Click the link below to reset your password:</p>
        <a href="${resetUrl}">${resetUrl}</a>
        <p>This link expires in 1 hour.</p>
        <p>If you didn't request this, please ignore this email.</p>
      `
    });
  },

  // Low stock alert (internal)
  async sendLowStockAlert(product: Product) {
    return await resend.emails.send({
      from: 'alerts@yourdomain.com',
      to: 'inventory@yourdomain.com',
      subject: `Low Stock Alert - ${product.name}`,
      html: `
        <h1>Low Stock Alert</h1>
        <p>Product: <strong>${product.name}</strong></p>
        <p>SKU: ${product.sku}</p>
        <p>Current Stock: ${product.stock} units</p>
        <p>Reorder Point: ${product.reorderPoint} units</p>
        <p><a href="${process.env.FRONTEND_URL}/admin/products/${product.id}">
          View Product
        </a></p>
      `
    });
  }
};
```

#### Step 3: Implement Email Queue

```typescript
// src/queues/email.queue.ts
import { emailService } from '../services/email.service';

// Use Graphile Worker for background jobs
export const emailQueue = {
  async sendEmail(task: EmailTask) {
    const { type, data } = task;

    try {
      switch (type) {
        case 'order_confirmation':
          await emailService.sendOrderConfirmation(data);
          break;
        
        case 'shipping_notification':
          await emailService.sendShippingNotification(data);
          break;
        
        case 'password_reset':
          await emailService.sendPasswordReset(data.user, data.token);
          break;
        
        case 'low_stock_alert':
          await emailService.sendLowStockAlert(data);
          break;
      }

      console.log(`Email sent: ${type}`);
    } catch (error) {
      console.error(`Email failed: ${type}`, error);
      throw error; // Will retry based on Graphile Worker config
    }
  }
};

// Queue email in your route/service:
await addJob('send_email', {
  type: 'order_confirmation',
  data: order
});
```

---

### Phase 8: Error Tracking Setup (Sentry)

#### Step 1: Configure Sentry Backend

```typescript
// src/config/sentry.ts
import * as Sentry from '@sentry/node';
import { ProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0, // Capture 100% of transactions
  profilesSampleRate: 1.0, // Profile 100% of transactions

  integrations: [
    new ProfilingIntegration(),
  ],

  // Filter out health check requests
  beforeSend(event, hint) {
    if (event.request?.url?.includes('/health')) {
      return null;
    }
    return event;
  },
});

export default Sentry;
```

#### Step 2: Add Sentry Middleware

```typescript
// src/app.ts
import Sentry from './config/sentry';

const app = express();

// Sentry request handler (must be first)
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// ... your routes ...

// Sentry error handler (must be before other error handlers)
app.use(Sentry.Handlers.errorHandler());

// Custom error handler
app.use((err, req, res, next) => {
  console.error(err);
  res.status(500).json({
    error: 'Internal server error',
    message: process.env.NODE_ENV === 'development' ? err.message : undefined
  });
});
```

#### Step 3: Configure Sentry Frontend

```typescript
// frontend/src/main.tsx
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  integrations: [
    new Sentry.BrowserTracing({
      tracePropagationTargets: ['localhost', /^https:\/\/yourapi\.com/],
    }),
    new Sentry.Replay({
      maskAllText: false,
      blockAllMedia: false,
    }),
  ],
  tracesSampleRate: 1.0,
  replaysSessionSampleRate: 0.1, // 10% of sessions
  replaysOnErrorSampleRate: 1.0, // 100% when error occurs
});
```

#### Step 4: Custom Error Tracking

```typescript
// frontend/src/utils/errorTracking.ts
import * as Sentry from '@sentry/react';

export const trackError = (error: Error, context?: Record<string, any>) => {
  console.error(error);
  
  Sentry.captureException(error, {
    extra: context,
    tags: {
      component: context?.component,
      action: context?.action
    }
  });
};

// Usage:
try {
  await processOrder(order);
} catch (error) {
  trackError(error, {
    component: 'OrderCheckout',
    action: 'processPayment',
    orderId: order.id
  });
}
```

---

## 🎯 Environment Configuration

### Complete .env.example

```bash
# ============================================
# APPLICATION
# ============================================
NODE_ENV=production
PORT=3000
FRONTEND_URL=https://yourdomain.com
API_VERSION=v1

# ============================================
# DATABASE (Neon.tech)
# ============================================
# Pooled connection (recommended for production)
DATABASE_URL=postgresql://user:password@ep-xxx-pooler.neon.tech/dbname?sslmode=require&pgbouncer=true

# Direct connection (for migrations)
DIRECT_DATABASE_URL=postgresql://user:password@ep-xxx.neon.tech/dbname?sslmode=require

# ============================================
# REDIS CACHE (Upstash)
# ============================================
REDIS_URL=rediss://default:password@xxx.upstash.io:6379

# OR use REST API (for serverless)
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=AxxxxxxxxxxxxxxxxxxxxQ

# ============================================
# JWT AUTHENTICATION
# ============================================
JWT_SECRET=your-super-secret-key-minimum-32-characters-long
JWT_REFRESH_SECRET=another-different-secret-key-for-refresh-tokens
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# ============================================
# FILE STORAGE (Cloudinary)
# ============================================
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=123456789012345
CLOUDINARY_API_SECRET=abcdefghijklmnopqrstuvwxyzABCDEF

# ============================================
# EMAIL SERVICE (Resend)
# ============================================
RESEND_API_KEY=re_xxxxxxxxxxxxxxxxxxxxxxxxxxxx
FROM_EMAIL=noreply@yourdomain.com
SUPPORT_EMAIL=support@yourdomain.com

# ============================================
# ERROR TRACKING (Sentry)
# ============================================
SENTRY_DSN=https://xxxxxxx@xxxxxxx.ingest.sentry.io/1234567

# ============================================
# PAYMENT GATEWAY (Razorpay)
# ============================================
RAZORPAY_KEY_ID=rzp_test_xxxxxxxxxxxx
RAZORPAY_KEY_SECRET=xxxxxxxxxxxxxxxxxxxx
RAZORPAY_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx

# ============================================
# SMS GATEWAY (Optional - MSG91)
# ============================================
SMS_API_KEY=your-sms-api-key
SMS_SENDER_ID=TXTLCL
SMS_ENABLED=false

# ============================================
# CORS CONFIGURATION
# ============================================
CORS_ORIGIN=https://yourdomain.com
CORS_CREDENTIALS=true

# ============================================
# RATE LIMITING
# ============================================
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100

# ============================================
# SESSION CONFIGURATION
# ============================================
SESSION_SECRET=another-secret-for-sessions
SESSION_MAX_AGE=86400000

# ============================================
# MARKETPLACE APIs (Optional)
# ============================================
AMAZON_SELLER_ID=
AMAZON_MWS_AUTH_TOKEN=
FLIPKART_APP_ID=
FLIPKART_APP_SECRET=
MEESHO_API_KEY=

# ============================================
# SHIPPING PROVIDERS (Optional)
# ============================================
DELHIVERY_API_KEY=
DELHIVERY_CLIENT_ID=
BLUEDART_API_KEY=

# ============================================
# MONITORING & LOGGING
# ============================================
LOG_LEVEL=info
ENABLE_REQUEST_LOGGING=true
```

---

## 🚀 Performance Optimization

### 1. Database Optimization

```typescript
// Enable Prisma query optimization
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "metrics"]
}

// Use connection pooling
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
  directUrl = env("DIRECT_DATABASE_URL")
}
```

```typescript
// Implement efficient queries
// Bad: N+1 query problem
const orders = await prisma.order.findMany();
for (const order of orders) {
  const customer = await prisma.customer.findUnique({
    where: { id: order.customerId }
  });
}

// Good: Use include/select
const orders = await prisma.order.findMany({
  include: {
    customer: {
      select: { id: true, name: true, email: true }
    },
    items: {
      include: { product: true }
    }
  }
});
```

### 2. Redis Caching Strategy

```typescript
// Cache frequently accessed data
export const cacheStrategies = {
  // Product list (30 minutes)
  products: 1800,
  
  // Categories (1 hour)
  categories: 3600,
  
  // User session (24 hours)
  session: 86400,
  
  // Cart (1 hour)
  cart: 3600,
};

// Implement cache-aside pattern
export async function getProducts() {
  const cacheKey = 'products:list';
  
  // Try cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // Query database
  const products = await prisma.product.findMany();
  
  // Store in cache
  await redis.setex(
    cacheKey, 
    cacheStrategies.products, 
    JSON.stringify(products)
  );
  
  return products;
}
```

### 3. Frontend Optimization

```typescript
// Lazy load routes
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Products = lazy(() => import('./pages/Products'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/products" element={<Products />} />
      </Routes>
    </Suspense>
  );
}

// Image optimization with Cloudinary
const optimizedImage = (url: string, width: number) => {
  return url.replace(
    '/upload/',
    `/upload/w_${width},f_auto,q_auto/`
  );
};

// Use React Query for caching
import { useQuery } from '@tanstack/react-query';

function ProductList() {
  const { data, isLoading } = useQuery({
    queryKey: ['products'],
    queryFn: fetchProducts,
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 30 * 60 * 1000, // 30 minutes
  });
}
```

---

## 📊 Scaling Strategy

### Phase 1: Free Tier (0-1000 users)
**Current Setup - $0/month**

```
Capacity:
- 500-1000 concurrent users
- 10,000 page views/day
- 1,000 orders/day
- 10GB database
- 300K Redis operations/day
```

### Phase 2: Low-Cost Tier (1000-5000 users)
**Estimated Cost: $30-50/month**

```
Upgrades:
- Render: Starter ($7/month) - No sleep, 512MB RAM
- Neon: Launch ($19/month) - 50GB, always-on compute
- Upstash: Pay-as-you-go (~$5/month)
- Keep: Vercel, Cloudinary, Resend free tiers
```

### Phase 3: Production Tier (5000-50,000 users)
**Estimated Cost: $200-500/month**

```
Migration to AWS/DO:
- DigitalOcean App Platform ($12/month per service)
- Managed PostgreSQL ($15-60/month)
- Redis managed ($15-50/month)
- Cloudinary Pro ($89/month)
- CDN (CloudFlare Pro $20/month)
```

### Phase 4: Enterprise Scale (50,000+ users)
**Estimated Cost: $1000+/month**

```
Full AWS infrastructure:
- ECS Fargate auto-scaling
- RDS Multi-AZ
- ElastiCache cluster
- S3 + CloudFront
- Application Load Balancer
```

---

## 🔧 Troubleshooting

### Common Issues & Solutions

#### 1. Render Cold Start (15min sleep)

**Problem:** First request takes 30 seconds after inactivity

**Solutions:**
```bash
# Option A: Keep-alive service (free)
# Use cron-job.org to ping every 14 minutes
# URL: https://your-app.onrender.com/health
# Interval: */14 * * * *

# Option B: Frontend handling
// Show friendly message during cold start
const api = axios.create({
  baseURL: process.env.VITE_API_URL,
  timeout: 60000 // 60 seconds timeout
});

api.interceptors.request.use((config) => {
  showLoading('Connecting to server...');
  return config;
});

# Option C: Upgrade to paid ($7/month - no sleep)
```

#### 2. Neon Database Auto-Pause

**Problem:** First query after pause takes 500ms

**Solutions:**
```typescript
// Implement connection warming
setInterval(async () => {
  await prisma.$queryRaw`SELECT 1`;
}, 4 * 60 * 1000); // Every 4 minutes

// Or upgrade to scale plan for always-on compute
```

#### 3. Upstash Daily Limit Exceeded

**Problem:** Exceeding 10K commands/day

**Solutions:**
```typescript
// Implement graceful fallback
export async function getCached(key: string) {
  try {
    return await redis.get(key);
  } catch (error) {
    console.warn('Redis unavailable, skipping cache');
    return null; // Fall back to database
  }
}

// Optimize cache usage
// Bad: Cache every request
// Good: Cache only heavy queries

// Monitor usage
const dailyCount = await redis.incr('usage:count');
if (dailyCount > 9000) {
  console.warn('Approaching Redis limit');
  // Reduce caching temporarily
}
```

#### 4. Cloudinary Bandwidth Limit

**Problem:** 25GB/month bandwidth exceeded

**Solutions:**
```typescript
// Optimize image delivery
const imageUrl = (publicId: string, options = {}) => {
  const { width = 800, quality = 'auto' } = options;
  
  return `https://res.cloudinary.com/${cloudName}/image/upload/` +
         `w_${width},q_${quality},f_auto/${publicId}`;
};

// Use responsive images
<img 
  srcSet={`
    ${imageUrl(id, { width: 400 })} 400w,
    ${imageUrl(id, { width: 800 })} 800w,
    ${imageUrl(id, { width: 1200 })} 1200w
  `}
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1200px"
/>

// Or upgrade to $89/month for 65GB
```

#### 5. Resend Email Limit (100/day)

**Problem:** Exceeding daily email limit

**Solutions:**
```typescript
// Implement email queuing
const emailQueue = [];
let dailyCount = 0;

export async function queueEmail(email: EmailData) {
  if (dailyCount >= 95) {
    // Store in database for next day
    await prisma.emailQueue.create({ data: email });
    return;
  }
  
  await sendEmail(email);
  dailyCount++;
}

// Reset counter daily
cron.schedule('0 0 * * *', () => {
  dailyCount = 0;
  // Process queued emails
});

// Or upgrade to $20/month for 50K emails
```

---

## ✅ Deployment Checklist

### Pre-Deployment

- [ ] All environment variables configured
- [ ] Database migrations completed
- [ ] Seed data loaded
- [ ] API endpoints tested
- [ ] CORS configured correctly
- [ ] Rate limiting enabled
- [ ] Error tracking setup
- [ ] Email templates tested
- [ ] Payment gateway in test mode
- [ ] Security headers configured

### Post-Deployment

- [ ] Health check endpoint responding
- [ ] Database connections working
- [ ] Redis cache functioning
- [ ] File uploads working
- [ ] Email delivery verified
- [ ] Error tracking receiving events
- [ ] Frontend connected to API
- [ ] SSL certificates active
- [ ] Custom domain configured
- [ ] Monitoring dashboard setup

### Ongoing Maintenance

- [ ] Monitor error rates (Sentry)
- [ ] Check service quotas daily
- [ ] Review API performance
- [ ] Database backup verification
- [ ] Security updates applied
- [ ] User feedback collection
- [ ] Performance metrics tracking
- [ ] Cost monitoring

---

## 📈 Monitoring & Alerts

### Set Up Monitoring

```typescript
// Create monitoring endpoint
router.get('/metrics', async (req, res) => {
  const metrics = {
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    
    database: {
      connected: await checkDatabaseConnection(),
      activeConnections: await getActiveConnections(),
    },
    
    redis: {
      connected: await checkRedisConnection(),
      usedMemory: await getRedisMemory(),
    },
    
    api: {
      requestsPerMinute: await getRequestRate(),
      averageResponseTime: await getAvgResponseTime(),
      errorRate: await getErrorRate(),
    }
  };
  
  res.json(metrics);
});
```

### Configure Alerts

```bash
# Use UptimeRobot (free) for uptime monitoring
# 1. Add monitors for:
#    - Frontend: https://yourdomain.com
#    - Backend: https://api.yourdomain.com/health
#    - Check interval: 5 minutes
# 2. Configure alerts:
#    - Email when down
#    - Email when back up
#    - Weekly summary report
```

---

## 🎓 Best Practices

### 1. Security
- ✅ Always use HTTPS (automatic with Vercel/Render)
- ✅ Never commit secrets to Git
- ✅ Rotate JWT secrets regularly
- ✅ Implement rate limiting
- ✅ Validate all inputs
- ✅ Use Prisma for SQL injection prevention
- ✅ Enable CORS only for your domain

### 2. Performance
- ✅ Cache aggressively but invalidate properly
- ✅ Use database indexes on frequently queried fields
- ✅ Implement pagination for large lists
- ✅ Optimize images through Cloudinary
- ✅ Use CDN for static assets
- ✅ Implement lazy loading
- ✅ Monitor slow queries

### 3. Reliability
- ✅ Implement health checks
- ✅ Use queue for background jobs
- ✅ Retry failed operations
- ✅ Log errors to Sentry
- ✅ Have database backups
- ✅ Test error scenarios
- ✅ Plan for service outages

### 4. Cost Management
- ✅ Monitor service usage daily
- ✅ Set up billing alerts
- ✅ Optimize cache usage
- ✅ Compress images
- ✅ Clean up unused data
- ✅ Review quotas weekly
- ✅ Plan upgrades before hitting limits

---

## 📝 Next Steps

### Week 1: Setup & Deploy
- [ ] Create all service accounts
- [ ] Deploy database and run migrations
- [ ] Deploy backend to Render
- [ ] Deploy frontend to Vercel
- [ ] Configure custom domain
- [ ] Test all endpoints

### Week 2: Integration & Testing
- [ ] Integrate payment gateway
- [ ] Setup email notifications
- [ ] Implement file uploads
- [ ] Add error tracking
- [ ] Load testing
- [ ] Security audit

### Week 3: Launch Preparation
- [ ] Create user documentation
- [ ] Setup monitoring alerts
- [ ] Prepare customer support
- [ ] Marketing materials
- [ ] Beta testing
- [ ] Final security review

### Week 4: Launch! 🚀
- [ ] Soft launch to limited users
- [ ] Monitor metrics closely
- [ ] Gather feedback
- [ ] Fix critical bugs
- [ ] Public launch
- [ ] Celebrate! 🎉

---

## 📞 Support Resources

### Documentation
- Render: https://render.com/docs
- Vercel: https://vercel.com/docs
- Neon: https://neon.tech/docs
- Upstash: https://upstash.com/docs
- Cloudinary: https://cloudinary.com/documentation
- Resend: https://resend.com/docs
- Sentry: https://docs.sentry.io

### Community
- Render Community: https://community.render.com
- Vercel Discussions: https://github.com/vercel/vercel/discussions
- Prisma Discord: https://pris.ly/discord
- Node.js Slack: https://node-js.slack.com

---

**Document Status:** ✅ Production Ready  
**Last Updated:** March 26, 2026  
**Maintained By:** Platform Team

---

*This guide will be updated as services evolve and new free tiers become available.*
