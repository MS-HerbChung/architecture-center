# Apply tactical DDD patterns

Domain driven design has two distinct phases or aspects: Strategic and tactical. The goal of strategic DDD is to define the large-scale structure of the system. During the strategic phase, you are mapping out the business domain, defining bounded contexts for your domain models, and developing a ubiquitous language. 

Tactical DDD provides a set of design patterns that you can use to create the domain model. The patterns are applied within a single bounded context. In a microservices architecture, we are particularly interested in the entity and aggregate patterns. Applying these patterns will help us to identify natural boundaries for the services in our application. As a general principle, a microservice should be no smaller than an aggregate, and no larger than a bounded context. First, we'll review the tactical patterns, then we'll apply them to the Shipping bounded context in the drone delivery application.

## The patterns

If you are already familiar with DDD, you can skip this section. The patterns are described in more detail in *Domain Driven Design* by Eric Evans (see chapters 5 &ndash; 6), and *Implementing Domain-Driven Design* by Vaughn Vernon. This section is only a summary of the patterns.

**Entities**. An entity is an object with a unique identity that persists over time. For example, in a banking application, customers and accounts would be entities. 

- An entity has a unique identifier in the system, which can be used to look up or retrieve the entity. That doesn't mean the identifier is necessarily exposed to users. It could be a GUID or a primary key in a database. The identifier might be a composite key, especially for child entities.
- The attributes of an entity may change over time. For example, a person's name or address might change, but they are still the same person. 
- An identity may span multiple bounded contexts, and may endure beyond the lifetime of the application. 
 
**Value objects**. A value object has no identity. It is defined only by the values of its attributes. Value objects are also immutable. To update a value object, you always create a new instance to replace the old one. Value objects can have methods that encapsulate domain logic, but those methods should have no side-effects on the object's state. Typical examples of value objects include colors, dates and times, and currency values. 

**Aggregates**. An aggregate defines a consistency boundary around one or more entities. Exactly one entity in an aggegrate is the root. Lookup is done using the root entity's identifier. Any other entities in the aggregate are children of the root, and are referenced by following pointers from the root. 

The purpose of an aggregate is to model transactional invariants. Things in the real world have complex webs of relationships. Customers create orders, orders contain products, products have suppliers, and so on. If the application modifies several related objects, how does it guarantee consistency? How do we keep track of invariants and enforce them?  

Traditional applications have often used database transactions to enforce consistency. In a distributed application, however, that's often not be feasible. A single business transaction may span multiple data stores, or may be long running, or may involve third-party services. Ultimately it's up to the application, not the data layer, to enforce the invariants required for the domain. 

> [!NOTE]
> It's actually common for an aggregate to consist of a single entity, with no child entities. Even so, the distinction between aggregates and entities is still important. An aggregate enforces transactional semantics, while an entity might not.

**Services**. In DDD terminology, a service is an object that implements some logic without holding any state. Evans distinguishes between *domain services*, which encapsulate domain logic, and *application services*, which provide technical functionality, such as sending an SMS message. Domain services are often used to model behavior that spans multiple entities. 

> [!NOTE]
> The term service is rather overloaded. The definition here is not directly related to microservices.
 
There are a few other DDD patterns not listed here, including factories, repositories, and modules, but these are less relevant for our purposes.

## Define entities and aggregates

We start with the scenarios that the Shipping bounded context must handle.

- A business (the sender) can schedule a drone to deliver a package to a customer (the receiver). Alternatively, a user can request a drone to pick up goods from a business that is registered with the drone delivery service. 
- The sender generates a tag (barcode or RFID) to put on the package. 
- A drone will pick up and deliver a package from the source location to the destination location.
- When a user schedules a delivery, the system provides an ETA based on route information, weather conditions, historical data, and so forth. 
- When the drone is in flight, the sender and the receiver can track the current location and the latest ETA. 
- Until a drone has picked up the package, the user can cancel a delivery.
- When the delivery is complete, the sender and the receiver are notified.
- The sender can request delivery confirmation from the user, in the form of a signature or finger print.
- A user can look up the history of a completed delivery.

From these scenarios, the development team identified the following entities:

- Delivery
- Package
- Drone
- UserAccount
- Confirmation
- Notification
- Tag

Of these, Delivery, Package, Drone, and UserAccount are aggregates. Tag, Confirmation, and Notification are associated with Delivery entities. 

