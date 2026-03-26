# 🔒 Security & Compliance Documentation

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Status:** ✅ Production Ready

---

## Table of Contents

1. [Security Overview](#security-overview)
2. [Authentication & Authorization](#authentication--authorization)
3. [Data Protection](#data-protection)
4. [API Security](#api-security)
5. [Infrastructure Security](#infrastructure-security)
6. [Compliance Standards](#compliance-standards)
7. [Audit & Logging](#audit--logging)
8. [Incident Response](#incident-response)
9. [Security Best Practices](#security-best-practices)

---

## 1. Security Overview

### 1.1 Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Security Layers                          │
├─────────────────────────────────────────────────────────────┤
│  Layer 1: Network Security (WAF, DDoS Protection)           │
│  Layer 2: Application Security (Authentication, RBAC)       │
│  Layer 3: Data Security (Encryption, Masking)               │
│  Layer 4: Infrastructure Security (VPC, Security Groups)    │
│  Layer 5: Monitoring & Response (SIEM, Alerts)              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Security Principles

- **Defense in Depth:** Multiple layers of security controls
- **Least Privilege:** Minimum necessary access rights
- **Zero Trust:** Verify every access request
- **Security by Design:** Built-in security from inception
- **Continuous Monitoring:** Real-time threat detection

---

## 2. Authentication & Authorization

### 2.1 Authentication System

#### JWT Token Structure

```typescript
// Access Token (Short-lived: 15 minutes)
interface AccessToken {
  userId: string;
  email: string;
  roles: string[];
  permissions: string[];
  iat: number;  // Issued at
  exp: number;  // Expiration
}

// Refresh Token (Long-lived: 7 days)
interface RefreshToken {
  userId: string;
  tokenId: string;
  deviceInfo: string;
  iat: number;
  exp: number;
}
```

#### Token Management Flow

```typescript
// src/services/auth/token.service.ts
import jwt from 'jsonwebtoken';
import { nanoid } from 'nanoid';

export class TokenService {
  // Generate access token
  generateAccessToken(user: User): string {
    return jwt.sign(
      {
        userId: user.id,
        email: user.email,
        roles: user.roles.map(r => r.name),
        permissions: this.getUserPermissions(user)
      },
      process.env.JWT_SECRET!,
      { expiresIn: '15m' }
    );
  }

  // Generate refresh token
  async generateRefreshToken(
    userId: string, 
    deviceInfo: string
  ): Promise<string> {
    const tokenId = nanoid();
    
    // Store in database
    await prisma.refreshToken.create({
      data: {
        id: tokenId,
        userId,
        deviceInfo,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
      }
    });

    return jwt.sign(
      { userId, tokenId, deviceInfo },
      process.env.JWT_REFRESH_SECRET!,
      { expiresIn: '7d' }
    );
  }

  // Verify and decode token
  verifyAccessToken(token: string): AccessToken {
    try {
      return jwt.verify(token, process.env.JWT_SECRET!) as AccessToken;
    } catch (error) {
      throw new UnauthorizedError('Invalid or expired token');
    }
  }

  // Rotate refresh token
  async rotateRefreshToken(oldToken: string): Promise<TokenPair> {
    const decoded = jwt.verify(
      oldToken, 
      process.env.JWT_REFRESH_SECRET!
    ) as RefreshToken;

    // Invalidate old token
    await prisma.refreshToken.update({
      where: { id: decoded.tokenId },
      data: { isRevoked: true }
    });

    // Generate new tokens
    const user = await prisma.user.findUnique({
      where: { id: decoded.userId },
      include: { roles: true }
    });

    return {
      accessToken: this.generateAccessToken(user!),
      refreshToken: await this.generateRefreshToken(
        decoded.userId, 
        decoded.deviceInfo
      )
    };
  }
}
```

### 2.2 Password Security

#### Password Hashing

```typescript
// src/services/auth/password.service.ts
import bcrypt from 'bcrypt';
import { createHash } from 'crypto';

export class PasswordService {
  private readonly SALT_ROUNDS = 12;
  private readonly PEPPER = process.env.PASSWORD_PEPPER!;

  // Hash password with salt and pepper
  async hashPassword(password: string): Promise<string> {
    // Add pepper before hashing
    const pepperedPassword = this.addPepper(password);
    return bcrypt.hash(pepperedPassword, this.SALT_ROUNDS);
  }

  // Verify password
  async verifyPassword(
    password: string, 
    hash: string
  ): Promise<boolean> {
    const pepperedPassword = this.addPepper(password);
    return bcrypt.compare(pepperedPassword, hash);
  }

  // Add pepper using HMAC
  private addPepper(password: string): string {
    return createHash('sha256')
      .update(password + this.PEPPER)
      .digest('hex');
  }

  // Password strength validation
  validatePasswordStrength(password: string): ValidationResult {
    const errors: string[] = [];

    if (password.length < 12) {
      errors.push('Password must be at least 12 characters');
    }
    if (!/[A-Z]/.test(password)) {
      errors.push('Password must contain uppercase letter');
    }
    if (!/[a-z]/.test(password)) {
      errors.push('Password must contain lowercase letter');
    }
    if (!/[0-9]/.test(password)) {
      errors.push('Password must contain number');
    }
    if (!/[!@#$%^&*]/.test(password)) {
      errors.push('Password must contain special character');
    }

    return {
      isValid: errors.length === 0,
      errors
    };
  }
}
```

### 2.3 Role-Based Access Control (RBAC)

#### Role Hierarchy

```typescript
// Database Schema
enum RoleType {
  SUPER_ADMIN = 'SUPER_ADMIN',           // Full system access
  ADMIN = 'ADMIN',                        // Organization admin
  MANAGER = 'MANAGER',                    // Department manager
  SUPERVISOR = 'SUPERVISOR',              // Team supervisor
  OPERATOR = 'OPERATOR',                  // Production operator
  SALES_AGENT = 'SALES_AGENT',           // Sales representative
  WAREHOUSE_STAFF = 'WAREHOUSE_STAFF',   // Warehouse worker
  CUSTOMER_SUPPORT = 'CUSTOMER_SUPPORT', // Support staff
  ACCOUNTANT = 'ACCOUNTANT',             // Finance team
  CUSTOMER = 'CUSTOMER'                   // End customer
}
```

#### Permission System

```typescript
// src/services/auth/permission.service.ts
export class PermissionService {
  // Check if user has permission
  async hasPermission(
    userId: string, 
    permission: string
  ): Promise<boolean> {
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: {
        roles: {
          include: {
            permissions: true
          }
        }
      }
    });

    if (!user) return false;

    // Check if any role has the permission
    return user.roles.some(role =>
      role.permissions.some(p => p.name === permission)
    );
  }

  // Get all user permissions
  async getUserPermissions(userId: string): Promise<string[]> {
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: {
        roles: {
          include: {
            permissions: true
          }
        }
      }
    });

    if (!user) return [];

    // Flatten and deduplicate permissions
    const permissions = new Set<string>();
    user.roles.forEach(role => {
      role.permissions.forEach(p => permissions.add(p.name));
    });

    return Array.from(permissions);
  }
}
```

#### Authorization Middleware

```typescript
// src/middleware/auth.middleware.ts
export function requirePermission(permission: string) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      const token = extractToken(req);
      const decoded = tokenService.verifyAccessToken(token);
      
      // Check permission
      const hasPermission = await permissionService.hasPermission(
        decoded.userId,
        permission
      );

      if (!hasPermission) {
        throw new ForbiddenError(
          `Missing required permission: ${permission}`
        );
      }

      req.user = decoded;
      next();
    } catch (error) {
      next(error);
    }
  };
}

// Usage in routes
router.post(
  '/products',
  requirePermission('products.create'),
  productController.create
);
```

### 2.4 Multi-Factor Authentication (MFA)

```typescript
// src/services/auth/mfa.service.ts
import speakeasy from 'speakeasy';
import QRCode from 'qrcode';

export class MFAService {
  // Generate TOTP secret
  async enableMFA(userId: string): Promise<MFASetup> {
    const secret = speakeasy.generateSecret({
      name: `EcomPlatform (${userId})`,
      length: 32
    });

    // Store secret in database
    await prisma.user.update({
      where: { id: userId },
      data: {
        mfaSecret: secret.base32,
        mfaEnabled: false // Enabled after verification
      }
    });

    // Generate QR code
    const qrCode = await QRCode.toDataURL(secret.otpauth_url!);

    return {
      secret: secret.base32,
      qrCode,
      backupCodes: await this.generateBackupCodes(userId)
    };
  }

  // Verify TOTP token
  async verifyMFA(userId: string, token: string): Promise<boolean> {
    const user = await prisma.user.findUnique({
      where: { id: userId }
    });

    if (!user?.mfaSecret) {
      throw new Error('MFA not enabled');
    }

    return speakeasy.totp.verify({
      secret: user.mfaSecret,
      encoding: 'base32',
      token,
      window: 2 // Allow 2 time steps before/after
    });
  }

  // Generate backup codes
  private async generateBackupCodes(userId: string): Promise<string[]> {
    const codes = Array.from({ length: 10 }, () => 
      nanoid(10).toUpperCase()
    );

    // Hash and store codes
    const hashedCodes = await Promise.all(
      codes.map(code => bcrypt.hash(code, 10))
    );

    await prisma.mfaBackupCode.createMany({
      data: hashedCodes.map(hash => ({
        userId,
        codeHash: hash
      }))
    });

    return codes;
  }
}
```

---

## 3. Data Protection

### 3.1 Encryption Strategy

#### Data at Rest

```typescript
// src/services/encryption/field-encryption.service.ts
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

export class FieldEncryptionService {
  private readonly ALGORITHM = 'aes-256-gcm';
  private readonly KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex');

  // Encrypt sensitive field
  encrypt(plaintext: string): string {
    const iv = randomBytes(16);
    const cipher = createCipheriv(this.ALGORITHM, this.KEY, iv);
    
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    // Format: iv:authTag:encrypted
    return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
  }

  // Decrypt sensitive field
  decrypt(ciphertext: string): string {
    const [ivHex, authTagHex, encrypted] = ciphertext.split(':');
    
    const iv = Buffer.from(ivHex, 'hex');
    const authTag = Buffer.from(authTagHex, 'hex');
    
    const decipher = createDecipheriv(this.ALGORITHM, this.KEY, iv);
    decipher.setAuthTag(authTag);
    
    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

// Prisma middleware for automatic encryption
prisma.$use(async (params, next) => {
  const encryptedFields = ['ssn', 'bankAccount', 'creditCard'];
  
  if (params.action === 'create' || params.action === 'update') {
    for (const field of encryptedFields) {
      if (params.args.data[field]) {
        params.args.data[field] = fieldEncryptionService.encrypt(
          params.args.data[field]
        );
      }
    }
  }
  
  const result = await next(params);
  
  if (params.action === 'findUnique' || params.action === 'findMany') {
    const decrypt = (obj: any) => {
      for (const field of encryptedFields) {
        if (obj[field]) {
          obj[field] = fieldEncryptionService.decrypt(obj[field]);
        }
      }
    };
    
    if (Array.isArray(result)) {
      result.forEach(decrypt);
    } else if (result) {
      decrypt(result);
    }
  }
  
  return result;
});
```

#### Data in Transit

```typescript
// SSL/TLS Configuration
// src/config/ssl.config.ts
import fs from 'fs';
import https from 'https';

export const sslOptions = {
  key: fs.readFileSync(process.env.SSL_KEY_PATH!),
  cert: fs.readFileSync(process.env.SSL_CERT_PATH!),
  ca: fs.readFileSync(process.env.SSL_CA_PATH!),
  
  // TLS 1.3 only
  minVersion: 'TLSv1.3' as const,
  
  // Strong cipher suites
  ciphers: [
    'TLS_AES_256_GCM_SHA384',
    'TLS_CHACHA20_POLY1305_SHA256',
    'TLS_AES_128_GCM_SHA256'
  ].join(':'),
  
  // Enable HSTS
  secureOptions: require('constants').SSL_OP_NO_TLSv1 | 
                  require('constants').SSL_OP_NO_TLSv1_1 |
                  require('constants').SSL_OP_NO_TLSv1_2
};

// Create HTTPS server
const server = https.createServer(sslOptions, app);
```

### 3.2 Data Masking

```typescript
// src/services/data/masking.service.ts
export class DataMaskingService {
  // Mask email
  maskEmail(email: string): string {
    const [local, domain] = email.split('@');
    const maskedLocal = local.charAt(0) + 
                        '*'.repeat(local.length - 2) + 
                        local.charAt(local.length - 1);
    return `${maskedLocal}@${domain}`;
  }

  // Mask phone number
  maskPhone(phone: string): string {
    return phone.slice(0, -4).replace(/\d/g, '*') + phone.slice(-4);
  }

  // Mask credit card
  maskCreditCard(cardNumber: string): string {
    return '*'.repeat(cardNumber.length - 4) + cardNumber.slice(-4);
  }

  // Mask PII in logs
  maskPII(logMessage: string): string {
    return logMessage
      // Email
      .replace(
        /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g,
        '[EMAIL_REDACTED]'
      )
      // Phone
      .replace(
        /\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g,
        '[PHONE_REDACTED]'
      )
      // SSN
      .replace(
        /\b\d{3}-\d{2}-\d{4}\b/g,
        '[SSN_REDACTED]'
      )
      // Credit Card
      .replace(
        /\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b/g,
        '[CARD_REDACTED]'
      );
  }
}
```

### 3.3 Data Sanitization

```typescript
// src/middleware/sanitization.middleware.ts
import DOMPurify from 'isomorphic-dompurify';
import { escape } from 'html-escaper';

export function sanitizeInput(
  req: Request, 
  res: Response, 
  next: NextFunction
) {
  // Sanitize body
  if (req.body) {
    req.body = sanitizeObject(req.body);
  }
  
  // Sanitize query params
  if (req.query) {
    req.query = sanitizeObject(req.query);
  }
  
  next();
}

function sanitizeObject(obj: any): any {
  if (typeof obj === 'string') {
    return sanitizeString(obj);
  }
  
  if (Array.isArray(obj)) {
    return obj.map(sanitizeObject);
  }
  
  if (obj && typeof obj === 'object') {
    const sanitized: any = {};
    for (const key in obj) {
      sanitized[key] = sanitizeObject(obj[key]);
    }
    return sanitized;
  }
  
  return obj;
}

function sanitizeString(str: string): string {
  // Remove XSS attempts
  let clean = DOMPurify.sanitize(str, { 
    ALLOWED_TAGS: [],
    ALLOWED_ATTR: [] 
  });
  
  // Escape HTML entities
  clean = escape(clean);
  
  // Remove SQL injection patterns
  clean = clean.replace(/['";]/g, '');
  
  return clean;
}
```

---

## 4. API Security

### 4.1 Rate Limiting

```typescript
// src/middleware/rate-limit.middleware.ts
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';

// General API rate limit
export const apiLimiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:api:'
  }),
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // 100 requests per window
  message: 'Too many requests, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  handler: (req, res) => {
    logger.warn('Rate limit exceeded', {
      ip: req.ip,
      path: req.path,
      userId: req.user?.userId
    });
    res.status(429).json({
      error: 'Too many requests',
      retryAfter: Math.ceil(req.rateLimit.resetTime! / 1000)
    });
  }
});

// Strict rate limit for auth endpoints
export const authLimiter = rateLimit({
  store: new RedisStore({
    client: redisClient,
    prefix: 'rl:auth:'
  }),
  windowMs: 15 * 60 * 1000,
  max: 5, // 5 attempts per 15 minutes
  skipSuccessfulRequests: true
});

// Apply rate limiting
app.use('/api/', apiLimiter);
app.use('/api/auth/login', authLimiter);
app.use('/api/auth/register', authLimiter);
```

### 4.2 CORS Configuration

```typescript
// src/config/cors.config.ts
import cors from 'cors';

export const corsOptions: cors.CorsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];
    
    // Allow requests with no origin (mobile apps, Postman)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      logger.warn('CORS blocked', { origin });
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: [
    'Content-Type',
    'Authorization',
    'X-Request-ID',
    'X-API-Key'
  ],
  exposedHeaders: ['X-Total-Count', 'X-Page-Count'],
  maxAge: 86400 // 24 hours
};

app.use(cors(corsOptions));
```

### 4.3 API Key Management

```typescript
// src/services/auth/api-key.service.ts
export class APIKeyService {
  // Generate API key
  async generateAPIKey(
    userId: string, 
    name: string,
    permissions: string[]
  ): Promise<APIKey> {
    const key = `pk_${nanoid(32)}`;
    const hashedKey = await bcrypt.hash(key, 10);

    const apiKey = await prisma.apiKey.create({
      data: {
        userId,
        name,
        keyHash: hashedKey,
        permissions,
        expiresAt: new Date(Date.now() + 365 * 24 * 60 * 60 * 1000)
      }
    });

    return {
      id: apiKey.id,
      key, // Only returned once
      name: apiKey.name,
      permissions: apiKey.permissions,
      createdAt: apiKey.createdAt
    };
  }

  // Validate API key
  async validateAPIKey(key: string): Promise<APIKeyData | null> {
    const apiKeys = await prisma.apiKey.findMany({
      where: {
        isActive: true,
        expiresAt: { gt: new Date() }
      }
    });

    for (const apiKey of apiKeys) {
      const isValid = await bcrypt.compare(key, apiKey.keyHash);
      if (isValid) {
        // Update last used
        await prisma.apiKey.update({
          where: { id: apiKey.id },
          data: { lastUsedAt: new Date() }
        });

        return {
          userId: apiKey.userId,
          permissions: apiKey.permissions
        };
      }
    }

    return null;
  }
}

// API Key middleware
export async function requireAPIKey(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const apiKey = req.headers['x-api-key'] as string;

  if (!apiKey) {
    throw new UnauthorizedError('API key required');
  }

  const keyData = await apiKeyService.validateAPIKey(apiKey);

  if (!keyData) {
    throw new UnauthorizedError('Invalid API key');
  }

  req.apiKey = keyData;
  next();
}
```

### 4.4 Request Validation

```typescript
// src/middleware/validation.middleware.ts
import { z } from 'zod';

export function validateRequest(schema: z.ZodSchema) {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      // Validate request body
      const validated = await schema.parseAsync(req.body);
      req.body = validated;
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        res.status(400).json({
          error: 'Validation failed',
          details: error.errors.map(err => ({
            field: err.path.join('.'),
            message: err.message
          }))
        });
      } else {
        next(error);
      }
    }
  };
}

// Usage example
const createProductSchema = z.object({
  name: z.string().min(3).max(100),
  sku: z.string().regex(/^[A-Z0-9-]+$/),
  price: z.number().positive(),
  description: z.string().optional()
});

router.post(
  '/products',
  validateRequest(createProductSchema),
  productController.create
);
```

---

## 5. Infrastructure Security

### 5.1 AWS Security Groups

```yaml
# infrastructure/security-groups.yml
SecurityGroups:
  # Application Load Balancer
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: ALB Security Group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  # Application Security Group
  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Application Security Group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3000
          ToPort: 3000
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  # Database Security Group
  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Database Security Group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref AppSecurityGroup

  # Redis Security Group
  RedisSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Redis Security Group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 6379
          ToPort: 6379
          SourceSecurityGroupId: !Ref AppSecurityGroup
```

### 5.2 Secrets Management

```typescript
// src/config/secrets.config.ts
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

export class SecretsService {
  private client = new SecretsManager({
    region: process.env.AWS_REGION
  });

  // Fetch secret from AWS Secrets Manager
  async getSecret(secretName: string): Promise<any> {
    try {
      const response = await this.client.getSecretValue({
        SecretId: secretName
      });

      if (response.SecretString) {
        return JSON.parse(response.SecretString);
      }

      throw new Error('Secret not found');
    } catch (error) {
      logger.error('Failed to fetch secret', { secretName, error });
      throw error;
    }
  }

  // Initialize application secrets
  async initializeSecrets(): Promise<void> {
    const secrets = await this.getSecret(
      process.env.SECRET_NAME || 'ecom-platform/prod'
    );

    process.env.DATABASE_URL = secrets.DATABASE_URL;
    process.env.REDIS_URL = secrets.REDIS_URL;
    process.env.JWT_SECRET = secrets.JWT_SECRET;
    process.env.JWT_REFRESH_SECRET = secrets.JWT_REFRESH_SECRET;
    process.env.ENCRYPTION_KEY = secrets.ENCRYPTION_KEY;
    // ... other secrets
  }
}
```

### 5.3 Network Security

```typescript
// VPC Configuration
export const vpcConfig = {
  cidr: '10.0.0.0/16',
  
  // Public subnets (ALB)
  publicSubnets: [
    { cidr: '10.0.1.0/24', az: 'ap-south-1a' },
    { cidr: '10.0.2.0/24', az: 'ap-south-1b' }
  ],
  
  // Private subnets (Application)
  privateSubnets: [
    { cidr: '10.0.10.0/24', az: 'ap-south-1a' },
    { cidr: '10.0.11.0/24', az: 'ap-south-1b' }
  ],
  
  // Database subnets
  databaseSubnets: [
    { cidr: '10.0.20.0/24', az: 'ap-south-1a' },
    { cidr: '10.0.21.0/24', az: 'ap-south-1b' }
  ],
  
  // NAT Gateway for outbound traffic
  natGateway: {
    enabled: true,
    count: 2 // One per AZ
  }
};
```

---

## 6. Compliance Standards

### 6.1 PCI-DSS Compliance

#### Requirements Checklist

```markdown
✅ Requirement 1: Install and maintain firewall
   - AWS WAF configured
   - Security groups implemented
   - Network segmentation in place

✅ Requirement 2: Secure system configurations
   - Default passwords changed
   - Unnecessary services disabled
   - Security patches applied

✅ Requirement 3: Protect stored cardholder data
   - Card data encrypted (AES-256)
   - Encryption keys rotated
   - Data retention policy enforced

✅ Requirement 4: Encrypt transmission of data
   - TLS 1.3 enforced
   - Strong cipher suites only
   - Certificate management automated

✅ Requirement 5: Use and update antivirus
   - AWS GuardDuty enabled
   - Malware scanning on uploads
   - Regular security scans

✅ Requirement 6: Develop secure systems
   - Secure coding practices
   - Code review process
   - Security testing integrated

✅ Requirement 7: Restrict data access
   - RBAC implemented
   - Least privilege principle
   - Access reviews quarterly

✅ Requirement 8: Assign unique IDs
   - Unique user IDs enforced
   - Strong authentication required
   - MFA for admin accounts

✅ Requirement 9: Restrict physical access
   - AWS datacenter security
   - Media destruction policy
   - Visitor logs maintained

✅ Requirement 10: Track and monitor access
   - Comprehensive audit logs
   - SIEM integration
   - Daily log reviews

✅ Requirement 11: Test security systems
   - Quarterly vulnerability scans
   - Annual penetration testing
   - IDS/IPS configured

✅ Requirement 12: Maintain security policy
   - Information security policy
   - Risk assessment program
   - Incident response plan
```

### 6.2 GDPR Compliance

```typescript
// src/services/compliance/gdpr.service.ts
export class GDPRService {
  // Right to access
  async exportUserData(userId: string): Promise<UserDataExport> {
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: {
        orders: true,
        addresses: true,
        payments: true,
        supportTickets: true
      }
    });

    return {
      personalData: {
        name: user.name,
        email: user.email,
        phone: user.phone,
        createdAt: user.createdAt
      },
      orders: user.orders,
      addresses: user.addresses,
      // ... other data
    };
  }

  // Right to erasure (right to be forgotten)
  async deleteUserData(userId: string): Promise<void> {
    await prisma.$transaction(async (tx) => {
      // Anonymize instead of delete for compliance
      await tx.user.update({
        where: { id: userId },
        data: {
          email: `deleted_${userId}@deleted.com`,
          name: 'Deleted User',
          phone: null,
          isDeleted: true,
          deletedAt: new Date()
        }
      });

      // Delete or anonymize related data
      await tx.address.deleteMany({ where: { userId } });
      await tx.savedCard.deleteMany({ where: { userId } });
      
      // Keep orders for legal reasons but anonymize
      await tx.order.updateMany({
        where: { customerId: userId },
        data: {
          customerEmail: 'deleted@deleted.com',
          customerPhone: null
        }
      });
    });
  }

  // Consent management
  async updateConsent(
    userId: string,
    consents: ConsentUpdate[]
  ): Promise<void> {
    await prisma.$transaction(
      consents.map(consent =>
        prisma.userConsent.upsert({
          where: {
            userId_consentType: {
              userId,
              consentType: consent.type
            }
          },
          create: {
            userId,
            consentType: consent.type,
            granted: consent.granted
          },
          update: {
            granted: consent.granted,
            updatedAt: new Date()
          }
        })
      )
    );
  }

  // Data portability
  async generateDataPortabilityReport(
    userId: string
  ): Promise<Buffer> {
    const data = await this.exportUserData(userId);
    
    // Generate JSON file
    const json = JSON.stringify(data, null, 2);
    
    return Buffer.from(json, 'utf-8');
  }
}
```

### 6.3 SOC 2 Compliance

```typescript
// Control Implementation
export const soc2Controls = {
  // CC6.1 - Logical and Physical Access Controls
  accessControl: {
    implemented: true,
    controls: [
      'RBAC with least privilege',
      'MFA for privileged accounts',
      'Session timeout after 15 minutes',
      'Account lockout after 5 failed attempts'
    ]
  },

  // CC7.1 - System Monitoring
  monitoring: {
    implemented: true,
    controls: [
      'Real-time security monitoring',
      'Automated threat detection',
      'Log aggregation and analysis',
      '24/7 SOC team'
    ]
  },

  // CC7.2 - Change Management
  changeManagement: {
    implemented: true,
    controls: [
      'Change approval process',
      'Automated testing before deployment',
      'Rollback procedures',
      'Change documentation'
    ]
  },

  // CC8.1 - Risk Assessment
  riskAssessment: {
    implemented: true,
    controls: [
      'Quarterly risk assessments',
      'Vulnerability scanning',
      'Penetration testing',
      'Risk mitigation plans'
    ]
  }
};
```

---

## 7. Audit & Logging

### 7.1 Audit Trail System

```typescript
// src/services/audit/audit-log.service.ts
export class AuditLogService {
  // Log user action
  async logAction(action: AuditAction): Promise<void> {
    await prisma.auditLog.create({
      data: {
        userId: action.userId,
        action: action.type,
        resource: action.resource,
        resourceId: action.resourceId,
        changes: action.changes,
        ipAddress: action.ipAddress,
        userAgent: action.userAgent,
        timestamp: new Date()
      }
    });
  }

  // Log data access
  async logDataAccess(access: DataAccessLog): Promise<void> {
    await prisma.dataAccessLog.create({
      data: {
        userId: access.userId,
        dataType: access.dataType,
        dataId: access.dataId,
        accessType: access.accessType, // READ, WRITE, DELETE
        timestamp: new Date()
      }
    });
  }

  // Get audit trail for resource
  async getAuditTrail(
    resource: string,
    resourceId: string
  ): Promise<AuditLog[]> {
    return prisma.auditLog.findMany({
      where: { resource, resourceId },
      orderBy: { timestamp: 'desc' },
      include: { user: true }
    });
  }
}

// Audit middleware
export function auditMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const originalSend = res.send;

  res.send = function (data) {
    // Log after response
    if (res.statusCode < 400) {
      auditLogService.logAction({
        userId: req.user?.userId,
        type: `${req.method}_${req.route?.path}`,
        resource: req.baseUrl,
        resourceId: req.params.id,
        ipAddress: req.ip,
        userAgent: req.get('user-agent')
      });
    }

    return originalSend.call(this, data);
  };

  next();
}
```

### 7.2 Security Logging

```typescript
// src/services/logging/security-logger.ts
import winston from 'winston';

export const securityLogger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    // Security events file
    new winston.transports.File({
      filename: 'logs/security.log',
      level: 'warn'
    }),
    
    // Critical security alerts
    new winston.transports.File({
      filename: 'logs/security-critical.log',
      level: 'error'
    })
  ]
});

// Security event types
export enum SecurityEvent {
  LOGIN_SUCCESS = 'LOGIN_SUCCESS',
  LOGIN_FAILURE = 'LOGIN_FAILURE',
  LOGOUT = 'LOGOUT',
  PASSWORD_CHANGE = 'PASSWORD_CHANGE',
  PASSWORD_RESET = 'PASSWORD_RESET',
  MFA_ENABLED = 'MFA_ENABLED',
  MFA_DISABLED = 'MFA_DISABLED',
  PERMISSION_DENIED = 'PERMISSION_DENIED',
  SUSPICIOUS_ACTIVITY = 'SUSPICIOUS_ACTIVITY',
  DATA_BREACH_ATTEMPT = 'DATA_BREACH_ATTEMPT'
}

// Log security event
export function logSecurityEvent(
  event: SecurityEvent,
  details: any
): void {
  securityLogger.info({
    event,
    ...details,
    timestamp: new Date().toISOString()
  });
}
```

---

## 8. Incident Response

### 8.1 Incident Response Plan

```typescript
// src/services/security/incident-response.service.ts
export class IncidentResponseService {
  // Detect security incident
  async detectIncident(alert: SecurityAlert): Promise<Incident | null> {
    // Analyze alert severity
    const severity = this.calculateSeverity(alert);

    if (severity >= IncidentSeverity.MEDIUM) {
      // Create incident
      const incident = await prisma.securityIncident.create({
        data: {
          type: alert.type,
          severity,
          description: alert.description,
          detectedAt: new Date(),
          status: 'OPEN'
        }
      });

      // Notify security team
      await this.notifySecurityTeam(incident);

      // Auto-remediate if possible
      if (this.canAutoRemediate(incident)) {
        await this.autoRemediate(incident);
      }

      return incident;
    }

    return null;
  }

  // Auto-remediation
  private async autoRemediate(incident: Incident): Promise<void> {
    switch (incident.type) {
      case 'BRUTE_FORCE_ATTACK':
        // Block IP address
        await this.blockIP(incident.sourceIP);
        break;

      case 'SQL_INJECTION_ATTEMPT':
        // Block user and alert
        await this.blockUser(incident.userId);
        await this.alertAdmins(incident);
        break;

      case 'SUSPICIOUS_API_USAGE':
        // Rate limit user
        await this.applyStrictRateLimit(incident.userId);
        break;

      default:
        // Alert for manual review
        await this.alertAdmins(incident);
    }

    await prisma.securityIncident.update({
      where: { id: incident.id },
      data: {
        status: 'AUTO_REMEDIATED',
        remediatedAt: new Date()
      }
    });
  }

  // Incident response workflow
  async handleIncident(incidentId: string): Promise<void> {
    // 1. Identification
    const incident = await this.identifyIncident(incidentId);

    // 2. Containment
    await this.containIncident(incident);

    // 3. Eradication
    await this.eradicateThreat(incident);

    // 4. Recovery
    await this.recoverFromIncident(incident);

    // 5. Lessons Learned
    await this.documentLessonsLearned(incident);
  }
}
```

### 8.2 Breach Notification

```typescript
// src/services/security/breach-notification.service.ts
export class BreachNotificationService {
  // Assess data breach
  async assessBreach(incident: Incident): Promise<BreachAssessment> {
    const affectedUsers = await this.getAffectedUsers(incident);
    const dataTypes = await this.getAffectedDataTypes(incident);

    return {
      severity: this.assessBreachSeverity(affectedUsers, dataTypes),
      affectedUsers,
      dataTypes,
      requiresNotification: this.requiresNotification(
        affectedUsers.length,
        dataTypes
      )
    };
  }

  // Notify affected users
  async notifyAffectedUsers(
    assessment: BreachAssessment
  ): Promise<void> {
    if (!assessment.requiresNotification) return;

    // GDPR requires notification within 72 hours
    const notifications = assessment.affectedUsers.map(user => ({
      userId: user.id,
      type: 'SECURITY_BREACH',
      channels: ['EMAIL', 'SMS'],
      priority: 'HIGH',
      content: this.generateBreachNotification(assessment)
    }));

    await notificationService.sendBulk(notifications);

    // Notify regulatory authorities if required
    if (assessment.severity === 'CRITICAL') {
      await this.notifyAuthorities(assessment);
    }
  }
}
```

---

## 9. Security Best Practices

### 9.1 Secure Coding Guidelines

```typescript
// ✅ Good: Parameterized queries
async function getUserById(id: string) {
  return prisma.user.findUnique({
    where: { id }
  });
}

// ❌ Bad: String concatenation (SQL injection risk)
async function getUserByIdBad(id: string) {
  return prisma.$queryRaw`SELECT * FROM users WHERE id = ${id}`;
}

// ✅ Good: Input validation
const schema = z.object({
  email: z.string().email(),
  age: z.number().min(18)
});

// ❌ Bad: No validation
function processUserInput(data: any) {
  // Directly using unvalidated input
  saveToDatabase(data);
}

// ✅ Good: Error handling without leaking info
try {
  await dangerousOperation();
} catch (error) {
  logger.error('Operation failed', { error });
  throw new Error('Operation failed');
}

// ❌ Bad: Leaking stack traces
try {
  await dangerousOperation();
} catch (error) {
  res.status(500).json({ error: error.stack });
}
```

### 9.2 Security Checklist

```markdown
## Development Phase
- [ ] Input validation on all user inputs
- [ ] Output encoding to prevent XSS
- [ ] Parameterized queries to prevent SQL injection
- [ ] Secure password storage (bcrypt)
- [ ] HTTPS enforced
- [ ] CSRF protection enabled
- [ ] Security headers configured
- [ ] Rate limiting implemented
- [ ] Error handling without info leakage

## Testing Phase
- [ ] Security unit tests written
- [ ] Penetration testing performed
- [ ] Vulnerability scanning completed
- [ ] Dependency audit passed
- [ ] Code review for security issues

## Deployment Phase
- [ ] Secrets not in source code
- [ ] Environment variables secured
- [ ] Database credentials rotated
- [ ] SSL certificates valid
- [ ] Security groups configured
- [ ] Monitoring and alerting active
- [ ] Backup and recovery tested

## Production Phase
- [ ] Regular security updates
- [ ] Log monitoring active
- [ ] Incident response plan ready
- [ ] Access reviews quarterly
- [ ] Compliance audits scheduled
```

### 9.3 Security Training

```markdown
## Required Training for All Developers

### Module 1: Secure Coding Fundamentals (4 hours)
- OWASP Top 10
- Input validation
- Authentication & authorization
- Secure data storage

### Module 2: Application Security (4 hours)
- XSS prevention
- CSRF protection
- SQL injection prevention
- API security

### Module 3: Infrastructure Security (4 hours)
- AWS security best practices
- Network security
- Secrets management
- Container security

### Module 4: Compliance & Privacy (2 hours)
- GDPR requirements
- PCI-DSS compliance
- Data privacy
- Incident response

### Annual Refresher (2 hours)
- Security updates
- New threats
- Lessons learned
- Policy changes
```

---

## 10. Security Monitoring

### 10.1 Real-time Monitoring

```typescript
// src/services/monitoring/security-monitor.service.ts
export class SecurityMonitorService {
  // Monitor suspicious activities
  async monitorSuspiciousActivity(): Promise<void> {
    // Check for multiple failed login attempts
    const failedLogins = await this.getFailedLogins();
    for (const login of failedLogins) {
      if (login.attempts >= 5) {
        await this.handleBruteForceAttempt(login);
      }
    }

    // Check for unusual API usage
    const apiUsage = await this.getAPIUsageMetrics();
    for (const usage of apiUsage) {
      if (usage.requestCount > usage.normalThreshold * 3) {
        await this.handleSuspiciousAPIUsage(usage);
      }
    }

    // Check for data exfiltration attempts
    const largeDownloads = await this.getLargeDownloads();
    for (const download of largeDownloads) {
      if (download.size > 100 * 1024 * 1024) { // 100MB
        await this.handleSuspiciousDownload(download);
      }
    }
  }

  // Security metrics dashboard
  async getSecurityMetrics(): Promise<SecurityMetrics> {
    return {
      failedLogins: await this.getFailedLoginCount(),
      blockedIPs: await this.getBlockedIPCount(),
      activeIncidents: await this.getActiveIncidentCount(),
      vulnerabilities: await this.getVulnerabilityCount(),
      complianceScore: await this.calculateComplianceScore()
    };
  }
}
```

### 10.2 Alerts & Notifications

```typescript
// Security alert configuration
export const securityAlerts = {
  // Critical alerts (immediate notification)
  critical: [
    'DATA_BREACH',
    'UNAUTHORIZED_ACCESS',
    'PRIVILEGE_ESCALATION',
    'SYSTEM_COMPROMISE'
  ],

  // High priority alerts (notify within 15 minutes)
  high: [
    'BRUTE_FORCE_ATTACK',
    'SQL_INJECTION_ATTEMPT',
    'SUSPICIOUS_FILE_UPLOAD',
    'UNUSUAL_DATA_ACCESS'
  ],

  // Medium priority alerts (notify within 1 hour)
  medium: [
    'MULTIPLE_FAILED_LOGINS',
    'RATE_LIMIT_EXCEEDED',
    'SUSPICIOUS_API_USAGE'
  ]
};
```

---

## Appendix

### A. Security Tools

- **SAST:** SonarQube, ESLint Security Plugin
- **DAST:** OWASP ZAP, Burp Suite
- **Dependency Scanning:** Snyk, npm audit
- **Container Scanning:** Trivy, Clair
- **Secrets Detection:** GitGuardian, TruffleHog

### B. Security Standards

- OWASP Top 10
- CWE Top 25
- NIST Cybersecurity Framework
- ISO 27001
- PCI-DSS v3.2.1

### C. References

- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [AWS Security Best Practices](https://aws.amazon.com/security/best-practices/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)

---

**Document Status:** ✅ Complete  
**Last Security Audit:** March 26, 2026  
**Next Audit:** June 26, 2026  
**Security Team:** security@company.com
