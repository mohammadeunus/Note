---
title: "Building a Custom Subscription System with ABP Framework and Stripe: A Developer's Journey 🚀"
description: "A deep dive into building a flexible, dynamic subscription system using ABP Framework and Stripe, including architecture, code, and lessons learned."
excerpt: "How I built a custom, dynamic SaaS subscription system with ABP Framework and Stripe."
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
weight: 50
images: []
categories: ["Development", "SaaS", "ABP Framework"]
tags: ["Stripe", "ABP Framework", "Custom Plans", "SaaS", "Payments"]
contributors: ["Henk Verlinde"]
pinned: false
homepage: false
---

# Building a Custom Subscription System with ABP Framework and Stripe: A Developer's Journey 🚀

Finally got a chance to integrate Stripe into a system! I have been longing to fill up this empty section of my knowledge for a long time. When my client approached me with a unique requirement - they wanted to offer custom subscription plans instead of the standard "one-size-fits-all" packages - I knew this was the perfect opportunity to dive deep into ABP Framework's payment system and create something truly flexible.

## The Challenge: Custom Subscription Plans 🎯

Traditional SaaS platforms offer predefined packages like "Standard," "Professional," and "Enterprise." But my client wanted something different - a system where customers could pick and choose exactly what they needed. Think of it like building your own pizza - you get to choose every single topping, and you only pay for what you want.

Now, here's where things get interesting. ABP Framework already has this amazing payment system built-in, but it's designed for those traditional fixed packages. My client's requirement? They wanted to create plans on-the-fly based on customer selections. That's like trying to fit a square peg in a round hole, but hey, that's what makes programming fun, right? 😄

## The Architecture: How We Made It Work 🏗️

Let me break down what we built here, because this is where the magic happens:

### 1. The Custom Plan Builder UI ✨
First, we created this beautiful interface where users can:
- Select the number of rooms and hotels they need
- Pick specific features they want (like integrations, add-ons, etc.)
- See real-time pricing updates
- Choose between monthly and annual billing

It's like a shopping cart, but for software features. Pretty cool, right?

### 2. The Dynamic Plan Creation Process 🔄
Here's the genius part - instead of having predefined plans, we create them dynamically:

```csharp
// When user submits their custom plan
var planPayload = new
{
    isMonthly = !priceSwitch.prop('checked'),
    roomCount = rooms,
    hotelCount = hotels,
    subscribedFeatures = selectedFeatures,
    saasTenant = {
        name = tenantName,
        adminEmailAddress = adminEmail,
        adminPassword = adminPassword
    }
};

// First, create the plan in our system
await nTouch.pMS.core.subscriptions.tenantSubscription.createPlan(planPayload);

// Then create the subscription and get Stripe session
const result = await nTouch.pMS.core.subscriptions.tenantSubscription.createSubscriptionAndTenant(payload);
```

### 3. The Payment Flow Magic 💫
This is where it gets really interesting. We're essentially doing this:

1. **User customizes their plan** → We calculate the price
2. **Create a plan in Stripe** → Dynamic plan creation
3. **Create a plan in our system** → ABP plan management
4. **Create a subscription request** → Track the payment
5. **Create a tenant with a "blank" edition** → No features yet
6. **Redirect to Stripe** → User pays
7. **Webhook processes payment** → We activate the features

It's like a three-step dance: Plan → Pay → Activate. 💃

## The Technical Deep Dive 🔧

### PrePayment Stage: Setting Up the Dance
```csharp
// In PrePayment.cshtml.cs
StartResult = await PaymentRequestAppService.StartAsync(StripeConsts.GatewayName, new PaymentRequestStartDto
{
    ReturnUrl = PaymentWebOptions.Value.RootUrl.RemovePostFix("/") + 
               "/api/core/tenant-subscription/complete-payment/{CHECKOUT_SESSION_ID}",
    CancelUrl = PaymentWebOptions.Value.RootUrl,
    PaymentRequestId = PaymentRequestId
});
```

