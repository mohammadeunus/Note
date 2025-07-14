---
title: "SOLID Principles: The Core of My Most Reliable Code"
description: "How applying SOLID principles transformed my code quality and reliability, with real-world project examples."
excerpt: "Discover how SOLID principles became the foundation of my most reliable software, with practical examples from real projects."
date: 2025-07-15T00:00:00+06:00
lastmod: 2025-07-15T00:00:00+06:00
draft: false
images: []
categories: ["Development", "Best Practices", "SOLID"]
tags: ["SOLID", "Clean Code", "OOP", "Best Practices"]
contributors: ["Your Name"]
pinned: false
homepage: false
---

Over the years of working with Domain-Driven Design (DDD) in C#, I've found that combining it with SOLID principles is essential to building systems that are not only clean and understandable but also maintainable over time.

This post is based on real scenarios I’ve encountered. Each principle comes with either a clean sample or a problematic pattern I’ve seen, plus how to correct it.

---

## Single Responsibility Principle (SRP)

One of the clearest signs of an SRP violation in real-world code is when a class constructor takes **too many dependencies**. A common rule of thumb:

> *"If a class has more than 5–6 constructor parameters, it's probably doing too much."*

I saw this firsthand when I found an application service like this:

```csharp
public class TenantSubscriptionAppService(
    EditionManager editionManager,
    IOptions<AbpDbConnectionOptions> dbConnectionOptions,
    IFeatureManager featureManager,
    IDistributedEventBus DistributedEventBus,
    ITenantManager tenantManager,
    ITenantRepository TenantRepository,
    ISettingManager settingManager,
    IEditionRepository EditionRepository,
    ISubscriptionManager subscriptionManager,
    IPlanRepository PlanRepository,
    IStripePaymentService stripePaymentService,
    ICostCalculationService costCalculationService,
    IRepository<Subscription, Guid> Repository)
    : ApplicationService, ITenantSubscriptionAppService
```

That’s over a dozen injected services. It clearly indicates the class is handling **multiple workflows**—tenant provisioning, feature management, payment setup, event dispatching, etc. Any change in one area risks affecting the entire service.

This violates **SRP**, because the class has **many reasons to change**. It also:

* Makes the code **hard to test** (lots of mocks/fakes required)
* **Increases cognitive load** for developers reading or modifying the code
* Discourages clean separation of business concerns

---

### ✅ After Refactoring

I refactored the class to this:

```csharp
public class TenantSubscriptionAppService(
    ITenantRepository tenantRepository,
    ISettingManager settingManager,
    IPaymentPlanService paymentPlanService,
    ICostCalculationService costCalculationService,
    IRepository<SubscriptionRequest, Guid> subscriptionRequestRepository)
    : ApplicationService, ITenantSubscriptionAppService
```

The refactored version delegates responsibility to **focused domain services** like `IPaymentPlanService`, which internally handles Stripe, editions, plans, etc. Now:

* The app service only orchestrates **tenant subscription logic**
* Each injected service has a **single, narrow responsibility**
* **Future changes** to plan or payment logic don’t affect the app service

This results in a **cleaner, testable, and more maintainable** codebase that aligns with SRP.

---

## Open/Closed Principle (OCP)

👉 *[Content coming soon]*

---

## Liskov Substitution Principle (LSP)

👉 *[Content coming soon]*

---

## Interface Segregation Principle (ISP)

One common pattern I’ve come across (even from senior developers) is the misuse of **overloaded, do-everything methods** with too many optional or nullable parameters. Here's how the same overloaded method is used in two different real-world scenarios in my system:

### Why is This Bad? (ISP Violation)

- **Clients are forced to know about everything:**
  If you just want to make a simple payment, you still have to know about insurance, vouchers, and more.
- **It’s easy to make mistakes:**
  You might pass the wrong value, or forget which parameters matter for your use case.
- **It’s hard to read and maintain:**
  Future developers (maybe even you!) will have a hard time figuring out what’s required for each scenario.