Value objects in this design include Location, ETA, PackageWeight, and PackageSize. 

## Define services

What is the right size for a service? A correct but less-than-helpful answer is: Not too big, and not too small. Here is a process to follow.

- Start with a bounded context. 
- Further break down the bounded context according to non-functional requirements. Consider aspects such as team size, data types, technologies used, scalability requirements, availability requirements, and security requirements.
- Next, look at your aggregates. There should be one or more aggregates that have substantial domain logic. (If not, you may need to revisit your domain model.) These aggregates are candidates for services.  Generally, a service should not be smaller than an aggregate or larger than a bounded context.
- Domain services (in the DDD sense) are also good candidates for services.

Here are some reasons to keep things within the same service: 

- Communication overhead. If splitting functionality into two services causes them to be overly chatty, it may be a symptom that these functions belong in the same service. 
- Data consistency. Services maintain their own data stores, and sometimes it's important to maintain data consistency by putting functionality into a single service. That said, consider whether you really need strong consistency. Patterns such as CQRS and Event Sourcing can help you to manage eventual consistency. See CQRS in microservices.

Here are some reasons to break up a service:

- To keep team sizes small. A team for a single service should probably not be more  than a dozen people (the "two-pizza rule").
- To enable faster release velocity.
- To limit dependencies.
- To use different technologies or data stores.

The development team settled on the following services:

- Delivery Scheduler. Receives client requests to schedule a delivery, and manages the workflow.
- Package. Stores information about packages.
- Delivery. Manages deliveries and tracks delivery status. This service generates status update events for each delivery.
- Delivery History. Records the history of all deliveries.
 
 ![](./images/services.svg)

The Delivery Scheduler service also depends on two services from other bounded contexts, namely Accounts and Drone Management. 

## Map tactical patterns to REST APIs

Patterns like entity, aggregate, and value object are designed to place certain constraints on the objects in your domain model. For example, value objects are meant to be immutable. In many discussions of DDD, the patterns are modeled using OO language concepts like constructors or property getters and setters. 

For example, here is a TypeScript implementation of a value object. The properties are declared to be read-only, so the only way to modify a Location is to create a new one. The properties are validated when the object is created.

```ts
export class Location {
    readonly latitude: number;
    readonly longitude: number;
    readonly altitude: number;

    constructor(latitude: number, longitude: number, altitude: number) {
        if (latitude < -90 || latitude > 90) {
            throw new RangeError('latitude must be between -90 and 90');
        }
        if (longitude < -180 || longitude > 180) {
            throw new RangeError('longitude must be between -180 and 180');
        }
        this.latitude = latitude;
        this.longitude = longitude;
        this.altitude = altitude;
    }
}
```

Coding practices like this are particularly important in a more monolithic application. In a large code base, many subsystems might use the `Location` object, so it's important to enforce the correct behaviors of objects. Patterns such as Repository ensure that other parts of the code cannot make arbitrary writes to the data store.

![](./images/repository.svg)

But in a microservices architecture, services don't share a code base. Instead, they communicate through APIs. Consider the case where the Delivery Scheduler service requests information about a drone from the Drone Management service. The Drone Management service will have an internal model of a drone, which is represented in code. But the Delivery Scheduler doesn't see that. Instead, what it sees is a wire representation of the entity &mdash; for example, JSON in an HTTP response body.

![](./images/ddd-rest.svg)

As a result, code has a smaller surface area. If the Drone Management service defines a Location class, the scope of that class is limited to the service, making its easier to validate the correct usage. If another service needs to update the drone location, it has to go through the Drone Management service API.

It turns out that RESTful APIs can model many of the tactical DDD concepts.

- Business logic is encapsulated in the API, so the internal state of an aggregate is always consistent. Don't expose APIs that allow clients to manipulate internal state in an inconsistent way. Favor coarse-grained APIs that expose aggregates as resources.

- Aggregates are addressable by ID. The URL is the stable identifier.

    ```    
    /api/orders/{id}
    ```
    
- Child entities can be navigated from the root, or via links in the representation (following HATEOS principles)

    ```    
    /api/orders/{id}/items/{order_item_id}
    ```    

- Value objects are updated by replacing the entire value through a PUT or PATCH request.

    ```    
    PUT /api/customer/{id}/address
    ```    

- A collection can act like a repository.

    ```    
    /api/users/{id}/orders/
    ```
