## **1. Functional Requirements (WHAT)**

### **FR1: Accept product update events from Shopify**

**Client Statement:** *"Customers order products showing 'in stock' on Shopify, but they're actually sold out on Salesforce"*

- Problem originates when Shopify has outdated data
- **Receive** updates from Shopify when something changes
- In real Shopify integration, this would be a webhook, but this will be simulated for this project

Without a way to receive update, Shopify would have to be constantly polled (inefficient). Webhooks are event-driven - they push data to other systems when changes occur.

```
a REST endpoint that can receive HTTP POST requests
containing product data (ID, name, price, stock level) is required.
```

The choice between webhooks (real-time) vs batch processing (hourly/daily) is determined by how quickly product updates in Shopify - price change, inventory adjustment, etc. - should be reflected in Salesforce.

***

### **FR2: Transform data to Salesforce format**

**Client Statement:** *"We're running Shopify and Salesforce Commerce Cloud"*

- Two different systems = two different data formats
- Shopify might call it `inventory_quantity`, Salesforce might call it `stockLevel`
- Field types might differ (strings vs integer, date formats, etc.)

Systems rarely speak the same "language." If Shopify sends:

```json
{
  "product_id": "SHOP-12345",
  "inventory_count": 50,
  "price": "99.99"
}
```

Salesforce might expect:

```json
{
  "sku": "SHOP-12345",
  "stockLevel": 50,
  "listPrice": {
    "amount": 99.99,
    "currency": "USD"
  }
}
```

```
Mapping logic (MapStruct) is required to convert between different schemas. This is the "translation layer."
```

This is where **80% of integration bugs** occur. Mismatched field names, wrong data types, missing fields. String mapping and validation is needed.

***

### **FR3: Push updates to Salesforce Commerce Cloud**

**Client Statement:** *"Both systems need to stay in sync"*

- Reading data isn't enough - **writing data** to Salesforce is required
- This is essentially a middleware/bridge between systems

The sync is incomplete if data is only received and stored. It's necessary to:

1. Receive from Shopify
2. Transform the data
3. **Send to Salesforce** (FR3)

```
an HTTP client (RestTemplate/WebClient) is required to make
outbound API calls to Salesforce endpoints.
```

In production, Salesforce has rate limits (API call quotas). It's necessary to handle throttling, but for this project, Salesforce is simulated as a mock service.

***

### **FR4: Handle bidirectional sync**

**Client Statement:** *"Inventory changes in either system should update the other"*

- Sync isn't one-way (Shopify → Salesforce only)
  - B2C customer buys on Shopify (stock: 50 → 49)
  - B2B customer buys on Salesforce (stock: 50 → 45)
  - Without bidirectional sync, both systems show different inventory, causing overselling
- Sales on Salesforce should reduce Shopify inventory too
- This is **bidirectional data flow**

```
TWO endpoints are required:
- POST /api/sync/from-shopify (receives Shopify updates)
- POST /api/sync/from-salesforce (receives Salesforce updates)

Each endpoint transforms and pushes to the opposite system.
```

Bidirectional sync introduces **conflict resolution**, if both systems update the same product simultaneously. For this project, "last write wins" (timestamp-based) is used, but in production, business rules are required.

***

### **FR5: Maintain audit trail of all sync operations**

**Client Statement:** *"inventory discrepancies"*

- When things go wrong, it should be possible to trace what happened
- "Why does Salesforce show 10 units but Shopify shows 15?"

Audit logs are an **operational necessity**. Without them, it becomes challenging to:

- debug production issues
- prove compliance (some industries require audit trails)
- analyze failure patterns

```
Store every sync operation in PostgreSQL:
- Timestamp
- Source system
- Target system
- Product ID
- Old value vs New value
- Success/Failure status
- Error message (if failed)
```

Audit logs are similar to flight recorders, hopefully not needed, but invaluable when a plane crash occurs. State changes in integration systems should always be logged.

***

### **FR6: Retry failed syncs automatically**

**Client Statement:** *"Updates are delayed"*

- Network calls fail (timeouts, service downtime, rate limits)
- Currently, failed syncs just... fail. No recovery mechanism.

In distributed systems, **failure is normal**:

- Salesforce API might be temporarily down (99.9% uptime = 43 minutes downtime/month)
- Network hiccups cause timeouts
- Rate limits cause temporary rejections

