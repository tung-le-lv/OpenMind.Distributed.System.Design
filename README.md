## Table of Contents

- [Synchronously Calling Another Service to Get Data](#synchronously-calling-another-service-to-get-data)
  - [Context](#context)
  - [Solution](#solution)
  - [Alternative Techniques](#alternative-techniques)
    - [Local Data Projection](#local-data-projection)
    - [Separated Interface](#separated-interface)
- [When to Use Synchronous Calls Between Microservices](#when-to-use-synchronous-calls-between-microservices)
  - [Platform Capability Services](#platform-capability-services)
  - [Microservices Within the Same Bounded Context](#microservices-within-the-same-bounded-context)
  - [Saga / Process Manager](#saga--process-manager)

---

## Synchronously Calling Another Service to Get Data

### Context

A common design challenge is when a service needs data owned by another service to do its job. Consider an e-commerce example: the Payment service needs a customer's billing address to complete checkout, but that data is owned by the Customer service. What should we do?

![image](docs/context.png)

The first obvious solution is to have Payment make a synchronous RPC call to Customer to retrieve the billing address. However, this approach raises a number of concerns.

#### 1. Wrong Domain Boundary

As Udi Dahan puts it in his [distributed systems design course](https://particular.net/adsd):

> If a service needs to synchronously call another service to get data in order to do its job, that's usually a sign the boundaries are wrong — not a problem to be solved with a better RPC mechanism.

The first move should always be to challenge the requirement, not to satisfy it.

A service is a vertical slice that owns *all* the data and logic for a business capability. If Service A constantly needs Service B's data to make a decision, the behavior requiring that data may actually belong inside B — or the two may really be one service. The goal is autonomy: a service should be able to do its work even when every other service is down. A synchronous data dependency destroys that.

#### 2. Temporal Coupling

When Payment calls Customer, Customer must be reachable for the call to succeed. If Customer is unavailable, the call fails.

Synchronous services rely on other services to perform their work. Those services, in turn, have their own dependencies, which have their own dependencies. This makes it difficult to trace which service is responsible for each piece of business logic.

#### 3. Dependent Scaling

One of the most important attributes of a microservice is the ability to scale independently. With synchronous RPC, if there are 1,000 requests to the Payment service, it generates 1,000 immediate requests to the Customer service, making it scaling dependent.

#### 4. Distributed Monolith Anti-Pattern

This pattern commonly emerges when teams decompose a monolith and use synchronous point-to-point calls to mirror the original internal boundaries.

In a monolith system, there is only one database — when Payment needs a billing address, it queries the Customer table directly. When teams decompose a monolith into microservices, they carry that same mental model: "I need data from Customer, so I'll just call it." A synchronous RPC call feels identical to a SQL query but this is not a microservice architecture at all.

#### 5. Violate AP preference in CAP Theorem

As Chris Richardson notes in *Microservices Patterns*, a system can only guarantee two of three properties: consistency, availability, and partition tolerance. In distrubited system, partition tolerance is unavoidable, so the real choice is between availability and consistency. Most architects today favor availability over consistency.

When Payment requires Customer to be available, it reduces Payment's own availability. If Customer is down, the Payment API fails too.

> If you want to maximize availability, you must minimize synchronous communication.

---

### Solution

To solve this, we need to ask: **why doesn't the Payment service own the billing address in the first place?**

It's natural to think of billing address as belonging to a "Customer." This leads to creating a single, all-encompassing Customer model:

```csharp
public class Customer {
    public Guid Id;
    public string Name;
    public string Email;
    public BillingAddress BillingAddress;
    public PaymentCards PaymentCards;
    public ShippingAddress ShippingAddress;
    public ShoppingPreferences Preferences;

    public void CreateContact() { }
    public void AddBillingAddress() { }
    public void AddPaymentCard() { }
    public void AddShippingInfo() { }
}
```

But in Domain-Driven Design, Eric Evans argues that creating one large model to satisfy every context is a design smell — which is why he introduced the concepts of **Bounded Context** and **Ubiquitous Language**. For more about DDD, checkout my article and code example at https://github.com/tung-le-lv/OpenMind.DDD.Patterns.

The same person means different things in different contexts:
- In Customer Management: they are a **Customer**
- In Shopping: they are a **Buyer**
- In Payment: they are a **Payer**
- In Shipping: they are a **Recipient**

Each context owns different data and exposes different operations.

Instead of the god class above, model each context separately:

```csharp
// Customer context — identity and contact info only
public class Customer {
    public Guid Id;
    public string Name;
    public string Email;
    public string Phone;

    public void AddContactInfo();
}

// Shopping context — purchasing relating only
public class Buyer {
    public Id;
    public Identity CustomerId;
    public ShoppingPreferences Preferences;

    public void AddShoppingPreferences();
}

// Payment context — billing and payment details only
public class Payer {
    public Guid Id;
    public Identity CustomerId;
    public BillingAddress BillingAddress;
    public List<PaymentCard> PaymentCards;

    public void AddPaymentCards();
}

// Shipping context — delivery details only
public class Recipient {
    public Id;
    public Identity CustomerId;
    public ShippingAddress ShippingAddress;

    public void AddShippingInfo();
}
```

Each model lives in its own bounded context and microservice.

**How data are created for each model from the UI perspective?**  
The UI can call each service's API directly, or route through an API gateway.

![image](docs/solution.png)

With this design, Payment owns the `BillingAddress` — no synchronous RPC call required. This is the cleanest solution to the problem.  

> **Side Note**  
> **Use UI composition for display needs**.  
> *A large fraction of "I need data from another service" is really "I need to show data from several services on one screen." Udi's answer is to compose at the presentation layer, not the backend. Each service contributes its own UI widget/fragment backed by its own data, and they're stitched together in the composite UI. This avoids the backend "god service" that aggregates by calling everyone — which is the anti-pattern that produces a distributed monolith.*  
> 
>*For this particular case, we often create an endpoint in api-gateway named GetCustomerDetails that performs RPC calls to each service in each bounded context to retrieve all the data needed for the UI "customer detail page".*
>
>*For a deep dive into UI data displaying patterns, see chapter 7 of Microservices Patterns by Chris Richardson: Implementing queries in a microservice architecture.*  

### Alternative Techniques

When proper bounded context models are not in place and a full refactor is too costly, the following techniques can help:

#### Local Data Projection

If restructuring the system is too expensive, consider **Local Data Projection**. When a customer is created in the Customer service, the Payment service subscribes to the `CustomerCreated` event, extracts the billing address (and any other data it needs), and saves it to a local read model — `PayerReadModel` table.

With this approach, Payment can still operate even when the Customer service is down. Udi Dahan discusses this technique in his distributed systems course.

Even Microsoft's architecture documentation covers this in the article: [Asynchronous microservice integration enforces microservice autonomy](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture#asynchronous-microservice-integration-enforces-microservices-autonomy). This is a citing from the article:  
> *If possible, never depend on synchronous communication (request/response) between multiple microservices, not even for queries. The goal of each microservice is to be autonomous and available to the client consumer.*
>
> *If a microservice needs data that's originally owned by other microservices, do not rely on making synchronous requests for that data. Instead, replicate or propagate that data (only the attributes you need) into the initial service's database by using eventual consistency.*
>
> *Duplicating some data across several microservices isn't an incorrect design—on the contrary, when doing that you can translate the data into the specific language or terms of that additional domain or Bounded Context. For instance, in the eShopOnContainers application you have a microservice named identity-api that's in charge of most of the user's data with an entity named User. However, when you need to store data about the user within the Ordering microservice, you store it as a different entity named Buyer. The Buyer entity shares the same identity with the original User entity, but it might have only the few attributes needed by the Ordering domain, and not the whole user profile.*  


#### Separated Interface

One issue with local projection is that the read model data may be stale at the time of payment. For example, what if a customer updates their payment card in the Customer service while making payment in Payment service?

Martin Fowler's **Separated Interface** pattern (from *Patterns of Enterprise Application Architecture*) addresses this. The service that needs the data (Payment) declares an interface:

```csharp
public interface INeedPaymentCardInfo {
    PaymentCardInfo GetPaymentCardInfo(Guid customerId);
}
```

This interface is published as a package (NuGet in .NET, a JAR in Java). The service that owns the data (Customer) implements the interface:

```csharp
public class CustomerPaymentInfoProvider : INeedPaymentCardInfo {
    public PaymentCardInfo GetPaymentCardInfo(Guid customerId) { ... }
}
```

Customer publishes its implementation as a separate package, which Payment imports and uses.

This is actually the pattern Customer-Supplier in Context Mapping introduced by Eric Evan is his DDD blue book.

We employ this technique when strong data consistency is required.  

This pattern also means Payment requires the Customer **database** to be available — not the Customer service itself — which is more reliable. Payment does not directly query Customer database but via a contract that is implemented by Customer service.

## When to Use Synchronous Calls Between Microservices

### Platform Capability Services

Platform capability services do not belong to any business bounded context. Common examples include authentication, authorization, ai-service, etc. Any microservice in any bounded context just call them synchronously.

### Microservices Within the Same Bounded Context

Microservices within the same bounded context are allowed to call each other synchronously. For example, `customer-service` can make an RPC call to `customer-achievement-service`, or `payment-service` can call `payment-gateway-service` synchronously to perform its business operations.

> One bounded context typically maps to a single microservice, but it can be split into smaller ones over time. Common decomposition drivers include:
> - Service scope and responsibility
> - Scalability and throughput requirements
> - Code volatility
> - Fault tolerance
> - Security boundaries
> - Extensibility
>
> Microservices produced by this decomposition remain within the same bounded context and may still communicate synchronously. Chapter 7 of *Software Architecture: The Hard Parts* covers these drivers in detail.

### Saga / Process Manager

To handle a business capability that spans multiple bounded contexts, use the **Saga** pattern or a **Process Manager**. A Saga orchestrator can perform a business operation by making synchronous calls to microservices across bounded contexts in a defined sequence.

For example, consider the requirement "unsubscribe customer from system". This must delete all customer-related data across the system, spanning multiple microservices and bounded contexts.

To handle this, we can create a Saga orchestrator named CustomerUnsubscriptionSaga that perform synchromous calls to each relevant microservice in different bounded contexts to remove the customer's data.
