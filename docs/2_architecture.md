## **Architecture Design**

### **Logical View: System Components**

1. **REST Controller (Presentation Layer)** → receive HTTP requests from Shopify
   - Handles HTTP requests/responses, validation, error formatting
   - Controllers are "thin" - just routing and validation
2. **Business Logic (Service layer)** → process business logic
   - Orchestrate the sync process, apply business rules
3. **External Client (Integration Layer)** → call Salesforce API
   - Communicate with external systems (Shopify, Salesforce)
   - Client classes encapsulate all HTTP details (header, auth, error handling)
4. **Database Repository (Data Access layer)** → store audit logs
   - Database operations
5. **External Systems** → systems to integrate with (Shopify, Salesforce)

**Layered Architecture Pattern:**

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT SYSTEMS                        │
│                  (Shopify, Salesforce)                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ HTTP/REST
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                   PRESENTATION LAYER                         │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         ProductSyncController                        │   │
│  │  - POST /api/sync/from-shopify                      │   │
│  │  - POST /api/sync/from-salesforce                   │   │
│  │  - GET  /api/sync/{id}                              │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ DTOs
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                    SERVICE LAYER                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         ProductSyncService                           │  │
│  │  - syncFromShopify(product)                         │  │
│  │  - syncFromSalesforce(product)                      │  │
│  │  - getSyncStatus(id)                                │  │
│  └─────────────┬────────────────────────┬───────────────┘  │
│                │                        │                   │
│  ┌─────────────▼──────────┐  ┌─────────▼──────────────┐  │
│  │  DataTransformService  │  │  RetryService          │  │
│  │  - toSalesforceFormat  │  │  - retryWithBackoff    │  │
│  │  - toShopifyFormat     │  │                        │  │
│  └────────────────────────┘  └────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                │                       │
┌───────────────▼──────────┐  ┌────────▼─────────────────────┐
│   INTEGRATION LAYER      │  │    DATA ACCESS LAYER         │
│  ┌────────────────────┐  │  │  ┌────────────────────────┐ │
│  │ ShopifyClient      │  │  │  │ SyncAuditRepository    │ │
│  │ - updateProduct()  │  │  │  │ - save()               │ │
│  └────────────────────┘  │  │  │ - findByProductId()    │ │
│  ┌────────────────────┐  │  │  └────────────────────────┘ │
│  │ SalesforceClient   │  │  │                              │
│  │ - updateProduct()  │  │  │  ┌────────────────────────┐ │
│  └────────────────────┘  │  │  │   SyncAudit (Entity)   │ │
└──────────────────────────┘  │  └────────────────────────┘ │
                              └──────────────┬───────────────┘
                                             │
                                             │ JDBC
                                             │
                              ┌──────────────▼───────────────┐
                              │      PostgreSQL Database     │
                              │        sync_audit table      │
                              └──────────────────────────────┘
```

***

### **Process View - Data Flow**

#### **Sequence Diagram: Happy Path**

```
Shopify          Controller       Service          Transform       SalesforceClient    Database
  │                  │               │                  │                │               │
  │  POST /sync      │               │                  │                │               │
  │─────────────────>│               │                  │                │               │
  │                  │               │                  │                │               │
  │                  │ syncFromShopify()                │                │               │
  │                  │──────────────>│                  │                │               │
  │                  │               │                  │                │               │
  │                  │               │ Save audit (PENDING)              │               │
  │                  │               │──────────────────────────────────────────────────>│
  │                  │               │<───────────────────────────────────────────────────│
  │                  │               │                  │                │               │
  │                  │               │ toSalesforceFormat()              │               │
  │                  │               │─────────────────>│                │               │
  │                  │               │<─────────────────│                │               │
  │                  │               │                  │                │               │
  │                  │               │ updateProduct()                   │               │
  │                  │               │──────────────────────────────────>│               │
  │                  │               │                  │                │               │
  │                  │               │                  │  PUT /products/12345           │
  │                  │               │                  │                │──────────┐    │
  │                  │               │                  │                │          │    │
  │                  │               │                  │                │<─────────┘    │
  │                  │               │                  │                │  200 OK       │
  │                  │               │<──────────────────────────────────│               │
  │                  │               │                  │                │               │
  │                  │               │ Update audit (SUCCESS)            │               │
  │                  │               │──────────────────────────────────────────────────>│
  │                  │               │<───────────────────────────────────────────────────│
  │                  │               │                  │                │               │
  │                  │<──────────────│                  │                │               │
  │                  │  SyncResult   │                  │                │               │
  │<─────────────────│               │                  │                │               │
  │  200 OK          │               │                  │                │               │