This is a classic violation of the Interface Segregation Principle.
You’re forced to pass a bunch of null and false values for things you don’t care about. This is confusing and error-prone.

```csharp
// Use case 1: Simple contract payment
var payment = await paymentManger.CreateAsync(
    bookingContract,
    bookingContract.Id,
    guest.Id,
    null,
    Clock.Now,
    roomBooking.PaidAmount,
    PaymentMethod.Cash,
    CardType.Visa,
    string.Empty,
    string.Empty,
    false,
    null,
    null,
    null,
    null,
    null,
    null);

// Use case 2: Insurance, voucher, and invoice payment
var payment = await paymentManger.CreateAsync(
    input.ContractId.HasValue ? contract : null,
    input.ContractId,
    input.ReceivedFromId,
    input.ItemId,
    Clock.Now,
    paymentAmount,
    paymentMethod,
    input.CardType,
    referenceNumber,
    note,
    isInsurancePayment,
    paymentAccountId,
    input.InsuranceAmount,
    input.InsurancePaymentMethod,
    input.InsuranceReferenceNumber,
    input.InsurancePaymentAccountId,
    input.InsuranceNote,
    input.ItemTypeName,
    input.VoucherType,
    input.VoucherId,
    input.InvoiceId
);
```

> That’s over 20 parameters! Some are only used for insurance payments, some for vouchers, some for regular payments. Most of the time, you’ll be passing null or default values for things you don’t care about.

### How to Fix It (Applying ISP)

**Step 1: Split the Interface**
Instead of one big method, create smaller, focused interfaces for each payment scenario:

```csharp
public interface IPaymentService
{
    Task<Payment> CreateCashAsync(Booking booking, Money amount, DateTimeOffset paidOn);
    Task<Payment> CreateCardAsync(Booking booking, Money amount, DateTimeOffset paidOn, CardInfo card);
    // ...other scenario-specific methods
}
```

**Step 2: Use Scenario-Specific Input Objects**

Now, when you want to make a regular payment, you only provide what’s needed:

```csharp
var payment = await paymentService.CreateCashAsync(booking, amount, paidOn);
```

No more null or irrelevant parameters!

#### DDD (Domain-Driven Design) Issues
This “fat” service also violates DDD best practices:
- It mixes different business processes in one method.
- It doesn’t use expressive domain models or aggregates.

But the main focus here is: **Don’t make your interfaces “fat.” Keep them focused and easy to use!**

---

## Dependency Inversion Principle (DIP)

In ABP modular monolith architecture, one rule is clear: **Core modules must never reference their child modules.** The Core should be reusable and independent.

But what if the Core needs to trigger some logic or get data that only exists in a child module?

I’ve faced this exact case. My solution was simple and correct:

* I created an **interface in the Core module**.
* Then I **implemented it inside the child module**.

Here’s an example:

```csharp
// Defined in Core
public interface IFeatureXValidator
{
    Task<bool> IsValidAsync(Guid entityId);
}
```

```csharp
// Implemented in child module
public class FeatureXValidator : IFeatureXValidator
{
    public Task<bool> IsValidAsync(Guid entityId)
    {
        // child-specific logic here
    }
}
```

The Core module only depends on the **abstraction**, not the implementation. The actual implementation is wired up using **dependency injection** at runtime.

This is a textbook application of the **Dependency Inversion Principle**:

> High-level modules (Core) should not depend on low-level modules (children). Both should depend on abstractions.

### ✅ Why this is the right approach

* 🔄 **No circular dependency** between modules
* 🧩 **Decoupled and testable**: Core can be tested with a fake implementation
* 📦 **Modular**: Feature modules can be replaced or restructured without touching the Core

By introducing the interface in the Core, and having child modules implement them, you enforce clean boundaries while still allowing extensibility. This makes your application architecture far more **maintainable** and **future-proof**.

> 💡 If a Core module needs to interact with child logic — use an interface in the Core, and let the child module plug into it. 