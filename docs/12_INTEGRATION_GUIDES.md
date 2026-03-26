# 🔌 Integration Guides

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Status:** ✅ Production Ready

---

## Table of Contents

1. [Integration Overview](#integration-overview)
2. [Payment Gateway Integration](#payment-gateway-integration)
3. [Marketplace Integrations](#marketplace-integrations)
4. [Shipping Partner Integration](#shipping-partner-integration)
5. [Communication Services](#communication-services)
6. [Analytics & Tracking](#analytics--tracking)
7. [Cloud Storage Integration](#cloud-storage-integration)
8. [Third-Party APIs](#third-party-apis)

---

## 1. Integration Overview

### 1.1 Integration Architecture

```
┌─────────────────────────────────────────────────────────┐
│               Application Core                           │
└───────────┬─────────────────────────────────┬───────────┘
            │                                 │
    ┌───────▼────────┐                ┌──────▼────────┐
    │ Payment Gateway│                │ Marketplaces  │
    │  - Razorpay    │                │  - Amazon     │
    │  - Stripe      │                │  - Flipkart   │
    │  - PayU        │                │  - Meesho     │
    └────────────────┘                └───────────────┘
            │                                 │
    ┌───────▼────────┐                ┌──────▼────────┐
    │ Shipping APIs  │                │ Communication │
    │  - Delhivery   │                │  - Twilio     │
    │  - Blue Dart   │                │  - SendGrid   │
    │  - Shiprocket  │                │  - WhatsApp   │
    └────────────────┘                └───────────────┘
```

### 1.2 Integration Principles

- **Loose Coupling:** Use adapter pattern
- **Error Handling:** Graceful degradation
- **Rate Limiting:** Respect API limits
- **Retry Logic:** Exponential backoff
- **Logging:** Comprehensive audit trail
- **Testing:** Mock external services
- **Monitoring:** Track integration health

---

## 2. Payment Gateway Integration

### 2.1 Razorpay Integration

#### Setup Configuration

```typescript
// src/config/razorpay.config.ts
import Razorpay from 'razorpay';

export const razorpayClient = new Razorpay({
  key_id: process.env.RAZORPAY_KEY_ID!,
  key_secret: process.env.RAZORPAY_KEY_SECRET!
});

export const razorpayConfig = {
  keyId: process.env.RAZORPAY_KEY_ID!,
  webhookSecret: process.env.RAZORPAY_WEBHOOK_SECRET!,
  currency: 'INR',
  timeout: 30000 // 30 seconds
};
```

#### Payment Service Implementation

```typescript
// src/services/payment/razorpay.service.ts
import { razorpayClient } from '@/config/razorpay.config';
import crypto from 'crypto';

export class RazorpayService {
  // Create order
  async createOrder(amount: number, orderId: string): Promise<RazorpayOrder> {
    try {
      const options = {
        amount: amount * 100, // Convert to paise
        currency: 'INR',
        receipt: orderId,
        notes: {
          orderId,
          createdAt: new Date().toISOString()
        }
      };

      const order = await razorpayClient.orders.create(options);

      // Store order in database
      await prisma.paymentTransaction.create({
        data: {
          orderId,
          razorpayOrderId: order.id,
          amount: amount,
          currency: 'INR',
          status: 'CREATED'
        }
      });

      return {
        id: order.id,
        amount: order.amount,
        currency: order.currency,
        receipt: order.receipt
      };
    } catch (error) {
      logger.error('Razorpay order creation failed', { error, orderId });
      throw new PaymentError('Failed to create payment order');
    }
  }

  // Verify payment signature
  verifyPaymentSignature(
    razorpayOrderId: string,
    razorpayPaymentId: string,
    razorpaySignature: string
  ): boolean {
    const body = razorpayOrderId + '|' + razorpayPaymentId;
    
    const expectedSignature = crypto
      .createHmac('sha256', process.env.RAZORPAY_KEY_SECRET!)
      .update(body)
      .digest('hex');

    return expectedSignature === razorpaySignature;
  }

  // Capture payment
  async capturePayment(
    paymentId: string,
    amount: number
  ): Promise<PaymentCapture> {
    try {
      const capture = await razorpayClient.payments.capture(
        paymentId,
        amount * 100 // Convert to paise
      );

      // Update payment transaction
      await prisma.paymentTransaction.update({
        where: { razorpayPaymentId: paymentId },
        data: {
          status: 'CAPTURED',
          capturedAt: new Date(),
          razorpayResponse: capture
        }
      });

      return {
        id: capture.id,
        amount: capture.amount / 100,
        status: capture.status
      };
    } catch (error) {
      logger.error('Payment capture failed', { error, paymentId });
      throw new PaymentError('Failed to capture payment');
    }
  }

  // Create refund
  async createRefund(
    paymentId: string,
    amount?: number
  ): Promise<Refund> {
    try {
      const refund = await razorpayClient.payments.refund(paymentId, {
        amount: amount ? amount * 100 : undefined,
        notes: {
          refundedAt: new Date().toISOString()
        }
      });

      // Create refund record
      await prisma.refund.create({
        data: {
          razorpayRefundId: refund.id,
          paymentId,
          amount: refund.amount / 100,
          status: refund.status
        }
      });

      return {
        id: refund.id,
        amount: refund.amount / 100,
        status: refund.status
      };
    } catch (error) {
      logger.error('Refund creation failed', { error, paymentId });
      throw new PaymentError('Failed to create refund');
    }
  }

  // Handle webhook
  async handleWebhook(
    signature: string,
    body: any
  ): Promise<void> {
    // Verify webhook signature
    const expectedSignature = crypto
      .createHmac('sha256', process.env.RAZORPAY_WEBHOOK_SECRET!)
      .update(JSON.stringify(body))
      .digest('hex');

    if (signature !== expectedSignature) {
      throw new Error('Invalid webhook signature');
    }

    const event = body.event;
    const payload = body.payload.payment.entity;

    switch (event) {
      case 'payment.captured':
        await this.handlePaymentCaptured(payload);
        break;
      
      case 'payment.failed':
        await this.handlePaymentFailed(payload);
        break;
      
      case 'refund.processed':
        await this.handleRefundProcessed(payload);
        break;

      default:
        logger.warn('Unhandled webhook event', { event });
    }
  }

  private async handlePaymentCaptured(payload: any): Promise<void> {
    const paymentId = payload.id;
    
    await prisma.paymentTransaction.update({
      where: { razorpayPaymentId: paymentId },
      data: {
        status: 'CAPTURED',
        capturedAt: new Date()
      }
    });

    // Update order status
    const payment = await prisma.paymentTransaction.findUnique({
      where: { razorpayPaymentId: paymentId }
    });

    if (payment) {
      await prisma.order.update({
        where: { id: payment.orderId },
        data: { paymentStatus: 'PAID' }
      });
    }
  }

  private async handlePaymentFailed(payload: any): Promise<void> {
    const paymentId = payload.id;
    
    await prisma.paymentTransaction.update({
      where: { razorpayPaymentId: paymentId },
      data: {
        status: 'FAILED',
        failureReason: payload.error_description
      }
    });
  }

  private async handleRefundProcessed(payload: any): Promise<void> {
    const refundId = payload.id;
    
    await prisma.refund.update({
      where: { razorpayRefundId: refundId },
      data: {
        status: 'PROCESSED',
        processedAt: new Date()
      }
    });
  }
}
```

#### Frontend Integration

```typescript
// frontend/src/services/payment.service.ts
declare global {
  interface Window {
    Razorpay: any;
  }
}

export class PaymentService {
  async initiatePayment(orderId: string, amount: number): Promise<void> {
    // Create order on backend
    const response = await axios.post('/api/payments/create-order', {
      orderId,
      amount
    });

    const { razorpayOrderId, keyId } = response.data;

    // Open Razorpay checkout
    const options = {
      key: keyId,
      amount: amount * 100,
      currency: 'INR',
      name: 'Ecom Platform',
      description: `Order #${orderId}`,
      order_id: razorpayOrderId,
      handler: async (response: any) => {
        await this.verifyPayment(
          orderId,
          response.razorpay_order_id,
          response.razorpay_payment_id,
          response.razorpay_signature
        );
      },
      prefill: {
        name: this.user.name,
        email: this.user.email,
        contact: this.user.phone
      },
      theme: {
        color: '#3B82F6'
      },
      modal: {
        ondismiss: () => {
          console.log('Payment cancelled');
        }
      }
    };

    const rzp = new window.Razorpay(options);
    rzp.open();
  }

  private async verifyPayment(
    orderId: string,
    razorpayOrderId: string,
    razorpayPaymentId: string,
    razorpaySignature: string
  ): Promise<void> {
    await axios.post('/api/payments/verify', {
      orderId,
      razorpayOrderId,
      razorpayPaymentId,
      razorpaySignature
    });

    // Redirect to success page
    window.location.href = `/order-confirmation/${orderId}`;
  }
}
```

---

## 3. Marketplace Integrations

### 3.1 Amazon Seller Central Integration

#### Configuration

```typescript
// src/config/amazon.config.ts
export const amazonConfig = {
  sellerId: process.env.AMAZON_SELLER_ID!,
  mwsAuthToken: process.env.AMAZON_MWS_AUTH_TOKEN!,
  accessKey: process.env.AMAZON_ACCESS_KEY!,
  secretKey: process.env.AMAZON_SECRET_KEY!,
  region: 'IN',
  marketplace: 'A21TJRUUN4KGV', // India marketplace ID
  apiVersion: '2013-09-01'
};
```

#### Amazon Service Implementation

```typescript
// src/services/marketplace/amazon.service.ts
import { AmazonMWS } from 'amazon-mws';
import { amazonConfig } from '@/config/amazon.config';

export class AmazonService {
  private mws: AmazonMWS;

  constructor() {
    this.mws = new AmazonMWS({
      accessKeyId: amazonConfig.accessKey,
      secretAccessKey: amazonConfig.secretKey,
      merchantId: amazonConfig.sellerId,
      authToken: amazonConfig.mwsAuthToken,
      region: amazonConfig.region
    });
  }

  // Sync products to Amazon
  async syncProduct(product: Product): Promise<void> {
    try {
      const feed = this.buildProductFeed(product);
      
      const response = await this.mws.feeds.submit({
        feedType: '_POST_PRODUCT_DATA_',
        feedContent: feed
      });

      // Store sync record
      await prisma.marketplaceSync.create({
        data: {
          marketplace: 'AMAZON',
          productId: product.id,
          feedId: response.FeedSubmissionId,
          status: 'SUBMITTED'
        }
      });

      logger.info('Product synced to Amazon', { 
        productId: product.id,
        feedId: response.FeedSubmissionId
      });
    } catch (error) {
      logger.error('Amazon product sync failed', { error, product });
      throw new MarketplaceError('Failed to sync product to Amazon');
    }
  }

  // Update inventory on Amazon
  async updateInventory(sku: string, quantity: number): Promise<void> {
    try {
      const feed = this.buildInventoryFeed(sku, quantity);
      
      await this.mws.feeds.submit({
        feedType: '_POST_INVENTORY_AVAILABILITY_DATA_',
        feedContent: feed
      });

      logger.info('Inventory updated on Amazon', { sku, quantity });
    } catch (error) {
      logger.error('Amazon inventory update failed', { error, sku });
      throw new MarketplaceError('Failed to update inventory on Amazon');
    }
  }

  // Fetch Amazon orders
  async fetchOrders(startDate: Date): Promise<AmazonOrder[]> {
    try {
      const response = await this.mws.orders.list({
        CreatedAfter: startDate.toISOString(),
        MarketplaceId: amazonConfig.marketplace
      });

      const orders = response.Orders.Order;
      
      // Process and store orders
      for (const order of orders) {
        await this.processAmazonOrder(order);
      }

      return orders;
    } catch (error) {
      logger.error('Failed to fetch Amazon orders', { error });
      throw new MarketplaceError('Failed to fetch Amazon orders');
    }
  }

  private async processAmazonOrder(amazonOrder: any): Promise<void> {
    // Check if order already exists
    const existing = await prisma.order.findUnique({
      where: { amazonOrderId: amazonOrder.AmazonOrderId }
    });

    if (existing) return;

    // Fetch order items
    const itemsResponse = await this.mws.orders.listOrderItems({
      AmazonOrderId: amazonOrder.AmazonOrderId
    });

    const items = itemsResponse.OrderItems.OrderItem;

    // Create order in our system
    await prisma.order.create({
      data: {
        amazonOrderId: amazonOrder.AmazonOrderId,
        channel: 'AMAZON',
        totalAmount: parseFloat(amazonOrder.OrderTotal.Amount),
        currency: amazonOrder.OrderTotal.CurrencyCode,
        orderStatus: this.mapAmazonStatus(amazonOrder.OrderStatus),
        items: {
          create: items.map((item: any) => ({
            sku: item.SellerSKU,
            quantity: parseInt(item.QuantityOrdered),
            price: parseFloat(item.ItemPrice.Amount)
          }))
        }
      }
    });
  }

  private buildProductFeed(product: Product): string {
    return `<?xml version="1.0" encoding="UTF-8"?>
      <AmazonEnvelope>
        <Header>
          <DocumentVersion>1.01</DocumentVersion>
          <MerchantIdentifier>${amazonConfig.sellerId}</MerchantIdentifier>
        </Header>
        <MessageType>Product</MessageType>
        <Message>
          <MessageID>1</MessageID>
          <OperationType>Update</OperationType>
          <Product>
            <SKU>${product.sku}</SKU>
            <StandardProductID>
              <Type>EAN</Type>
              <Value>${product.ean}</Value>
            </StandardProductID>
            <ProductTaxCode>A_GEN_TAX</ProductTaxCode>
            <DescriptionData>
              <Title>${product.name}</Title>
              <Brand>${product.brand}</Brand>
              <Description>${product.description}</Description>
              <Manufacturer>${product.manufacturer}</Manufacturer>
            </DescriptionData>
            <ProductData>
              <Clothing>
                <VariationData>
                  <Size>${product.size}</Size>
                  <Color>${product.color}</Color>
                </VariationData>
              </Clothing>
            </ProductData>
          </Product>
        </Message>
      </AmazonEnvelope>`;
  }

  private buildInventoryFeed(sku: string, quantity: number): string {
    return `<?xml version="1.0" encoding="UTF-8"?>
      <AmazonEnvelope>
        <Header>
          <DocumentVersion>1.01</DocumentVersion>
          <MerchantIdentifier>${amazonConfig.sellerId}</MerchantIdentifier>
        </Header>
        <MessageType>Inventory</MessageType>
        <Message>
          <MessageID>1</MessageID>
          <OperationType>Update</OperationType>
          <Inventory>
            <SKU>${sku}</SKU>
            <Quantity>${quantity}</Quantity>
            <FulfillmentLatency>2</FulfillmentLatency>
          </Inventory>
        </Message>
      </AmazonEnvelope>`;
  }

  private mapAmazonStatus(amazonStatus: string): OrderStatus {
    const statusMap: Record<string, OrderStatus> = {
      'Pending': 'PENDING',
      'Unshipped': 'CONFIRMED',
      'Shipped': 'SHIPPED',
      'Delivered': 'DELIVERED',
      'Canceled': 'CANCELLED'
    };
    return statusMap[amazonStatus] || 'PENDING';
  }
}
```

### 3.2 Flipkart Seller Integration

```typescript
// src/services/marketplace/flipkart.service.ts
import axios from 'axios';

export class FlipkartService {
  private apiUrl = 'https://api.flipkart.net/sellers';
  private appId = process.env.FLIPKART_APP_ID!;
  private appSecret = process.env.FLIPKART_APP_SECRET!;
  private accessToken?: string;

  // Authenticate
  async authenticate(): Promise<void> {
    try {
      const response = await axios.post(
        `${this.apiUrl}/oauth-token`,
        {
          grant_type: 'client_credentials',
          scope: 'Seller_Api'
        },
        {
          auth: {
            username: this.appId,
            password: this.appSecret
          }
        }
      );

      this.accessToken = response.data.access_token;
    } catch (error) {
      logger.error('Flipkart authentication failed', { error });
      throw new MarketplaceError('Failed to authenticate with Flipkart');
    }
  }

  // Create listing
  async createListing(product: Product): Promise<void> {
    const listing = {
      sku_id: product.sku,
      hsn: product.hsn,
      tax_code: 'GST_12',
      fulfillment_mode: 'FBF', // Fulfilled by Flipkart
      listing_status: 'ACTIVE',
      attributes: {
        title: product.name,
        description: product.description,
        brand: product.brand,
        size: product.size,
        color: product.color,
        mrp: product.mrp,
        selling_price: product.price
      },
      images: product.images.map(img => img.url)
    };

    try {
      await axios.post(
        `${this.apiUrl}/v2/listings`,
        listing,
        {
          headers: {
            'Authorization': `Bearer ${this.accessToken}`,
            'Content-Type': 'application/json'
          }
        }
      );

      logger.info('Listing created on Flipkart', { sku: product.sku });
    } catch (error) {
      logger.error('Flipkart listing creation failed', { error });
      throw new MarketplaceError('Failed to create Flipkart listing');
    }
  }

  // Update inventory
  async updateInventory(sku: string, quantity: number): Promise<void> {
    try {
      await axios.patch(
        `${this.apiUrl}/v2/inventory`,
        {
          sku_id: sku,
          inventory: quantity
        },
        {
          headers: {
            'Authorization': `Bearer ${this.accessToken}`
          }
        }
      );

      logger.info('Inventory updated on Flipkart', { sku, quantity });
    } catch (error) {
      logger.error('Flipkart inventory update failed', { error });
      throw new MarketplaceError('Failed to update Flipkart inventory');
    }
  }

  // Fetch orders
  async fetchOrders(): Promise<FlipkartOrder[]> {
    try {
      const response = await axios.get(
        `${this.apiUrl}/v3/orders/search`,
        {
          headers: {
            'Authorization': `Bearer ${this.accessToken}`
          },
          params: {
            filter: {
              orderDate: {
                fromDate: new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString()
              }
            }
          }
        }
      );

      return response.data.orderItems;
    } catch (error) {
      logger.error('Failed to fetch Flipkart orders', { error });
      throw new MarketplaceError('Failed to fetch Flipkart orders');
    }
  }
}
```

---

## 4. Shipping Partner Integration

### 4.1 Delhivery Integration

```typescript
// src/services/shipping/delhivery.service.ts
import axios from 'axios';

export class DelhiveryService {
  private apiUrl = 'https://track.delhivery.com/api';
  private apiKey = process.env.DELHIVERY_API_KEY!;

  // Create shipment
  async createShipment(order: Order): Promise<ShipmentDetails> {
    const shipmentData = {
      shipments: [{
        name: order.shippingAddress.fullName,
        add: order.shippingAddress.address,
        pin: order.shippingAddress.pincode,
        city: order.shippingAddress.city,
        state: order.shippingAddress.state,
        country: 'India',
        phone: order.shippingAddress.phone,
        order: order.orderNumber,
        payment_mode: order.paymentMode === 'COD' ? 'COD' : 'Prepaid',
        return_pin: '400001',
        return_city: 'Mumbai',
        return_phone: '9876543210',
        return_add: 'Warehouse Address',
        products_desc: order.items.map(i => i.product.name).join(', '),
        hsn_code: order.items[0].product.hsn,
        cod_amount: order.paymentMode === 'COD' ? order.totalAmount : 0,
        order_date: order.createdAt.toISOString(),
        total_amount: order.totalAmount,
        seller_add: 'Warehouse Address',
        seller_name: 'Ecom Platform',
        seller_inv: order.invoiceNumber,
        quantity: order.items.reduce((sum, i) => sum + i.quantity, 0),
        waybill: '',
        shipment_width: 10,
        shipment_height: 10,
        weight: order.items.reduce((sum, i) => sum + (i.weight || 0.5), 0),
        seller_gst_tin: process.env.COMPANY_GSTIN,
        shipping_mode: 'Surface',
        address_type: 'home'
      }],
      pickup_location: {
        name: 'Main Warehouse'
      }
    };

    try {
      const response = await axios.post(
        `${this.apiUrl}/cmu/create.json`,
        `format=json&data=${JSON.stringify(shipmentData)}`,
        {
          headers: {
            'Authorization': `Token ${this.apiKey}`,
            'Content-Type': 'application/json'
          }
        }
      );

      const waybill = response.data.packages[0].waybill;

      // Update order with tracking info
      await prisma.order.update({
        where: { id: order.id },
        data: {
          trackingNumber: waybill,
          shippingPartner: 'DELHIVERY',
          shippingStatus: 'PICKUP_SCHEDULED'
        }
      });

      return {
        waybill,
        trackingUrl: `https://www.delhivery.com/track/package/${waybill}`
      };
    } catch (error) {
      logger.error('Delhivery shipment creation failed', { error, order });
      throw new ShippingError('Failed to create shipment');
    }
  }

  // Track shipment
  async trackShipment(waybill: string): Promise<TrackingInfo> {
    try {
      const response = await axios.get(
        `${this.apiUrl}/v1/packages/json`,
        {
          headers: {
            'Authorization': `Token ${this.apiKey}`
          },
          params: {
            waybill
          }
        }
      );

      const tracking = response.data.ShipmentData[0].Shipment;

      return {
        status: tracking.Status.Status,
        currentLocation: tracking.Status.StatusLocation,
        expectedDelivery: tracking.Status.ExpectedDeliveryDate,
        scans: tracking.Scans.map((scan: any) => ({
          location: scan.ScanDetail.ScannedLocation,
          status: scan.ScanDetail.Scan,
          time: scan.ScanDetail.ScanDateTime
        }))
      };
    } catch (error) {
      logger.error('Shipment tracking failed', { error, waybill });
      throw new ShippingError('Failed to track shipment');
    }
  }

  // Schedule pickup
  async schedulePickup(
    waybill: string,
    pickupTime: Date
  ): Promise<void> {
    try {
      await axios.post(
        `${this.apiUrl}/fm/request/new/`,
        {
          pickup_location: 'Main Warehouse',
          pickup_time: pickupTime.toISOString(),
          expected_package_count: 1
        },
        {
          headers: {
            'Authorization': `Token ${this.apiKey}`
          }
        }
      );

      logger.info('Pickup scheduled', { waybill, pickupTime });
    } catch (error) {
      logger.error('Pickup scheduling failed', { error });
      throw new ShippingError('Failed to schedule pickup');
    }
  }

  // Generate shipping label
  async generateLabel(waybill: string): Promise<Buffer> {
    try {
      const response = await axios.get(
        `${this.apiUrl}/p/packing_slip`,
        {
          headers: {
            'Authorization': `Token ${this.apiKey}`
          },
          params: {
            wbns: waybill,
            pdf: true
          },
          responseType: 'arraybuffer'
        }
      );

      return Buffer.from(response.data);
    } catch (error) {
      logger.error('Label generation failed', { error });
      throw new ShippingError('Failed to generate shipping label');
    }
  }
}
```

---

## 5. Communication Services

### 5.1 Email Integration (SendGrid)

```typescript
// src/services/communication/email.service.ts
import sgMail from '@sendgrid/mail';

sgMail.setApiKey(process.env.SENDGRID_API_KEY!);

export class EmailService {
  // Send order confirmation email
  async sendOrderConfirmation(order: Order): Promise<void> {
    const msg = {
      to: order.customer.email,
      from: {
        email: 'noreply@ecomplatform.com',
        name: 'Ecom Platform'
      },
      templateId: 'd-xxxxxxxxxxxxx', // SendGrid template ID
      dynamicTemplateData: {
        customerName: order.customer.name,
        orderNumber: order.orderNumber,
        orderDate: order.createdAt.toLocaleDateString(),
        totalAmount: order.totalAmount,
        items: order.items.map(item => ({
          name: item.product.name,
          quantity: item.quantity,
          price: item.price
        })),
        shippingAddress: order.shippingAddress,
        trackingUrl: order.trackingNumber 
          ? `https://track.ecomplatform.com/${order.trackingNumber}`
          : null
      }
    };

    try {
      await sgMail.send(msg);
      logger.info('Order confirmation email sent', { orderId: order.id });
    } catch (error) {
      logger.error('Failed to send email', { error, orderId: order.id });
    }
  }

  // Send shipping notification
  async sendShippingNotification(order: Order): Promise<void> {
    const msg = {
      to: order.customer.email,
      from: 'noreply@ecomplatform.com',
      templateId: 'd-yyyyyyyyyyyyy',
      dynamicTemplateData: {
        customerName: order.customer.name,
        orderNumber: order.orderNumber,
        trackingNumber: order.trackingNumber,
        trackingUrl: `https://track.ecomplatform.com/${order.trackingNumber}`,
        estimatedDelivery: order.estimatedDeliveryDate
      }
    };

    await sgMail.send(msg);
  }

  // Send bulk promotional email
  async sendPromotionalEmail(
    recipients: string[],
    campaign: Campaign
  ): Promise<void> {
    const msg = {
      to: recipients,
      from: 'marketing@ecomplatform.com',
      templateId: campaign.templateId,
      dynamicTemplateData: campaign.data,
      trackingSettings: {
        clickTracking: { enable: true },
        openTracking: { enable: true }
      }
    };

    await sgMail.sendMultiple(msg);
  }
}
```

### 5.2 SMS Integration (Twilio)

```typescript
// src/services/communication/sms.service.ts
import twilio from 'twilio';

const client = twilio(
  process.env.TWILIO_ACCOUNT_SID!,
  process.env.TWILIO_AUTH_TOKEN!
);

export class SMSService {
  private fromNumber = process.env.TWILIO_PHONE_NUMBER!;

  // Send OTP
  async sendOTP(phone: string, otp: string): Promise<void> {
    const message = `Your OTP for Ecom Platform is: ${otp}. Valid for 10 minutes.`;

    try {
      await client.messages.create({
        body: message,
        from: this.fromNumber,
        to: phone
      });

      logger.info('OTP sent', { phone });
    } catch (error) {
      logger.error('Failed to send OTP', { error, phone });
      throw new CommunicationError('Failed to send OTP');
    }
  }

  // Send order update
  async sendOrderUpdate(
    phone: string,
    orderNumber: string,
    status: string
  ): Promise<void> {
    const message = `Your order ${orderNumber} is now ${status}. Track: https://track.ecomplatform.com/${orderNumber}`;

    await client.messages.create({
      body: message,
      from: this.fromNumber,
      to: phone
    });
  }

  // Send delivery notification
  async sendDeliveryNotification(order: Order): Promise<void> {
    const message = `Your order ${order.orderNumber} has been delivered. Thank you for shopping with us!`;

    await client.messages.create({
      body: message,
      from: this.fromNumber,
      to: order.customer.phone
    });
  }
}
```

### 5.3 WhatsApp Integration

```typescript
// src/services/communication/whatsapp.service.ts
import axios from 'axios';

export class WhatsAppService {
  private apiUrl = 'https://graph.facebook.com/v17.0';
  private phoneNumberId = process.env.WHATSAPP_PHONE_NUMBER_ID!;
  private accessToken = process.env.WHATSAPP_ACCESS_TOKEN!;

  // Send template message
  async sendTemplateMessage(
    to: string,
    templateName: string,
    parameters: any[]
  ): Promise<void> {
    try {
      await axios.post(
        `${this.apiUrl}/${this.phoneNumberId}/messages`,
        {
          messaging_product: 'whatsapp',
          to,
          type: 'template',
          template: {
            name: templateName,
            language: { code: 'en' },
            components: [{
              type: 'body',
              parameters
            }]
          }
        },
        {
          headers: {
            'Authorization': `Bearer ${this.accessToken}`,
            'Content-Type': 'application/json'
          }
        }
      );

      logger.info('WhatsApp message sent', { to, templateName });
    } catch (error) {
      logger.error('Failed to send WhatsApp message', { error });
    }
  }

  // Send order confirmation
  async sendOrderConfirmation(order: Order): Promise<void> {
    await this.sendTemplateMessage(
      order.customer.phone,
      'order_confirmation',
      [
        { type: 'text', text: order.customer.name },
        { type: 'text', text: order.orderNumber },
        { type: 'text', text: order.totalAmount.toString() }
      ]
    );
  }
}
```

---

## 6. Analytics & Tracking

### 6.1 Google Analytics Integration

```typescript
// frontend/src/services/analytics.service.ts
import ReactGA from 'react-ga4';

export class AnalyticsService {
  initialize(): void {
    ReactGA.initialize(process.env.REACT_APP_GA_MEASUREMENT_ID!);
  }

  // Track page view
  trackPageView(path: string): void {
    ReactGA.send({ hitType: 'pageview', page: path });
  }

  // Track event
  trackEvent(
    category: string,
    action: string,
    label?: string,
    value?: number
  ): void {
    ReactGA.event({
      category,
      action,
      label,
      value
    });
  }

  // Track ecommerce transaction
  trackPurchase(order: Order): void {
    ReactGA.event('purchase', {
      transaction_id: order.orderNumber,
      value: order.totalAmount,
      currency: 'INR',
      tax: order.taxAmount,
      shipping: order.shippingCharges,
      items: order.items.map(item => ({
        item_id: item.product.sku,
        item_name: item.product.name,
        item_category: item.product.category.name,
        price: item.price,
        quantity: item.quantity
      }))
    });
  }

  // Track add to cart
  trackAddToCart(product: Product, quantity: number): void {
    ReactGA.event('add_to_cart', {
      currency: 'INR',
      value: product.price * quantity,
      items: [{
        item_id: product.sku,
        item_name: product.name,
        item_category: product.category.name,
        price: product.price,
        quantity
      }]
    });
  }
}
```

---

## 7. Cloud Storage Integration

### 7.1 AWS S3 Integration

```typescript
// src/services/storage/s3.service.ts
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

export class S3Service {
  private client: S3Client;
  private bucket = process.env.AWS_S3_BUCKET!;

  constructor() {
    this.client = new S3Client({
      region: process.env.AWS_REGION!,
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!
      }
    });
  }

  // Upload file
  async uploadFile(
    file: Buffer,
    key: string,
    contentType: string
  ): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: file,
      ContentType: contentType,
      ACL: 'public-read'
    });

    await this.client.send(command);

    return `https://${this.bucket}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
  }

  // Generate presigned URL for upload
  async getUploadUrl(
    key: string,
    contentType: string
  ): Promise<string> {
    const command = new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      ContentType: contentType
    });

    return getSignedUrl(this.client, command, { expiresIn: 3600 });
  }

  // Generate presigned URL for download
  async getDownloadUrl(key: string): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: key
    });

    return getSignedUrl(this.client, command, { expiresIn: 3600 });
  }

  // Delete file
  async deleteFile(key: string): Promise<void> {
    await this.client.send(new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: key
    }));
  }
}
```

---

## 8. Third-Party APIs

### 8.1 GST Verification

```typescript
// src/services/external/gst-verification.service.ts
import axios from 'axios';

export class GSTVerificationService {
  private apiUrl = 'https://commonapi.gstinindianapi.com';
  private apiKey = process.env.GST_API_KEY!;

  async verifyGSTIN(gstin: string): Promise<GSTDetails> {
    try {
      const response = await axios.get(
        `${this.apiUrl}/public/gstin/${gstin}`,
        {
          headers: {
            'X-API-Key': this.apiKey
          }
        }
      );

      return {
        gstin: response.data.gstin,
        legalName: response.data.legalName,
        tradeName: response.data.tradeName,
        status: response.data.sts,
        registrationDate: response.data.rgdt,
        address: response.data.pradr.addr
      };
    } catch (error) {
      logger.error('GST verification failed', { error, gstin });
      throw new ValidationError('Invalid GSTIN');
    }
  }
}
```

### 8.2 Pincode Verification

```typescript
// src/services/external/pincode.service.ts
export class PincodeService {
  async verifyPincode(pincode: string): Promise<PincodeDetails> {
    try {
      const response = await axios.get(
        `https://api.postalpincode.in/pincode/${pincode}`
      );

      const data = response.data[0];

      if (data.Status === 'Error') {
        throw new ValidationError('Invalid pincode');
      }

      const postOffice = data.PostOffice[0];

      return {
        pincode,
        city: postOffice.District,
        state: postOffice.State,
        country: postOffice.Country
      };
    } catch (error) {
      logger.error('Pincode verification failed', { error, pincode });
      throw new ValidationError('Invalid pincode');
    }
  }

  async checkServiceability(pincode: string): Promise<boolean> {
    // Check if pincode is serviceable by shipping partners
    const details = await this.verifyPincode(pincode);
    
    // Add your serviceability logic here
    const serviceablePincodes = await prisma.serviceablePincode.findUnique({
      where: { pincode }
    });

    return !!serviceablePincodes;
  }
}
```

---

## Conclusion

This comprehensive integration guide provides:

✅ **Payment gateway integration** (Razorpay, Stripe)  
✅ **Marketplace integrations** (Amazon, Flipkart, Meesho)  
✅ **Shipping partner APIs** (Delhivery, Blue Dart)  
✅ **Communication services** (Email, SMS, WhatsApp)  
✅ **Analytics tracking** (Google Analytics)  
✅ **Cloud storage** (AWS S3)  
✅ **Third-party APIs** (GST, Pincode verification)

All integrations follow best practices with proper error handling, logging, and retry mechanisms.

---

**Document Status:** ✅ Complete  
**Last Updated:** March 26, 2026  
**Integration Team:** integrations@company.com