```


#### **Sequence Diagram: Failure with Retry**

```
Shopify          Service          SalesforceClient         RetryService         Database
  │                 │                      │                      │                 │
  │ syncFromShopify()                     │                      │                 │
  │────────────────>│                      │                      │                 │
  │                 │                      │                      │                 │
  │                 │ Save audit (PENDING) │                      │                 │
  │                 │──────────────────────────────────────────────────────────────>│
  │                 │                      │                      │                 │
  │                 │ executeWithRetry()   │                      │                 │
  │                 │─────────────────────────────────────────────>│                 │
  │                 │                      │                      │                 │
  │                 │                      │   Attempt 1          │                 │
  │                 │                      │<─────────────────────│                 │
  │                 │                      │ updateProduct()      │                 │
  │                 │                      │                      │                 │
  │                 │                      │  PUT /products/...   │                 │
  │                 │                      │──────────┐           │                 │
  │                 │                      │          │           │                 │
  │                 │                      │<─────────┘           │                 │
  │                 │                      │  503 Service Unavailable             │
  │                 │                      │─────────────────────>│                 │
  │                 │                      │                      │                 │
  │                 │                      │  Wait 2s, Attempt 2  │                 │
  │                 │                      │<─────────────────────│                 │
  │                 │                      │ updateProduct()      │                 │
  │                 │                      │                      │                 │
  │                 │                      │  PUT /products/...   │                 │
  │                 │                      │──────────┐           │                 │
  │                 │                      │<─────────┘           │                 │
  │                 │                      │  200 OK              │                 │
  │                 │                      │─────────────────────>│                 │
  │                 │<─────────────────────────────────────────────│                 │
  │                 │                      │                      │                 │
  │                 │ Update audit (SUCCESS)                      │                 │
  │                 │──────────────────────────────────────────────────────────────>│
```

### **Database Schema Design**

```sql
CREATE TABLE sync_audit (
    -- Primary key
    id BIGSERIAL PRIMARY KEY,
    
    -- Core sync information
    product_id VARCHAR(100) NOT NULL,
    source_system VARCHAR(50) NOT NULL,
    target_system VARCHAR(50) NOT NULL,
    
    -- Status tracking
    status VARCHAR(20) NOT NULL,  -- PENDING, SUCCESS, FAILED, RETRYING
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    duration_ms INTEGER,  -- How long the sync took
    
    -- Error handling
    error_message TEXT,
    error_code VARCHAR(50),
    retry_count INTEGER DEFAULT 0,
    
    -- Data snapshot (for debugging)
    request_payload JSONB,  -- What we sent to target system
    response_payload JSONB, -- What we got back
    
    -- Audit fields
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes for common queries
CREATE INDEX idx_product_id ON sync_audit(product_id);
CREATE INDEX idx_timestamp ON sync_audit(timestamp DESC);
CREATE INDEX idx_status ON sync_audit(status);
CREATE INDEX idx_status_timestamp ON sync_audit(status, timestamp DESC);

-- Composite index for finding recent failures by product
CREATE INDEX idx_product_status_timestamp 
    ON sync_audit(product_id, status, timestamp DESC);
```

***

### **API Contract Design**

#### **Endpoint 1: Sync from Shopify**

**Request:**

```
POST /api/sync/from-shopify
Content-Type: application/json

{
  "productId": "SHOP-12345",
  "name": "Wireless Mouse",
  "description": "Ergonomic wireless mouse with 6 buttons",
  "price": 29.99,
  "currency": "USD",
  "inventoryCount": 150,
  "sku": "WM-001",
  "category": "Electronics",
  "lastUpdated": "2025-12-14T10:30:00Z"
}
```

**Response (Success):**

```
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "syncId": 12345,
  "status": "PENDING",
  "message": "Product sync queued successfully",
  "estimatedProcessingTime": "2-5 seconds",
  "statusCheckUrl": "/api/sync/12345"
}
```

**Response (Validation Error):**

```
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "VALIDATION_ERROR",
  "message": "Invalid product data",
  "details": [
    {
      "field": "price",
      "error": "must be greater than 0"
    },
    {
      "field": "inventoryCount",
      "error": "must not be null"
    }
  ]
}
```

#### **Endpoint 2: Sync from Salesforce**

**Request:**

```
POST /api/sync/from-salesforce
Content-Type: application/json

{
  "sku": "WM-001",
  "productName": "Wireless Mouse",
  "listPrice": {
    "amount": 29.99,
    "currency": "USD"
  },
  "stockLevel": 148,
  "lastModifiedDate": "2025-12-14T10:35:00Z"
}
```

#### **Endpoint 3: Check Sync Status**

**Request:**

```
GET /api/sync/12345
```

**Response:**

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "syncId": 12345,
  "productId": "SHOP-12345",
  "sourceSystem": "SHOPIFY",
  "targetSystem": "SALESFORCE",
  "status": "SUCCESS",
  "timestamp": "2025-12-14T10:30:05Z",
  "durationMs": 450,
  "retryCount": 0
}
```

