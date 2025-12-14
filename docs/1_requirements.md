## Functional Requirements

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