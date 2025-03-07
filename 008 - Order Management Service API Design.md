# Order Management Service API Design

## Table of Contents
- [API Endpoints and Routes](#api-endpoints-and-routes)
- [Data Models](#data-models)
- [Request and Response Schemas](#request-and-response-schemas)
- [Implementation Considerations (Java Spring Boot)](#implementation-considerations-java-spring-boot)
- [Pagination, Filtering, and Sorting](#pagination-filtering-and-sorting)
- [Security Considerations](#security-considerations)
- [Best Practices](#best-practices)
- [Interview Discussion](#interview-discussion)

## API Endpoints and Routes

The API is designed around an Order resource, following RESTful conventions with plural nouns for resources and standard HTTP methods. Key endpoints include:

- **POST /orders** – Create a new order.  
  - The client provides order details in the request body.  
  - The server creates an order resource.  
  - On success, returns HTTP 201 Created with the new order's data.

- **GET /orders** – Retrieve a list of orders.  
  - Supports query parameters for pagination and filtering (e.g., `?status=SHIPPED&customerId=123`, `?page=2&size=20`).  
  - Returns a paginated list of orders, possibly in a JSON array or an object containing both data and pagination metadata.

- **GET /orders/{orderId}** – Retrieve a specific order by ID.  
  - Returns the order resource in JSON if found, or 404 Not Found if the ID doesn’t exist.

- **PUT /orders/{orderId}** – Update an existing order fully.  
  - The client sends the entire order representation.  
  - On success, returns 200 OK (or 204 No Content).  
  - If the order is not found, return 404; for validation errors, 400 Bad Request.

- **DELETE /orders/{orderId}** – Cancel an order.  
  - Marks the order as canceled (or deletes it).  
  - Returns 204 No Content on success.  
  - If the order cannot be canceled due to its state, return 400 or 409, and 404 if the ID doesn’t exist.

Each endpoint uses HTTP verbs consistently (GET for retrieval, POST for creation, PUT for update, DELETE for removal).

## Data Models

Core entities in the Order Management system include:

- **Order**: Represents a customer order. Key attributes:
  - `id` (string/UUID or long) – Unique identifier for the order.
  - `customerId` (string or long) – ID of the customer who placed the order.
  - `items` (list of `OrderItem`) – Line items in the order.
  - `totalPrice` (decimal) – Total cost of the order.
  - `status` (string/enum) – Current status (e.g. "NEW", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED").
  - `createdAt` (timestamp) – Time when the order was created (ISO 8601).

- **OrderItem**: Represents an individual item in an order.
  - `orderItemId` (string/long) – Unique ID for the line item (or composite key with order).
  - `productId` (string/long) – Identifier of the product.
  - `quantity` (int) – Quantity of the product ordered (must be >= 1).
  - `price` (decimal) – Unit price of the product at the time of order.

- **Customer** (optional reference): Represents a customer in the system. In the Orders service, only `customerId` is stored per order; details would be retrieved from a separate Customer service if needed.

In a relational schema, `OrderItem` would typically have a foreign key linking to its parent `Order`. The `Customer` entity is mentioned for clarity, but the Order service might only store the customer reference rather than full customer data.

## Request and Response Schemas

Requests and responses use JSON. Examples:

### Create Order – POST /orders

Request body example:

```json
{
  "customerId": 123,
  "items": [
    { "productId": 456, "quantity": 2 },
    { "productId": 789, "quantity": 1 }
  ]
}
```

Response example (on success):

```json
{
  "id": 101,
  "customerId": 123,
  "items": [
    { "orderItemId": 1, "productId": 456, "quantity": 2, "price": 19.99 },
    { "orderItemId": 2, "productId": 789, "quantity": 1, "price": 5.49 }
  ],
  "totalPrice": 45.47,
  "status": "NEW",
  "createdAt": "2025-03-07T15:30:00Z"
}
```

HTTP status 201 is returned. The response includes all key fields, with `totalPrice` and `status` set by the server.

### List Orders – GET /orders

Paginated response example:

```json
{
  "data": [
    {
      "id": 100,
      "customerId": 50,
      "totalPrice": 99.90,
      "status": "SHIPPED",
      "createdAt": "2025-03-01T10:00:00Z"
    },
    {
      "id": 101,
      "customerId": 123,
      "totalPrice": 45.47,
      "status": "NEW",
      "createdAt": "2025-03-07T15:30:00Z"
    }
  ],
  "page": 1,
  "size": 2,
  "totalElements": 50,
  "totalPages": 25
}
```

### Get Order by ID – GET /orders/{orderId}

Example response (if found):

```json
{
  "id": 101,
  "customerId": 123,
  "items": [
    ...
  ],
  "totalPrice": 45.47,
  "status": "NEW",
  "createdAt": "2025-03-07T15:30:00Z"
}
```

Status 200 OK is returned, or 404 if the order is not found.

### Update Order – PUT /orders/{orderId}

Example request body:

```json
{
  "customerId": 123,
  "items": [
    { "productId": 456, "quantity": 3 },
    { "productId": 789, "quantity": 1 }
  ],
  "status": "PROCESSING"
}
```

The server responds with 200 OK and updated JSON, recalculating `totalPrice`. If an order cannot be updated (e.g., already shipped), return 400 or 409.

### Cancel Order – DELETE /orders/{orderId}

No request body. Returns 204 No Content if successful. For invalid states, return 400 or 409. For nonexistent IDs, 404.

All timestamps use ISO 8601 format. Errors return a JSON object like `{"error":"Something went wrong"}`.

## Implementation Considerations (Java Spring Boot)

A clean layered architecture helps maintainability:

- **Controllers**: Classes annotated with `@RestController` handle routes. They parse JSON input, validate requests, and return the correct HTTP responses. Controllers delegate to a service layer for business logic.
- **Services**: Classes annotated with `@Service` contain business workflows. For example, `OrderService` handles `createOrder`, `getOrders` (with filters), `getOrderById`, etc. It enforces rules such as calculating `totalPrice` or preventing cancellation of shipped orders.
- **Repositories (DAO)**: Using Spring Data JPA, repository interfaces (`@Repository`) manage database operations. For example, `OrderRepository` extends a JPA repository to provide CRUD methods. This layer is responsible for data access only.
- **Data Models and Entities**: JPA entity classes map to database tables. An `Order` entity may map to an `orders` table, and `OrderItem` to `order_items`. Relationships (like one-to-many) are annotated so that saving an `Order` persists its items.
- **Validation**: Java Bean Validation annotations (`@NotNull`, `@Positive`, etc.) ensure request data is correct. Spring returns 400 Bad Request if validation fails.

A typical structure:

```
com.example.orderservice
   ├── controller (OrderController, etc.)
   ├── service (OrderService, etc.)
   ├── model (Order, OrderItem entities, DTOs)
   ├── repository (OrderRepository, OrderItemRepository)
   └── OrdersApplication (Spring Boot entry point)
```

## Pagination, Filtering, and Sorting

The `GET /orders` endpoint supports pagination, filtering, and sorting:

- **Pagination**: Clients specify `page` and `size` to limit results. The server returns a portion of data plus metadata (`totalElements`, etc.). This prevents large result sets from overwhelming the system.
- **Filtering**: Query parameters like `status=SHIPPED` or `customerId=123` can narrow results. The system interprets them to build queries. Validate and sanitize inputs to avoid injection or bad values.
- **Sorting**: A parameter (e.g., `sort=createdAt,desc`) lets clients choose sort fields. Ensure only valid fields are allowed. Default sorting might be by creation date descending.

The response typically includes both the data array and pagination info. For fast-changing data, consider cursor-based pagination, though page-based is usually sufficient for orders.

## Security Considerations

- **Authentication**: Require valid credentials (e.g., JWT token) on most endpoints. A stateless approach is common, with each request including a bearer token. 
- **Authorization (RBAC)**: Implement role-based rules. Admins can manage all orders, while standard users can only access their own. Return 403 Forbidden if a user tries to access someone else’s order.
- **Input Validation and Sanitization**: Thoroughly validate path, query, and JSON inputs to prevent injection. Use prepared statements, validate field ranges, and properly handle unexpected types or large strings.
- **Data Privacy**: Exclude sensitive info from responses. Store only necessary customer references. If handling payment data, do so securely (usually in a separate service).
- **Common Protections**: Enforce HTTPS to prevent eavesdropping. Log important actions. Consider rate limiting or throttling to mitigate abuse.

## Best Practices

- **HTTP Methods and Status Codes**: Use standard methods (GET, POST, PUT, DELETE) and return appropriate statuses (e.g., 201 for create, 200 for success, 404 for not found, 409 for conflicts).
- **RESTful Resource Design**: Keep URLs intuitive, use plural nouns, avoid verbs in endpoints. 
- **Idempotency and Safety**: GET requests should have no side effects, PUT and DELETE should be idempotent. POST is not idempotent by default; consider an idempotency key if duplicate creations are a concern.
- **Statelessness**: Each request is independent, passing all necessary info (auth tokens, filters) so any server instance can handle it.
- **Consistent Modeling**: Keep JSON fields and naming uniform. If major changes happen, consider versioning (e.g., `/v1/orders`).
- **Error Handling**: Return detailed error messages in JSON. For instance, `{"error":"Quantity must be positive"}` on a 400. Use global exception handling to keep responses consistent.
- **Documentation**: Provide OpenAPI/Swagger specs to describe endpoints, request/response schemas, and status codes. 
- **Testing**: Write unit and integration tests for controllers, services, and security rules. Verify correct status codes, error handling, and data flow.

## Interview Discussion

When presenting this design at a senior or principal engineer level, emphasize:

- **Layered Architecture**: Controllers handle HTTP logic, services enforce business rules, repositories manage data. This ensures maintainability.
- **Scalability**: The service is stateless and can be load-balanced. Pagination helps with large data sets. A robust database scheme and queries ensure good performance.
- **Security**: JWT auth or OAuth2, role-based permissions, data validation, and HTTPS. 
- **Trade-offs**: Page-based vs. cursor-based pagination, JWT vs. API keys, how to handle concurrency issues or distributed transactions if needed.
- **Monitoring**: Logging, metrics, and alerting for observability. 
- **Future Extensions**: Handling inventory checks, sending notifications, or integrating with third-party systems.

A thorough, best-practices-based approach to designing an Order Management API ensures clarity for clients, maintainability for developers, and robustness for real-world scenarios.