#### **Endpoint 4: Get Sync History**

**Request:**

```
GET /api/sync/history?productId=SHOP-12345&limit=10
```

**Response:**

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "productId": "SHOP-12345",
  "totalSyncs": 45,
  "successRate": 95.6,
  "syncs": [
    {
      "syncId": 12345,
      "status": "SUCCESS",
      "timestamp": "2025-12-14T10:30:05Z",
      "sourceSystem": "SHOPIFY",
      "targetSystem": "SALESFORCE"
    },
    {
      "syncId": 12344,
      "status": "SUCCESS",
      "timestamp": "2025-12-13T15:20:10Z",
      "sourceSystem": "SALESFORCE",
      "targetSystem": "SHOPIFY"
    }
  ]
}
```


### **Package Structure**

```
src/main/java/com/dav/productsync/
│
├── ProductSyncApplication.java           # Main Spring Boot class
│
├── controller/                           # REST endpoints
│   ├── ProductSyncController.java
│   └── SyncStatusController.java
│
├── service/                              # Business logic
│   ├── ProductSyncService.java           # Main orchestration
│   ├── DataTransformService.java         # DTO transformations
│   └── RetryService.java                 # Retry logic
│
├── client/                               # External API clients
│   ├── SalesforceClient.java
│   └── ShopifyClient.java                # (For bidirectional sync)
│
├── repository/                           # Database access
│   └── SyncAuditRepository.java
│
├── model/                                # Data models
│   ├── entity/
│   │   └── SyncAudit.java                # JPA entity
│   ├── dto/
│   │   ├── ShopifyProductDTO.java        # Request from Shopify
│   │   ├── SalesforceProductDTO.java     # Request to Salesforce
│   │   ├── SyncResponse.java             # API response
│   │   └── SyncStatusDTO.java
│   └── mapper/
│       └── ProductMapper.java            # MapStruct mapper
│
├── config/                               # Spring configuration
│   ├── RestTemplateConfig.java           # HTTP client config
│   ├── RetryConfig.java                  # Retry policy config
│   └── DatabaseConfig.java               # DataSource config
│
├── exception/                            # Custom exceptions
│   ├── SyncException.java
│   ├── SalesforceApiException.java
│   └── ValidationException.java
│
└── util/                                 # Utility classes
    └── JsonUtil.java                     # JSON serialization helpers

src/main/resources/
├── application.yml                       # Configuration
├── application-dev.yml                   # Dev environment
├── application-prod.yml                  # Production environment
└── db/migration/
    └── V1__create_sync_audit_table.sql   # Flyway migration

src/test/java/com/dav/productsync/
├── controller/
│   └── ProductSyncControllerTest.java
├── service/
│   └── ProductSyncServiceTest.java
└── integration/
    └── ProductSyncIntegrationTest.java
```

***

### **Tech Stack:**

- Spring Boot
- PostgreSQL
- MapStruct
- Resilience4j
- Lombok

***

### **Configuration management**

**application.yml:**

```yaml
spring:
  application:
    name: product-sync-service
  
  datasource:
    url: jdbc:postgresql://localhost:5432/product_sync
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: validate  # Don't auto-create tables (use Flyway)
    show-sql: false
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect

# External API configuration
salesforce:
  api:
    url: ${SALESFORCE_API_URL:http://localhost:9090/api}
    timeout: 5000  # 5 seconds
    
shopify:
  api:
    url: ${SHOPIFY_API_URL:http://localhost:9091/api}
    timeout: 5000

# Retry configuration
resilience4j:
  retry:
    instances:
      externalApi:
        maxAttempts: 3
        waitDuration: 2s
        exponentialBackoffMultiplier: 2

# Logging
logging:
  level:
    com.techbridge.productsync: DEBUG
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"

# Server configuration
server:
  port: 8080
  error:
    include-message: always
    include-stacktrace: on_param  # Include stack trace if ?trace=true
```

**application-dev.yml:**

```yaml
salesforce:
  api:
    url: http://localhost:9090/mock-salesforce

shopify:
  api:
    url: http://localhost:9091/mock-shopify

logging:
  level:
    com.dav.productsync: DEBUG
```

**application-prod.yml:**

```yaml
salesforce:
  api:
    url: https://api.salesforce.com

shopify:
  api:
    url: https://api.shopify.com

logging:
  level:
    com.dav.productsync: INFO
```