Without retry, inventory updates can be lost.

```
Implement retry logic with exponential backoff:
- 1st attempt fails → wait 2 seconds, retry
- 2nd attempt fails → wait 4 seconds, retry  
- 3rd attempt fails → wait 8 seconds, retry
- After N attempts → log as permanent failure, alert operations
```

**Idempotent retry logic:** Not all failures should retry. A `400 Bad Request` (malformed data) won't fix itself - retrying is pointless. A `503 Service Unavailable` should retry.

***

## **2. Non-Functional Requirements (HOW WELL)**

### **NFR1: Response time < 2 secs**

**Client Statement:** *"near-real-time sync"*

- UX research shows 2 secs is the threshold before users perceive slowness
- This affects operational workflows

```
User action (product update) → Sync API called → Response
                               <---- 2 seconds ---->
```

Syncing taking 10 seconds might cause staff to update the same product twice, resulting in conflicts.

- Database queries must be optimized (indexes on product_id)
- External API calls need timeouts (don't wait forever)
- Use asynchronous processing if sync takes longer (queue-based)

Add logging: `log.info("Sync completed in {}ms", duration);`

***

### **NFR2: Availability 99.5% uptime**

A **business decision** based on cost-benefit analysis:

| Uptime | Downtime/Year | Cost to Achieve | Business Impact |
| :-- | :-- | :-- | :-- |
| 99% | 3.65 days | Low | High |
| 99.5% | 1.83 days | Medium | Acceptable |
| 99.9% | 8.76 hours | High | Low |
| 99.99% | 52.6 minutes | Very High | Minimal |

- **Cost:** Achieving 99.9% requires redundant servers, load balancers, monitoring - expensive
- **Impact:** 1.83 days downtime spread across a year (mostly during off-peak maintenance) is acceptable
- **SLA:** Any client that isn't Amazon - short downtimes for patches are okay

```
- Health check endpoints (`/actuator/health`)
- Graceful shutdown (finish processing requests before restarting)
- Error handling (one bad request shouldn't crash the server)
```

100% uptime is mathematically impossible and economically impractical. Not even AWS can guarantee 100%.

***

### **NFR3: Data consistency - No data loss**

- Lost inventory updates = lost revenue
- Inconsistent data = customer complaints

```
If Shopify sends: "Product A: 100 units"
Then Salesforce MUST receive: "Product A: 100 units"
NOT: 99 units, NOT: missing update
```

- **Database transactions:** If saving audit log fails, rollback the sync
- **Idempotency:** Same request twice = same result (prevent duplicate syncs)
- **Dead letter queue:** If sync fails after all retries, don't lose the data - store it for manual review

```java
@Transactional // If anything fails, rollback everything
public void syncProduct(ProductDTO product) {
    // 1. Save to audit log
    // 2. Call external API
    // 3. Update audit log with result
    // If step 2 fails, step 1 rollback automatically
}
```

***

### **NFR4: Scalability - 1000 products, 100 syncs/hour**

Hypothetical current scale:

- **1000 products:** current catalog size
- **100 syncs/hour:** Average update frequency (price changes, inventory adjustments)
- A system handling 100 syncs/hour has different architecture than one handling 10,000/sec
- Over-engineering is wasteful; under-engineering causes failure

For this scale:

- **Simple REST API is fine** (no need for Kafka, RabbitMQ)
- **Single PostgreSQL instance works** (no need for sharding)
- **Synchronous processing okay** (no need for complex queuing)

Build for **10x current load**, not 1000x. If they grow to 10,000 products and 1,000 sync/hour, this system should handle it. If they 100x overnight (unlikely), then refactor.

Refactor if:

- syncs exceed 500/hour consistently
- response times degrade
- database queries slow down

## **Constraints:**

### **TC1:** Audit logs will be stored in a simple table:

```sql
CREATE TABLE sync_audit (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP,
    product_id VARCHAR(50),
    source_system VARCHAR(20),
    target_system VARCHAR(20),
    status VARCHAR(20),
    error_message TEXT
);
```

REST conventions:

- `POST /api/products/sync` - Create/update a product sync
- `GET /api/products/sync/{id}` - Retrieve sync status
- `GET /api/products/sync` - List all syncs

### **TC1:** Docker setup

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres
    environment:
      POSTGRES_DB: product_sync_sys
      
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - postgres
```