### The Webhook: Where the Magic Happens ✨
```csharp
// In StripePaymentGateway.cs
public virtual async Task<bool> CompleteAsync(PaymentRequest paymentRequest, Dictionary<string, string> extraData)
{
    var sessionId = extraData[StripeConsts.ParameterNames.SessionId];
    var sessionService = new SessionService();
    var session = await sessionService.GetAsync(sessionId);
    
    if (session.PaymentStatus == "paid")
    {
        // Now we create the actual edition with the features the user paid for
        await CreateCustomEditionForPaymentRequest(paymentRequest);
        return true;
    }
    
    return false;
}
```

## The Security Considerations 🔒

Now, let me address something important - security. When I first started this project, I was like "Oh, this is going to be a security nightmare." But here's what we did right:

1. **Server-side plan creation** - No sensitive data exposed to client
2. **Webhook verification** - Stripe signs the webhooks, we verify them
3. **Tenant isolation** - Each tenant gets their own isolated environment
4. **Payment verification** - We double-check the payment status before activating features

## The Edge Cases We Handled 🎯

You know what's the most challenging part? Edge cases. Here are some we had to think about:

- **What if payment fails?** → We clean up the created tenant
- **What if webhook fails?** → We have retry mechanisms
- **What if user cancels?** → We handle the cancellation gracefully
- **What if Stripe is down?** → We have fallback mechanisms

## The Performance Optimizations ⚡

Creating plans dynamically sounds expensive, right? Well, we optimized it:

1. **Caching** - We cache frequently requested plan combinations
2. **Async processing** - Webhook processing is completely async
3. **Database optimization** - We use efficient queries for plan creation
4. **Stripe session management** - We handle session expiration properly

## The Result: A Flexible, Scalable System 🎉

What we ended up with is a system that:
- ✅ Creates custom plans on-demand
- ✅ Handles payments securely
- ✅ Scales with your business
- ✅ Provides real-time pricing
- ✅ Integrates seamlessly with ABP Framework

## Lessons Learned 📚

1. **ABP Framework is powerful** - But you need to understand its conventions
2. **Stripe webhooks are reliable** - But always verify them
3. **Dynamic plan creation is complex** - But totally worth it
4. **Security should be built-in** - Not added later

## The Code That Makes It All Work 💻

Here's a snippet of the core logic that ties everything together:

```javascript
// The complete flow in one function
async function handleCustomPlanPayment() {
    try {
        window.LoadingScreen.show();
        
        // Step 1: Create the plan
        const planResult = await nTouch.pMS.core.subscriptions.tenantSubscription.createPlan(planPayload);
        
        // Step 2: Create subscription and get Stripe session
        const subscriptionResult = await nTouch.pMS.core.subscriptions.tenantSubscription.createSubscriptionAndTenant(payload);
        
        // Step 3: Redirect to Stripe
        const stripe = Stripe(publishableKey);
        const { error } = await stripe.redirectToCheckout({
            sessionId: subscriptionResult.sessionId
        });
        
        if (error) {
            console.error('Payment failed:', error);
        }
    } catch (error) {
        console.error('Plan creation failed:', error);
    } finally {
        window.LoadingScreen.hide();
    }
}
```

## Conclusion: Why This Matters 🌟

This isn't just about building a payment system. It's about creating flexibility for your customers. In today's competitive SaaS market, the ability to offer truly custom plans can be a game-changer.

The beauty of this approach is that it leverages ABP Framework's existing infrastructure while adding the flexibility your business needs. You get the best of both worlds - enterprise-grade architecture with startup-level flexibility.

## Need Help Implementing This? 👨‍💻

If you're reading this and thinking "This is exactly what I need, but I don't have the time or expertise to implement it," well, you're in luck! I specialize in ABP Framework integrations and custom payment systems. Whether you need a similar custom subscription system or want to integrate any payment gateway with ABP Framework, I can help you build it right.

The key is understanding both the technical requirements and the business logic. It's not just about writing code - it's about creating a system that grows with your business.

So there you have it - a complete custom subscription system built on ABP Framework with Stripe integration. From dynamic plan creation to secure payment processing, this system handles it all while maintaining the flexibility your customers demand.

Remember, in the world of SaaS, flexibility is king. And with this system, you're not just selling software - you're selling solutions that fit perfectly into your customers' workflows. 👑

---

*P.S. If you found this helpful and want to implement something similar, feel free to reach out! I love helping fellow developers build amazing systems.* 😊
