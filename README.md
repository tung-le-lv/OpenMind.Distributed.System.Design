# Common distributed system designs pitfalls
## The problem of using synchronously call another service to get data in order to do its job
### Context
When designing a system, we may face the common use case that a service need data from another service to satisfy its job. Here is an example in ecomerce system, where making payment API need customer billing address to perform checkout but this data is owning by another service (Customer). What should we do?

![image](docs/i1.png)

The first obvious option would be: Payment makes a RPC call to Customer service to retrieve the billing address. However, there are so many questions to answer if applying this approach.

1. Wrong domain boundary

Udi Dahan in his course would give: if a service needs to synchronously call another service to get data in order to do its job, that's usually a sign the boundaries are wrong, not a problem to be solved with a better RPC mechanism. So the first move is always to challenge the requirement rather than satisfy it.

A service is a vertical slice that owns *all* the data and logic for a business capability. If Service A constantly needs Service B's data to make a decision, the behavior that needs that data may actually belong inside B, or the two may really be one service. Autonomy is the goal: a service should be able to do its work even when every other service is down. A synchronous data dependency destroys that.

2. Point-to-point couplings/ Temporal coupling

Temporal coupling is high. When the Payment makes a call to Customer, the Customer microservice needs to be reachable for the call to work. If the Customer microservice is unavailable, the call will fail.

Synchronous microservices rely on other services to help them perform their business tasks. Those services, in turn, have their own dependent services, which have their own dependent services.

This can lead to excessive fan-out and difficulty in tracing which services are responsible for fulfilling specific parts of the business logic.

3. Dependent scaling
One of the most important attributute of microservice is the ability to scale independently. Using synchronous RPC, if 1000 requests reaches Payment services, there are 1000 immediate requests accordingly to the Customer service.

The ability to scale up your own service depends on the ability of all dependent services to scale up as well and is directly related to the degree of communications fan-out.

4. Service failure handling

If a dependent service is down, then decisions must be made about how to handle the exception.

Deciding how to handle the outages, when to retry, when to fail, and how to recover to ensure data consistency becomes increasingly difficult the more services there are within the ecosystem.

5. Distributed monoliths

Services may be composed such that they act as a distributed monolith, with many intertwining calls being made between them.

This situation often arises when a team is decomposing a monolith and decides to use synchronous point-to-point calls to mimic the existing boundaries within their monolith.

6. Violate AP in CAP theorem
Regardung CAP, Chris Richardson mentiosn in this book "Microservies Patterns" that a system can only have two of the following three properties: consistency, availability, and partition tolerance. Today, architects prefer to have a system that’s available rather than one that’s consistent.

When making Payment, it requires Customer service to be available, which reduces the availability of the API. If Customer service is down, the API returns a failed status code.

If you want to maximize availability, you must minimize the amount of synchronous communication.

### Solution
To solve this problem, we need to answer the question: why the payment service doesn't own the billing address at the beggining?

Perhalf human mental model may think that billing address is something belong to customer in nature. We may create a single model Customer thats contains all information belong to them such as name, email, phone, billing address, customer shopping preferences, etc.

But in Domain Driven Design, Eric Evan emphasize that creating a big model to satify every context is an design smell. That is why he invented the term Bounded Context and Ubiquitous Language.

In the context of Customer management service, a person is a customer, but the same person in the context of shopping, it is a Buyer. In the context of Payment, it is a Payer, and even in context of shippipng, it is a recepient. We should accept that the same entity, in diferrent context, they have diffent meaning, they own diffent data and operations.
For example, the shopping context, we only care about customer shopping preferences. In payment context, we care about customer billing address or payment card details. Microsoft even had this artile (link https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture#asynchronous-microservice-integration-enforces-microservices-autonomy) about this topic.

So, in general, instead of creating a single model Customer (we often call it god class) like this:

public class Customer{
    public GUID Id;
    public string Name;
    public string Email:
    public BillingAddress;
    public PaymentCards;
    public ShippingAddress;
    public ShoppingPreferences preferences;

    // public methods
    public CreateContact();
    public AddBillingAddress();
    public AddPaymentCard();
    public AddShippingInfo();
}

we should break this model based on the context:
public class Customer{

}

public class Buyer{

}

public class Payer{

}

public class Recepient{

}

Each model should lives in its own bounded contexts and microsercices.

How to create these models? 
UI can call each API in each context to create them or laverage an API gateway.

![image](docs/i2.png)

Now we can see Payment now own the BillingAddress, no synchonous RPC call required anymore.

This the cleanest way to resolve the problem of synchonous RPC that gets data from other services.

There are some cases that we genuinly need the billing address, but the effort of re-factor our system is expensive, we can do a technique name "Local data projection". Here when creating a customer in customer sergice, payment service subscribes to the event CustomerCreated, extract billing address or any data that payment needs and save to a read model table name Payer. With this appoarch, payment can still operate even the case of customer service is in downtime. Udi Dahan mentions this apporach in his course about distributed system design.

One issue with the local projection is the data may be staled at the time of making payment. What if the payment card is updated in customer service while making a payment in payment service, we may need another technique, introducing by Marin Fowler, call "Separated Interface", in the book "Patterns of Enterprise Application Architecture". 
With this approach, the service that need the data (Payment) declard an interface (INeedPaymentCardInfo) and export it as an package (nuget packet in .net or package in java), the service that owns the data (Customer) implement this interface (CustomerPaymentInfoProvider) to retrive the data and export it as a package, Payment then import this package. This resolve the issue of staled data. It also revolve the issues of it needs Customer service to be available at all time, now it just need its database available, which is more stable. 

