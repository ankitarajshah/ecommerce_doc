# 08. Business Logic & Rules
## Enterprise Men's Clothing Manufacturing & Sales Platform

**Version:** 2.0  
**Last Updated:** March 26, 2026  
**Status:** ✅ Production-Ready

---

## 📋 Table of Contents

1. [Pricing Logic](#1-pricing-logic)
2. [Inventory Calculations](#2-inventory-calculations)
3. [Commission Logic](#3-commission-logic)
4. [Tax Calculations](#4-tax-calculations)
5. [Shipping Cost Calculation](#5-shipping-cost-calculation)
6. [Credit Limit Management](#6-credit-limit-management)
7. [Loyalty Points System](#7-loyalty-points-system)
8. [Discount Rules](#8-discount-rules)

---

## 1. Pricing Logic

### 1.1 Price Calculation Hierarchy

```typescript
// src/services/pricing/pricing.service.ts

interface PriceContext {
  customer: {
    id: number;
    type: 'retail' | 'wholesale' | 'agent';
    segment: 'regular' | 'vip' | 'premium';
    loyaltyTier: string;
  };
  product: {
    id: number;
    retailPrice: number;
    wholesalePrice: number;
    cost: number;
    compareAtPrice?: number;
  };
  quantity: number;
  couponCode?: string;
  seasonalOffer?: string;
}

export class PricingService {
  /**
   * Calculate final price for a product
   */
  async calculatePrice(context: PriceContext): Promise<PriceCalculation> {
    // 1. Determine base price based on customer type
    const basePrice = this.getBasePrice(context);

    // 2. Apply quantity-based discounts
    const quantityDiscount = this.calculateQuantityDiscount(
      basePrice,
      context.quantity
    );

    // 3. Apply customer segment discounts
    const segmentDiscount = this.calculateSegmentDiscount(
      basePrice,
      context.customer.segment
    );

    // 4. Apply seasonal offers
    const seasonalDiscount = await this.calculateSeasonalDiscount(
      basePrice,
      context.product.id,
      context.seasonalOffer
    );

    // 5. Apply coupon code
    const couponDiscount = await this.applyCouponCode(
      basePrice,
      context.couponCode
    );

    // 6. Calculate final price
    const totalDiscount =
      quantityDiscount + segmentDiscount + seasonalDiscount + couponDiscount;

    const finalPrice = Math.max(
      basePrice - totalDiscount,
      context.product.cost * 1.1 // Minimum 10% margin
    );

    return {
      basePrice,
      discounts: {
        quantity: quantityDiscount,
        segment: segmentDiscount,
        seasonal: seasonalDiscount,
        coupon: couponDiscount,
        total: totalDiscount,
      },
      finalPrice,
      savings: basePrice - finalPrice,
      savingsPercentage: ((basePrice - finalPrice) / basePrice) * 100,
    };
  }

  /**
   * Get base price based on customer type
   */
  private getBasePrice(context: PriceContext): number {
    switch (context.customer.type) {
      case 'retail':
        return context.product.retailPrice;

      case 'wholesale':
        return context.product.wholesalePrice;

      case 'agent':
        // Agents get wholesale price minus agent discount
        return context.product.wholesalePrice * 0.95; // 5% additional discount

      default:
        return context.product.retailPrice;
    }
  }

  /**
   * Calculate quantity-based discount
   */
  private calculateQuantityDiscount(
    basePrice: number,
    quantity: number
  ): number {
    let discountPercentage = 0;

    if (quantity >= 100) {
      discountPercentage = 15; // 15% for 100+ items
    } else if (quantity >= 50) {
      discountPercentage = 10; // 10% for 50-99 items
    } else if (quantity >= 20) {
      discountPercentage = 5; // 5% for 20-49 items
    } else if (quantity >= 10) {
      discountPercentage = 3; // 3% for 10-19 items
    }

    return (basePrice * discountPercentage) / 100;
  }

  /**
   * Calculate customer segment discount
   */
  private calculateSegmentDiscount(
    basePrice: number,
    segment: string
  ): number {
    const segmentDiscounts = {
      regular: 0,
      vip: 5, // 5% for VIP customers
      premium: 10, // 10% for premium customers
    };

    const discountPercentage = segmentDiscounts[segment] || 0;
    return (basePrice * discountPercentage) / 100;
  }

  /**
   * Calculate seasonal discount
   */
  private async calculateSeasonalDiscount(
    basePrice: number,
    productId: number,
    offerCode?: string
  ): Promise<number> {
    if (!offerCode) return 0;

    const offer = await prisma.seasonalOffers.findFirst({
      where: {
        code: offerCode,
        isActive: true,
        startDate: { lte: new Date() },
        endDate: { gte: new Date() },
      },
      include: {
        applicableProducts: true,
      },
    });

    if (!offer) return 0;

    // Check if product is eligible
    const isEligible =
      offer.applicableProducts.length === 0 ||
      offer.applicableProducts.some((p) => p.productId === productId);

    if (!isEligible) return 0;

    if (offer.discountType === 'percentage') {
      return (basePrice * offer.discountValue) / 100;
    } else if (offer.discountType === 'fixed') {
      return offer.discountValue;
    }

    return 0;
  }

  /**
   * Apply coupon code
   */
  private async applyCouponCode(
    basePrice: number,
    couponCode?: string
  ): Promise<number> {
    if (!couponCode) return 0;

    const coupon = await prisma.coupons.findUnique({
      where: { code: couponCode },
    });

    if (!coupon) throw new Error('Invalid coupon code');

    // Validate coupon
    if (!coupon.isActive) throw new Error('Coupon is not active');
    if (new Date() > coupon.expiryDate) throw new Error('Coupon expired');
    if (coupon.usageCount >= coupon.usageLimit)
      throw new Error('Coupon usage limit reached');
    if (basePrice < coupon.minOrderValue)
      throw new Error(
        `Minimum order value of ₹${coupon.minOrderValue} required`
      );

    let discount = 0;

    if (coupon.discountType === 'percentage') {
      discount = (basePrice * coupon.discountValue) / 100;
      if (coupon.maxDiscount) {
        discount = Math.min(discount, coupon.maxDiscount);
      }
    } else if (coupon.discountType === 'fixed') {
      discount = coupon.discountValue;
    }

    return discount;
  }
}
```

### 1.2 Pricing Rules Matrix

```yaml
Customer Type Pricing:
  Retail Customer:
    Base Price: Retail Price
    Quantity Discount: Yes
    Segment Discount: Yes
    Coupon: Yes
    Seasonal Offer: Yes

  Wholesale Customer:
    Base Price: Wholesale Price
    Quantity Discount: Yes
    Segment Discount: No
    Coupon: Limited
    Seasonal Offer: No

  Agent Customer:
    Base Price: Wholesale Price - 5%
    Quantity Discount: Yes
    Segment Discount: No
    Coupon: No
    Seasonal Offer: No

Quantity-Based Discounts:
  - 1-9 items: 0% discount
  - 10-19 items: 3% discount
  - 20-49 items: 5% discount
  - 50-99 items: 10% discount
  - 100+ items: 15% discount

Customer Segment Discounts:
  - Regular: 0% additional discount
  - VIP: 5% additional discount
  - Premium: 10% additional discount

Minimum Margin Rule:
  Final Price >= Cost Price × 1.10 (Minimum 10% margin)
```

---

## 2. Inventory Calculations

### 2.1 Stock Availability Logic

```typescript
// src/services/inventory/inventory-calculation.service.ts

export class InventoryCalculationService {
  /**
   * Calculate available quantity for a product variant
   */
  async calculateAvailableQuantity(variantId: number): Promise<number> {
    const inventory = await prisma.inventory.aggregate({
      where: { variantId },
      _sum: {
        quantity: true,
        reserved: true,
      },
    });

    const totalQuantity = inventory._sum.quantity || 0;
    const reservedQuantity = inventory._sum.reserved || 0;

    return Math.max(totalQuantity - reservedQuantity, 0);
  }

  /**
   * Reserve inventory for an order
   */
  async reserveInventory(
    variantId: number,
    quantity: number,
    orderId: number
  ): Promise<void> {
    // Get warehouses with available stock (FIFO order)
    const inventoryRecords = await prisma.inventory.findMany({
      where: {
        variantId,
      },
      orderBy: {
        createdAt: 'asc', // FIFO - First In First Out
      },
    });

    let remainingToReserve = quantity;

    for (const record of inventoryRecords) {
      if (remainingToReserve <= 0) break;

      const availableInWarehouse = record.quantity - record.reserved;

      if (availableInWarehouse > 0) {
        const toReserve = Math.min(remainingToReserve, availableInWarehouse);

        await prisma.inventory.update({
          where: { id: record.id },
          data: {
            reserved: record.reserved + toReserve,
          },
        });

        // Create reservation record
        await prisma.inventoryReservations.create({
          data: {
            inventoryId: record.id,
            orderId,
            quantity: toReserve,
          },
        });

        remainingToReserve -= toReserve;
      }
    }

    if (remainingToReserve > 0) {
      throw new Error(`Insufficient stock. Need ${remainingToReserve} more.`);
    }
  }

  /**
   * Release reserved inventory (when order is cancelled)
   */
  async releaseReservation(orderId: number): Promise<void> {
    const reservations = await prisma.inventoryReservations.findMany({
      where: { orderId },
    });

    for (const reservation of reservations) {
      await prisma.inventory.update({
        where: { id: reservation.inventoryId },
        data: {
          reserved: {
            decrement: reservation.quantity,
          },
        },
      });

      await prisma.inventoryReservations.delete({
        where: { id: reservation.id },
      });
    }
  }

  /**
   * Fulfill order (convert reservation to actual stock reduction)
   */
  async fulfillOrder(orderId: number): Promise<void> {
    const reservations = await prisma.inventoryReservations.findMany({
      where: { orderId },
    });

    for (const reservation of reservations) {
      await prisma.inventory.update({
        where: { id: reservation.inventoryId },
        data: {
          quantity: {
            decrement: reservation.quantity,
          },
          reserved: {
            decrement: reservation.quantity,
          },
        },
      });

      // Record stock movement
      await prisma.stockMovements.create({
        data: {
          inventoryId: reservation.inventoryId,
          type: 'out',
          quantity: reservation.quantity,
          reason: 'Order Fulfillment',
          reference: `ORDER-${orderId}`,
        },
      });

      await prisma.inventoryReservations.delete({
        where: { id: reservation.id },
      });
    }
  }

  /**
   * Check reorder point
   */
  async checkReorderPoint(variantId: number): Promise<boolean> {
    const totalQuantity = await this.calculateAvailableQuantity(variantId);

    const variant = await prisma.productVariants.findUnique({
      where: { id: variantId },
    });

    if (!variant) return false;

    const reorderPoint = variant.reorderPoint || 20;

    return totalQuantity <= reorderPoint;
  }

  /**
   * Calculate reorder quantity
   */
  async calculateReorderQuantity(variantId: number): Promise<number> {
    const variant = await prisma.productVariants.findUnique({
      where: { id: variantId },
    });

    if (!variant) throw new Error('Variant not found');

    const currentQuantity = await this.calculateAvailableQuantity(variantId);

    // Calculate average daily sales (last 30 days)
    const thirtyDaysAgo = new Date();
    thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

    const salesData = await prisma.orderItems.aggregate({
      where: {
        variantId,
        order: {
          status: 'delivered',
          createdAt: {
            gte: thirtyDaysAgo,
          },
        },
      },
      _sum: {
        quantity: true,
      },
    });

    const totalSold = salesData._sum.quantity || 0;
    const averageDailySales = totalSold / 30;

    // Lead time (days) - time from order to delivery
    const leadTimeDays = 14;

    // Safety stock (7 days worth)
    const safetyStockDays = 7;

    // Reorder quantity = (Lead time + Safety stock) × Average daily sales - Current quantity
    const reorderQuantity =
      (leadTimeDays + safetyStockDays) * averageDailySales - currentQuantity;

    // Round up to nearest 10
    return Math.ceil(Math.max(reorderQuantity, 0) / 10) * 10;
  }
}
```

### 2.2 Inventory Status Logic

```yaml
Stock Status Calculation:
  In Stock:
    Condition: Available Quantity > Min Stock Level
    Display: "In Stock"
    Color: Green

  Low Stock:
    Condition: Available Quantity <= Min Stock Level AND > 0
    Display: "Low Stock - Only X left"
    Color: Orange

  Out of Stock:
    Condition: Available Quantity = 0
    Display: "Out of Stock"
    Color: Red

  Back Order:
    Condition: Reserved > Available
    Display: "Available on Back Order"
    Color: Yellow

Reorder Point Calculation:
  Reorder Point = (Average Daily Sales × Lead Time Days) + Safety Stock
  
  Example:
  - Average Daily Sales: 5 units/day
  - Lead Time: 14 days
  - Safety Stock: 7 days worth = 35 units
  - Reorder Point = (5 × 14) + 35 = 105 units
```

---

## 3. Commission Logic

### 3.1 Commission Calculation

```typescript
// src/services/commission/commission.service.ts

export class CommissionService {
  /**
   * Calculate commission for an order
   */
  async calculateCommission(orderId: number): Promise<CommissionCalculation> {
    const order = await prisma.orders.findUnique({
      where: { id: orderId },
      include: {
        agent: true,
        items: {
          include: {
            variant: {
              include: {
                product: {
                  include: {
                    category: true,
                  },
                },
              },
            },
          },
        },
      },
    });

    if (!order || !order.agent) {
      throw new Error('Order or agent not found');
    }

    // 1. Determine agent tier
    const agentTier = await this.getAgentTier(order.agentId);

    // 2. Get base commission rate
    const baseCommissionRate = this.getBaseCommissionRate(agentTier);

    // 3. Calculate commission per item
    let totalCommission = 0;
    const itemCommissions = [];

    for (const item of order.items) {
      // Get category multiplier
      const categoryMultiplier = this.getCategoryMultiplier(
        item.variant.product.category.slug
      );

      // Calculate item commission
      const itemSubtotal = item.price * item.quantity;
      const effectiveRate = baseCommissionRate * categoryMultiplier;
      const itemCommission = (itemSubtotal * effectiveRate) / 100;

      itemCommissions.push({
        itemId: item.id,
        productName: item.variant.product.name,
        subtotal: itemSubtotal,
        rate: effectiveRate,
        commission: itemCommission,
      });

      totalCommission += itemCommission;
    }

    // 4. Apply performance bonuses
    const performanceBonus = await this.calculatePerformanceBonus(
      order.agentId,
      totalCommission
    );

    totalCommission += performanceBonus;

    return {
      orderId: order.id,
      agentId: order.agentId,
      agentName: order.agent.firstName + ' ' + order.agent.lastName,
      agentTier,
      baseRate: baseCommissionRate,
      orderValue: order.total,
      itemCommissions,
      baseCommission: totalCommission - performanceBonus,
      performanceBonus,
      totalCommission,
      tds: totalCommission * 0.1, // 10% TDS
      netPayable: totalCommission * 0.9,
      status: 'pending',
      eligibleDate: this.calculateEligibilityDate(order.deliveredAt),
    };
  }

  /**
   * Get agent tier based on monthly sales
   */
  private async getAgentTier(agentId: number): Promise<string> {
    const currentMonth = new Date();
    currentMonth.setDate(1);
    currentMonth.setHours(0, 0, 0, 0);

    const monthlySales = await prisma.orders.aggregate({
      where: {
        agentId,
        status: 'delivered',
        createdAt: {
          gte: currentMonth,
        },
      },
      _sum: {
        total: true,
      },
    });

    const totalSales = monthlySales._sum.total || 0;

    if (totalSales >= 200000) return 'platinum'; // ₹2L+
    if (totalSales >= 100000) return 'gold'; // ₹1L-₹2L
    if (totalSales >= 50000) return 'silver'; // ₹50K-₹1L
    return 'bronze'; // <₹50K
  }

  /**
   * Get base commission rate by tier
   */
  private getBaseCommissionRate(tier: string): number {
    const rates = {
      bronze: 5.0,
      silver: 6.0,
      gold: 7.0,
      platinum: 8.0,
    };

    return rates[tier] || 5.0;
  }

  /**
   * Get category multiplier
   */
  private getCategoryMultiplier(categorySlug: string): number {
    const multipliers = {
      'premium-collection': 1.2, // +20% for premium
      'clearance-sale': 0.8, // -20% for clearance
      'seasonal-wear': 1.1, // +10% for seasonal
      default: 1.0,
    };

    return multipliers[categorySlug] || multipliers.default;
  }

  /**
   * Calculate performance bonus
   */
  private async calculatePerformanceBonus(
    agentId: number,
    baseCommission: number
  ): Promise<number> {
    const currentMonth = new Date();
    currentMonth.setDate(1);

    // Get orders count this month
    const ordersCount = await prisma.orders.count({
      where: {
        agentId,
        status: 'delivered',
        createdAt: {
          gte: currentMonth,
        },
      },
    });

    // Bonus for high order volume
    if (ordersCount >= 50) return baseCommission * 0.1; // 10% bonus
    if (ordersCount >= 30) return baseCommission * 0.05; // 5% bonus

    return 0;
  }

  /**
   * Calculate commission eligibility date
   */
  private calculateEligibilityDate(deliveredAt: Date): Date {
    const eligibleDate = new Date(deliveredAt);
    eligibleDate.setDate(eligibleDate.getDate() + 15); // 15 days return window
    return eligibleDate;
  }
}
```

### 3.2 Commission Structure

```yaml
Agent Tier Structure:
  Bronze (<₹50K/month):
    Base Rate: 5%
    Premium Products: +1% (6% total)
    Clearance: -1% (4% total)
  
  Silver (₹50K-₹1L/month):
    Base Rate: 6%
    Premium Products: +1.2% (7.2% total)
    Clearance: -1.2% (4.8% total)
  
  Gold (₹1L-₹2L/month):
    Base Rate: 7%
    Premium Products: +1.4% (8.4% total)
    Clearance: -1.4% (5.6% total)
  
  Platinum (>₹2L/month):
    Base Rate: 8%
    Premium Products: +1.6% (9.6% total)
    Clearance: -1.6% (6.4% total)

Performance Bonuses:
  - 30+ orders/month: +5% bonus on total commission
  - 50+ orders/month: +10% bonus on total commission
  - Top 10 agents: Additional ₹5,000 monthly bonus

Commission Payout Rules:
  - Lock-in Period: 15 days (return window)
  - Payout Frequency: Monthly (1st of every month)
  - TDS Deduction: 10%
  - Minimum Payout: ₹500
  - Carry Forward: If < ₹500, carry to next month

Example Calculation:
  Order Value: ₹10,000
  Agent Tier: Silver (6%)
  Product: Premium Shirt (×1.2)
  Effective Rate: 7.2%
  Base Commission: ₹720
  Performance Bonus (30 orders): ₹36
  Total Commission: ₹756
  TDS (10%): ₹75.60
  Net Payable: ₹680.40
```

---

## 4. Tax Calculations

### 4.1 GST Calculation Logic

```typescript
// src/services/tax/tax-calculation.service.ts

export class TaxCalculationService {
  /**
   * Calculate GST for an order
   */
  async calculateGST(order: OrderCalculationInput): Promise<TaxCalculation> {
    const { items, shippingAddress, billingAddress } = order;

    // Determine if it's interstate or intrastate
    const isInterstate =
      shippingAddress.state !== billingAddress.state ||
      shippingAddress.state !== 'Maharashtra'; // Assuming business in Maharashtra

    let totalTax = 0;
    const itemTaxes = [];

    for (const item of items) {
      // Get HSN code and tax rate for product
      const product = await prisma.products.findUnique({
        where: { id: item.productId },
        select: {
          hsnCode: true,
          gstRate: true,
        },
      });

      const gstRate = product?.gstRate || 18; // Default 18% for apparel

      const itemSubtotal = item.price * item.quantity;
      const itemTax = (itemSubtotal * gstRate) / 100;

      if (isInterstate) {
        // IGST (Integrated GST)
        itemTaxes.push({
          itemId: item.id,
          hsnCode: product?.hsnCode || '6203',
          taxableAmount: itemSubtotal,
          igst: itemTax,
          igstRate: gstRate,
          cgst: 0,
          sgst: 0,
          totalTax: itemTax,
        });
      } else {
        // CGST + SGST (Central + State GST)
        const cgst = itemTax / 2;
        const sgst = itemTax / 2;

        itemTaxes.push({
          itemId: item.id,
          hsnCode: product?.hsnCode || '6203',
          taxableAmount: itemSubtotal,
          igst: 0,
          cgst,
          cgstRate: gstRate / 2,
          sgst,
          sgstRate: gstRate / 2,
          totalTax: itemTax,
        });
      }

      totalTax += itemTax;
    }

    return {
      isInterstate,
      itemTaxes,
      totalTaxableAmount: items.reduce(
        (sum, item) => sum + item.price * item.quantity,
        0
      ),
      totalTax,
      roundOff: this.calculateRoundOff(order.subtotal + totalTax),
    };
  }

  /**
   * Calculate round-off
   */
  private calculateRoundOff(amount: number): number {
    const rounded = Math.round(amount);
    return rounded - amount;
  }

  /**
   * Get HSN code based on product type
   */
  private getHSNCode(productType: string): string {
    const hsnCodes = {
      shirts: '6205',
      trousers: '6203',
      jackets: '6201',
      tshirts: '6109',
      jeans: '6203',
      suits: '6203',
    };

    return hsnCodes[productType] || '6203';
  }
}
```

### 4.2 GST Rate Structure

```yaml
GST Rates for Clothing:
  Ready-made Garments:
    - Retail Price > ₹1000: 12% GST
    - Retail Price ≤ ₹1000: 5% GST
  
  Premium/Branded: 18% GST
  
  Fabrics: 5% GST
  
  Accessories:
    - Belts, Wallets: 18% GST
    - Ties, Pocket Squares: 12% GST

HSN Codes:
  6201: Men's overcoats, jackets, blazers
  6203: Men's suits, trousers, shorts
  6205: Men's shirts
  6109: T-shirts, singlets
  6110: Sweaters, pullovers
  6217: Accessories (ties, belts, etc.)

Interstate vs Intrastate:
  Interstate (Different States):
    - IGST: Full GST rate
    - CGST: 0%
    - SGST: 0%
  
  Intrastate (Same State):
    - IGST: 0%
    - CGST: Half of GST rate
    - SGST: Half of GST rate

Example Calculation:
  Product: Premium Formal Shirt
  Price: ₹1,299
  Quantity: 2
  Subtotal: ₹2,598
  GST Rate: 18%
  
  Interstate (Mumbai → Delhi):
    IGST: ₹2,598 × 18% = ₹467.64
    Total: ₹3,065.64
  
  Intrastate (Mumbai → Pune):
    CGST: ₹2,598 × 9% = ₹233.82
    SGST: ₹2,598 × 9% = ₹233.82
    Total: ₹3,065.64
```

---

## 5. Shipping Cost Calculation

### 5.1 Shipping Logic

```typescript
// src/services/shipping/shipping-calculator.service.ts

export class ShippingCalculatorService {
  /**
   * Calculate shipping cost
   */
  async calculateShippingCost(order: ShippingInput): Promise<ShippingCost> {
    const { items, destination, orderValue } = order;

    // Free shipping for orders above threshold
    if (orderValue >= 999) {
      return {
        method: 'standard',
        cost: 0,
        estimatedDays: this.getEstimatedDeliveryDays(destination),
        freeShipping: true,
      };
    }

    // Calculate total weight
    const totalWeight = await this.calculateTotalWeight(items);

    // Get distance zone
    const zone = await this.getShippingZone(destination);

    // Base rate calculation
    let shippingCost = this.calculateBaseCost(totalWeight, zone);

    // Add handling fee for fragile items
    const hasFragile = items.some((item) => item.isFragile);
    if (hasFragile) {
      shippingCost += 50;
    }

    // Rural area surcharge
    if (destination.type === 'rural') {
      shippingCost += 30;
    }

    // Cash on delivery charges
    if (order.paymentMethod === 'cod') {
      shippingCost += 40;
    }

    return {
      method: 'standard',
      cost: Math.round(shippingCost),
      estimatedDays: this.getEstimatedDeliveryDays(destination),
      freeShipping: false,
      breakdown: {
        baseShipping: this.calculateBaseCost(totalWeight, zone),
        fragileHandling: hasFragile ? 50 : 0,
        ruralSurcharge: destination.type === 'rural' ? 30 : 0,
        codCharges: order.paymentMethod === 'cod' ? 40 : 0,
      },
    };
  }

  /**
   * Calculate total weight of items
   */
  private async calculateTotalWeight(items: OrderItem[]): Promise<number> {
    let totalWeight = 0;

    for (const item of items) {
      const variant = await prisma.productVariants.findUnique({
        where: { id: item.variantId },
        select: { weight: true },
      });

      const weight = variant?.weight || 250; // Default 250g per item
      totalWeight += weight * item.quantity;
    }

    return totalWeight; // in grams
  }

  /**
   * Get shipping zone
   */
  private async getShippingZone(
    destination: Address
  ): Promise<'local' | 'zonal' | 'national'> {
    // Mumbai (business location)
    if (destination.city === 'Mumbai' && destination.state === 'Maharashtra') {
      return 'local';
    }

    // Maharashtra and neighboring states
    const zonalStates = [
      'Maharashtra',
      'Gujarat',
      'Goa',
      'Madhya Pradesh',
      'Karnataka',
    ];
    if (zonalStates.includes(destination.state)) {
      return 'zonal';
    }

    return 'national';
  }

  /**
   * Calculate base shipping cost
   */
  private calculateBaseCost(
    weightInGrams: number,
    zone: 'local' | 'zonal' | 'national'
  ): number {
    const weightInKg = Math.ceil(weightInGrams / 1000);

    const rates = {
      local: {
        base: 40, // First 500g
        additional: 10, // Per additional 500g
      },
      zonal: {
        base: 60,
        additional: 15,
      },
      national: {
        base: 80,
        additional: 20,
      },
    };

    const rate = rates[zone];
    const additionalWeight = Math.max(0, weightInKg - 0.5);
    const additionalUnits = Math.ceil(additionalWeight / 0.5);

    return rate.base + additionalUnits * rate.additional;
  }

  /**
   * Get estimated delivery days
   */
  private getEstimatedDeliveryDays(destination: Address): string {
    const zone = this.getShippingZone(destination);

    const estimates = {
      local: '1-2 days',
      zonal: '3-5 days',
      national: '5-7 days',
    };

    return estimates[zone];
  }
}
```

### 5.2 Shipping Rate Card

```yaml
Free Shipping:
  Condition: Order Value >= ₹999
  Applies To: All locations

Standard Shipping Rates:
  Local (Mumbai):
    First 500g: ₹40
    Additional 500g: ₹10
    Estimated Delivery: 1-2 days
  
  Zonal (Maharashtra + Neighboring States):
    First 500g: ₹60
    Additional 500g: ₹15
    Estimated Delivery: 3-5 days
  
  National (Rest of India):
    First 500g: ₹80
    Additional 500g: ₹20
    Estimated Delivery: 5-7 days

Additional Charges:
  Fragile Item Handling: ₹50
  Rural Area Surcharge: ₹30
  Cash on Delivery (COD): ₹40

Express Shipping (Optional):
  Local: +₹100 (Same day)
  Zonal: +₹150 (1-2 days)
  National: +₹200 (2-3 days)

Weight Calculation:
  - Shirts: 250g each
  - Trousers: 400g each
  - Jackets: 600g each
  - T-shirts: 180g each
  - Jeans: 500g each

Example:
  Order: 2 Shirts + 1 Trouser
  Weight: (250g × 2) + 400g = 900g
  Zone: National
  Base Cost: ₹80 (first 500g) + ₹20 (next 500g) = ₹100
  COD Charge: ₹40
  Total Shipping: ₹140
```

---

## 6. Credit Limit Management

### 6.1 Credit Limit Logic

```typescript
// src/services/credit/credit-management.service.ts

export class CreditManagementService {
  /**
   * Check if customer has sufficient credit
   */
  async checkCreditLimit(
    customerId: number,
    orderAmount: number
  ): Promise<CreditCheckResult> {
    const customer = await prisma.customers.findUnique({
      where: { id: customerId },
      select: {
        creditLimit: true,
        creditUsed: true,
      },
    });

    if (!customer) {
      throw new Error('Customer not found');
    }

    // Calculate outstanding amount
    const outstanding = await this.getOutstandingAmount(customerId);

    const availableCredit = customer.creditLimit - outstanding;
    const hasCredit = availableCredit >= orderAmount;

    return {
      creditLimit: customer.creditLimit,
      creditUsed: outstanding,
      availableCredit,
      requestedAmount: orderAmount,
      hasCredit,
      shortfall: hasCredit ? 0 : orderAmount - availableCredit,
    };
  }

  /**
   * Get outstanding amount
   */
  private async getOutstandingAmount(customerId: number): Promise<number> {
    const unpaidOrders = await prisma.orders.aggregate({
      where: {
        customerId,
        paymentStatus: {
          in: ['pending', 'partial'],
        },
        status: {
          not: 'cancelled',
        },
      },
      _sum: {
        total: true,
      },
    });

    return unpaidOrders._sum.total || 0;
  }

  /**
   * Update credit limit (for review)
   */
  async updateCreditLimit(customerId: number): Promise<number> {
    const customer = await prisma.customers.findUnique({
      where: { id: customerId },
      include: {
        orders: {
          where: {
            status: 'delivered',
          },
          orderBy: {
            createdAt: 'desc',
          },
          take: 12, // Last 12 months
        },
      },
    });

    if (!customer) {
      throw new Error('Customer not found');
    }

    // Calculate metrics
    const totalOrders = customer.orders.length;
    const totalSpent = customer.orders.reduce(
      (sum, order) => sum + order.total,
      0
    );
    const averageOrderValue = totalOrders > 0 ? totalSpent / totalOrders : 0;

    // Check payment history
    const latePayments = await prisma.payments.count({
      where: {
        customerId,
        status: 'late',
      },
    });

    const paymentScore = Math.max(100 - latePayments * 10, 0);

    // Calculate new credit limit
    let newCreditLimit = 0;

    if (paymentScore >= 90 && totalOrders >= 10) {
      newCreditLimit = averageOrderValue * 3; // 3x average order
    } else if (paymentScore >= 70 && totalOrders >= 5) {
      newCreditLimit = averageOrderValue * 2; // 2x average order
    } else if (paymentScore >= 50 && totalOrders >= 3) {
      newCreditLimit = averageOrderValue * 1.5; // 1.5x average order
    } else {
      newCreditLimit = 5000; // Minimum credit limit
    }

    // Cap maximum credit limit
    newCreditLimit = Math.min(newCreditLimit, 100000);

    await prisma.customers.update({
      where: { id: customerId },
      data: { creditLimit: Math.round(newCreditLimit) },
    });

    return Math.round(newCreditLimit);
  }
}
```

### 6.2 Credit Limit Rules

```yaml
Initial Credit Limit:
  New Customer: ₹5,000
  After 3 Orders: Review for increase
  After 10 Orders: Auto-increase if good payment history

Credit Limit Calculation:
  Formula: Average Order Value × Multiplier
  
  Multipliers Based on Payment Score:
    - 90-100 (Excellent): 3x
    - 70-89 (Good): 2x
    - 50-69 (Fair): 1.5x
    - <50 (Poor): No increase

Maximum Credit Limit: ₹1,00,000

Payment Score Calculation:
  Base Score: 100
  Deductions:
    - Each late payment: -10 points
    - Each disputed payment: -20 points
    - Each returned check: -30 points

Credit Review Triggers:
  - Every 6 months automatically
  - After 10 successful orders
  - After any late payment
  - On customer request

Example:
  Customer A:
    - Total Orders: 15
    - Total Spent: ₹75,000
    - Average Order: ₹5,000
    - Late Payments: 1
    - Payment Score: 90
    - Calculated Limit: ₹5,000 × 3 = ₹15,000
```

---

## 7. Loyalty Points System

### 7.1 Points Earning Logic

```typescript
// src/services/loyalty/loyalty-points.service.ts

export class LoyaltyPointsService {
  /**
   * Calculate loyalty points for an order
   */
  async calculatePoints(orderId: number): Promise<PointsCalculation> {
    const order = await prisma.orders.findUnique({
      where: { id: orderId },
      include: {
        customer: true,
        items: {
          include: {
            variant: {
              include: {
                product: {
                  include: {
                    category: true,
                  },
                },
              },
            },
          },
        },
      },
    });

    if (!order) {
      throw new Error('Order not found');
    }

    // Base points: ₹100 = 1 point
    const basePoints = Math.floor(order.total / 100);

    // Tier multiplier
    const tierMultiplier = this.getTierMultiplier(order.customer.loyaltyTier);

    // Category bonus
    let categoryBonus = 0;
    for (const item of order.items) {
      const bonus = this.getCategoryBonus(
        item.variant.product.category.slug
      );
      categoryBonus += (item.price * item.quantity * bonus) / 100;
    }

    // Special event bonus (double points weekend)
    const eventMultiplier = await this.getEventMultiplier();

    const totalPoints = Math.floor(
      (basePoints * tierMultiplier + categoryBonus) * eventMultiplier
    );

    return {
      orderId: order.id,
      orderValue: order.total,
      basePoints,
      tierMultiplier,
      categoryBonus: Math.floor(categoryBonus),
      eventMultiplier,
      totalPoints,
      expiryDate: this.calculateExpiryDate(),
    };
  }

  /**
   * Get tier multiplier
   */
  private getTierMultiplier(tier: string): number {
    const multipliers = {
      bronze: 1.0,
      silver: 1.2,
      gold: 1.5,
      platinum: 2.0,
    };

    return multipliers[tier] || 1.0;
  }

  /**
   * Get category bonus
   */
  private getCategoryBonus(categorySlug: string): number {
    const bonuses = {
      'premium-collection': 50, // 50% extra points
      'new-arrivals': 25, // 25% extra points
      'seasonal-wear': 10, // 10% extra points
    };

    return bonuses[categorySlug] || 0;
  }

  /**
   * Redeem points
   */
  async redeemPoints(
    customerId: number,
    points: number
  ): Promise<RedemptionResult> {
    const customer = await prisma.customers.findUnique({
      where: { id: customerId },
      select: {
        loyaltyPoints: true,
      },
    });

    if (!customer) {
      throw new Error('Customer not found');
    }

    if (customer.loyaltyPoints < points) {
      throw new Error('Insufficient points');
    }

    // Conversion: 100 points = ₹100
    const discountAmount = points;

    await prisma.customers.update({
      where: { id: customerId },
      data: {
        loyaltyPoints: {
          decrement: points,
        },
      },
    });

    // Create redemption record
    await prisma.loyaltyRedemptions.create({
      data: {
        customerId,
        points,
        amount: discountAmount,
      },
    });

    return {
      pointsRedeemed: points,
      discountAmount,
      remainingPoints: customer.loyaltyPoints - points,
    };
  }

  /**
   * Calculate expiry date (1 year)
   */
  private calculateExpiryDate(): Date {
    const expiry = new Date();
    expiry.setFullYear(expiry.getFullYear() + 1);
    return expiry;
  }
}
```

### 7.2 Loyalty Program Rules

```yaml
Points Earning:
  Base Rate: ₹100 = 1 point
  
  Tier Multipliers:
    - Bronze: 1.0x (₹100 = 1 point)
    - Silver: 1.2x (₹100 = 1.2 points)
    - Gold: 1.5x (₹100 = 1.5 points)
    - Platinum: 2.0x (₹100 = 2 points)
  
  Category Bonuses:
    - Premium Collection: +50% points
    - New Arrivals: +25% points
    - Seasonal Wear: +10% points
  
  Special Events:
    - Birthday Month: 2x points
    - Double Points Weekend: 2x points
    - Anniversary: 3x points

Points Redemption:
  Conversion Rate: 100 points = ₹100 discount
  Minimum Redemption: 100 points
  Maximum Per Order: 50% of order value
  
Points Expiry:
  Validity: 1 year from earning date
  Expiry Notification: 30 days before expiry

Tier Qualification:
  Bronze: Default tier
  Silver: ₹25,000 spent in 12 months
  Gold: ₹75,000 spent in 12 months
  Platinum: ₹1,50,000 spent in 12 months

Example Calculation:
  Order Value: ₹5,000
  Customer Tier: Gold (1.5x)
  Category: Premium (+50%)
  Event: Double Points Weekend (2x)
  
  Base Points: 5000 ÷ 100 = 50 points
  With Tier: 50 × 1.5 = 75 points
  With Category: 75 + (75 × 0.5) = 112.5 points
  With Event: 112.5 × 2 = 225 points
  
  Final Points Earned: 225 points
```

---

## 8. Discount Rules

### 8.1 Discount Priority & Stacking

```yaml
Discount Priority (Highest to Lowest):
  1. Clearance/Seasonal Sale Pricing
  2. Customer Segment Discount
  3. Quantity Discount
  4. Coupon Code
  5. Loyalty Points Redemption

Stacking Rules:
  Allowed Combinations:
    ✓ Quantity + Segment Discount
    ✓ Quantity + Coupon
    ✓ Segment + Coupon
    
  Not Allowed:
    ✗ Multiple Coupons
    ✗ Coupon + Sale Price
    ✗ Points + Coupon (must choose one)

Maximum Discount Cap:
  Regular Products: 50% off
  Premium Products: 30% off
  New Arrivals: 20% off

Discount Approval Requirements:
  <10%: Auto-approved
  10-20%: Sales Manager approval
  20-30%: Senior Manager approval
  >30%: Director approval

Example Scenario:
  Product: Premium Shirt
  Price: ₹1,299
  
  Applied Discounts:
  1. Customer: VIP (5%) = ₹64.95
  2. Quantity: 10 items (3%) = ₹38.97
  3. Coupon: SAVE10 (10%) = ₹129.90
  
  Total Discount: ₹233.82
  Final Price: ₹1,065.18 per shirt
```

---

**Document Version:** 2.0  
**Last Updated:** March 26, 2026  
**Next Review:** April 26, 2026  
**Status:** ✅ Production-Ready

---

[← Back to Index](./00_MASTER_INDEX.md)
