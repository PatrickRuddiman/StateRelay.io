# 12 - Billing & Stripe Integration

**Operated by**: Pinnacle Ridge Cloud Solutions LLC

## Subscription Tiers

| Tier | Price | Devices | Features |
|------|-------|---------|----------|
| **Free** | $0/mo | 2 | Basic Relay Devices |
| **Pro** | $4.99/mo | 20 | All Relay Device types, email support |
| **Business** | $19.99/mo | Unlimited | API access, priority support |

## Stripe Products Setup

Create in Stripe Dashboard:

```bash
# Create products via Stripe CLI
stripe products create --name="StateRelay Pro" --description="20 Relay Devices"
stripe prices create --product=prod_xxx --unit-amount=499 --currency=usd --recurring[interval]=month

stripe products create --name="StateRelay Business" --description="Unlimited Relay Devices"
stripe prices create --product=prod_yyy --unit-amount=1999 --currency=usd --recurring[interval]=month
```

## Checkout Flow

```typescript
// POST /api/v1/subscriptions/checkout
export async function createCheckout(req: Request, res: Response) {
  const { priceId } = req.body;
  
  // Get or create Stripe customer
  let customerId = req.user.stripeCustomerId;
  if (!customerId) {
    const customer = await stripe.customers.create({
      email: req.user.email,
      metadata: { userId: req.user.id },
    });
    customerId = customer.id;
    await prisma.subscription.update({
      where: { userId: req.user.id },
      data: { stripeCustomerId: customerId },
    });
  }
  
  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    mode: 'subscription',
    line_items: [{ price: priceId, quantity: 1 }],
    success_url: `https://app.staterelay.io/billing?success=true`,
    cancel_url: `https://app.staterelay.io/billing?canceled=true`,
  });
  
  res.json({ url: session.url });
}
```

## Webhook Handler

```typescript
// POST /api/v1/stripe/webhook
export async function stripeWebhook(req: Request, res: Response) {
  const sig = req.headers['stripe-signature'];
  const event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET);
  
  switch (event.type) {
    case 'checkout.session.completed':
      await handleCheckoutComplete(event.data.object);
      break;
    case 'customer.subscription.updated':
      await handleSubscriptionUpdate(event.data.object);
      break;
    case 'customer.subscription.deleted':
      await handleSubscriptionCanceled(event.data.object);
      break;
    case 'invoice.payment_failed':
      await handlePaymentFailed(event.data.object);
      break;
  }
  
  res.json({ received: true });
}

async function handleCheckoutComplete(session: Stripe.Checkout.Session) {
  const subscription = await stripe.subscriptions.retrieve(session.subscription as string);
  const priceId = subscription.items.data[0].price.id;
  
  const tier = getTierFromPriceId(priceId);
  const maxDevices = tier === 'pro' ? 20 : 999999;
  
  await prisma.subscription.update({
    where: { stripeCustomerId: session.customer as string },
    data: {
      stripeSubscriptionId: subscription.id,
      tier,
      status: 'active',
      maxDevices,
      currentPeriodStart: new Date(subscription.current_period_start * 1000),
      currentPeriodEnd: new Date(subscription.current_period_end * 1000),
    },
  });
}

async function handleSubscriptionCanceled(subscription: Stripe.Subscription) {
  await prisma.subscription.update({
    where: { stripeSubscriptionId: subscription.id },
    data: {
      tier: 'free',
      status: 'canceled',
      maxDevices: 2,
      canceledAt: new Date(),
    },
  });
}
```

## Customer Portal

```typescript
// POST /api/v1/subscriptions/portal
export async function createPortalSession(req: Request, res: Response) {
  const sub = await prisma.subscription.findUnique({
    where: { userId: req.user.id },
  });
  
  const session = await stripe.billingPortal.sessions.create({
    customer: sub.stripeCustomerId,
    return_url: `https://app.staterelay.io/billing`,
  });
  
  res.json({ url: session.url });
}
```

## Device Limit Enforcement

```typescript
export async function canCreateDevice(userId: string): Promise<boolean> {
  const [subscription, deviceCount] = await Promise.all([
    prisma.subscription.findUnique({ where: { userId } }),
    prisma.device.count({ where: { userId } }),
  ]);
  
  const maxDevices = subscription?.maxDevices ?? 2;
  return deviceCount < maxDevices;
}
```

## Environment Variables

```bash
STRIPE_SECRET_KEY=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
STRIPE_PRO_PRICE_ID=price_xxx
STRIPE_BUSINESS_PRICE_ID=price_yyy
