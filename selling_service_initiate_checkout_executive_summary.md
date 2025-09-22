# Orchestration.checkout.initiate API Documentation

## Executive Summary

The `POST /orchestration/checkout/initiate` API is the core entry point for the selling orchestrator loop that handles the complete checkout flow from line item creation to payment confirmation. This API orchestrates a complex multi-step process involving payment group creation, pricing hydration, invoice generation, payment collection requirements (PCR), and event-driven state transitions.

**Key Characteristics:**
- **Async API**: Returns immediately with execution context ID for tracking
- **Idempotent**: Supports retry mechanisms with execution context persistence
- **Event-driven**: Produces `line_items_updated` and `line_items_confirmed` events
- **Distributed Locking**: Uses DLM to prevent concurrent execution on same entities
- **Step-by-step orchestration**: Each step is persisted and can be resumed on failure

## API Location & Entry Point

**Endpoint**: `POST /orchestration/checkout/initiate`  
**Handler**: `src/service/orchestration_engine/index.ts:6` → `InitiateCheckout.execute`  
**Schema**: `schema/service_schema.json:199-226`  
**Service Export**: `src/service/index.ts:31` → `orchestration.checkout.initiate`

## Call Graph & Flow

```
HTTP Request → InitiateCheckout.execute() → OrchestratorManager → executeOrchestratorSteps()
```

**Main Flow**:
1. **Validation & Setup** (`src/service/orchestration_engine/initiate_checkout/index.ts:152-252`)
2. **Payment Group Creation** (`src/service/payment_plan/service`)
3. **Orchestration Steps** (`src/service/orchestration_engine/initiate_checkout/orchestrator.ts:391-679`)

## Feature Flags & Configuration

**Feature Flags** (`src/service/orchestration_engine/initiate_checkout/index.ts:31-51`):
- `isLineItemPIIDataEncryptionEnabled`: Controls PII data encryption for line items
- `isExecutionContextPersistenceEnabled`: Enables execution context persistence for retries

**Configuration Sources**:
- XP Lib service for feature flag evaluation
- Device OS, city, and user ID based flag resolution

## Orchestration Sequence

### Mermaid Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant API as InitiateCheckout.execute
    participant DLM as Distributed Lock Manager
    participant OM as OrchestratorManager
    participant PG as PaymentGroup
    participant PS as PostSellingPreferences
    participant INV as Invoice Engine
    participant CO as CheckoutOrder
    participant OPT as OptIn Service
    participant LI as LineItems
    participant PCR as Payment Requirements
    participant EVT as Event Producer

    Client->>API: POST /orchestration/checkout/initiate
    API->>API: Validate request & get flags
    API->>PG: createPaymentGroup()
    PG-->>API: paymentGroupIds + lineItems
    
    API->>DLM: Acquire locks (executionContextId + paymentGroupIds)
    DLM->>OM: initializeExecutionContext()
    
    Note over OM: Step 1: Create Post-Selling Preferences
    OM->>PS: createPostSellingPreferences()
    PS-->>OM: preferenceIds
    
    Note over OM: Step 2: Get Invoice & Hydrate Pricing
    OM->>INV: getLineItemSummary()
    INV-->>OM: invoice + pricingHydratedLineItems
    
    Note over OM: Step 3: Create Checkout Order
    OM->>CO: createCheckoutOrder()
    CO-->>OM: checkoutOrderId
    
    Note over OM: Step 4: Handle OptIn Creation/Modification
    OM->>OPT: handleOptinCreationAndModification()
    OPT-->>OM: optInIds
    
    Note over OM: Step 5: Upsert Line Items & Create PCR
    OM->>LI: upsertLineItems()
    LI-->>OM: upsertedLineItems
    OM->>PCR: createPaymentRequest()
    PCR-->>OM: paymentCollectionRequirement
    
    Note over OM: Step 6: Hydrate Payments & Update Line Items
    OM->>LI: hydratePaymentAndUpdateLineItems()
    LI-->>OM: updatedLineItems
    
    Note over OM: Step 7: Create Preferences Mapping
    OM->>LI: createPreferencesToLineItemsMapping()
    
    Note over OM: Step 8: Produce Line Items Updated Event
    OM->>EVT: produceLineItemsUpdatedEvent()
    EVT-->>OM: eventPublished
    
    alt PCR Success (Sync)
        Note over OM: Step 9a: Execute PCR Success Steps
        OM->>OM: executePCRSuccessSteps()
        OM->>EVT: produceLineItemsConfirmedEvent()
    else PCR Async
        Note over OM: Step 9b: Wait for PCR Success Event
        Note over PCR: Payment processing...
        PCR->>EVT: payment_collection_requirement_success
        EVT->>OM: handlePCRSuccess()
        OM->>EVT: produceLineItemsConfirmedEvent()
    end
    
    OM-->>API: InitiateCheckoutResponse
    API-->>Client: 200 OK + response
```

### Detailed Step Breakdown

| Step | Function/Class | Input DTOs | Output DTOs | File Location |
|------|----------------|------------|-------------|---------------|
| 1 | `createPostSellingPreferences` | `CreatePostSellingPreferences` | `CreatePostSellingPreferencesResp` | `src/service/orchestration_engine/checkout_steps/create_post_selling_preferences.ts` |
| 2 | `getInvoiceAndHydratePricingForLineItems` | `HydratePricingForLineItems` | `GetInvoiceAndHydratePricingResponse` | `src/service/orchestration_engine/checkout_steps/get_invoice_and_hydrate_pricing.ts` |
| 3 | `createCheckoutOrder` | `CreateCheckout` | `CreateCheckoutOrderResponse` | `src/service/orchestration_engine/checkout_steps/create_checkout_order.ts` |
| 4 | `handleOptinCreationAndModification` | `HandleOptinCreationAndModification` | `HandleOptinCreationAndModificationResponse` | `src/service/orchestration_engine/checkout_steps/handle_optin_creation_and_modification.ts` |
| 5 | `upsertLineItems` + `createPaymentRequest` | `UpsertLineItemsRequest` + `CreatePaymentRequest` | `UpsertLineItemsResponse` + `CreatePaymentRequirementAPIResponse` | `src/service/orchestration_engine/checkout_steps/upsert_line_items.ts` + `create_payment_request.ts` |
| 6 | `hydratePaymentAndUpdateLineItems` | `HydratePaymentAndUpdateLineItems` | `UpsertLineItemsResponse` | `src/service/orchestration_engine/checkout_steps/hydrate_payment_and_update_line_items.ts` |
| 7 | `createPreferencesToLineItemsMapping` | `CreatePreferencesToLineItemsMapping` | `void` | `src/service/orchestration_engine/checkout_steps/create_line_item_and_preference_mapping.ts` |
| 8 | `produceLineItemsUpdatedEvent` | `LineItemsUpdatedEventParams` | `void` | `src/service/orchestration_engine/checkout_steps/produce_line_items_updated_event.ts` |
| 9 | `executePCRSuccessSteps` | `PCRSuccessPayload` | `InitiateCheckoutResponse` | `src/service/orchestration_engine/initiate_checkout/orchestrator.ts:340-389` |

## API Contracts

### Request Schema (`InitiateCheckoutRequestType`)

```json
{
  "userId": "string",
  "userType": "CUSTOMER | PARTNER",
  "device": {
    "os": "string",
    "version": "string"
  },
  "dimensions": {
    "city": { "key": "string" },
    "category": { "key": "string" }
  },
  "lineItems": {
    "created": [
      {
        "lineItemId": "string",
        "plan": { "planId": "string" },
        "initialPostSellingPreferences": { "location": "object" }
      }
    ],
    "updated": [
      {
        "lineItemId": "string",
        "preferences": { "location": "object" }
      }
    ],
    "existing": [
      {
        "lineItemId": "string"
      }
    ]
  },
  "sourceContext": {
    "id": "string",
    "type": "string"
  },
  "journeyContext": {
    "id": "string",
    "type": "CUSTOMER_JOURNEY"
  },
  "pricing": {
    "coupon": {
      "code": "string",
      "lockSessionId": "string"
    }
  },
  "payments": {
    "userPaymentIntent": "PAY_NOW | PAY_LATER"
  },
  "invoice": {
    "summaryComponents": {
      "charges": [{"type": "string", "value": "number"}],
      "discounts": [{"type": "string", "value": "number"}]
    },
    "shouldAttachArrears": "boolean",
    "arrearContext": "object",
    "editStrategy": "string"
  },
  "paymentPreferences": {
    "isCashPayment": "boolean",
    "selectedOnlinePaymentInfo": {
      "paymentMethodIdentifier": "string",
      "mandateInfo": {
        "preference": {
          "referenceId": "string",
          "referenceType": "OPT_IN"
        }
      }
    }
  },
  "planPreferences": {
    "routines": {
      "preferences": [{"oldOptinContext": "object"}],
      "updationContext": {"strategy": "string"}
    }
  },
  "isMock": "boolean",
  "executionContextId": "string",
  "categoriesData": [
    {
      "categoryKey": "string",
      "pricing": "object"
    }
  ]
}
```

### Response Schema (`InitiateCheckoutResponseType`)

```json
{
  "checkoutOrderId": "string",
  "lineItems": {
    "created": [
      {
        "lineItemId": "string",
        "preferenceId": "string"
      }
    ],
    "updated": [
      {
        "lineItemId": "string",
        "preferenceId": "string"
      }
    ]
  },
  "sourceContext": {
    "id": "string",
    "type": "string"
  },
  "paymentDetails": {
    "vendorTransactionInfo": {
      "skipProcessCall": "boolean",
      "processPaymentViaSDKParams": {
        "vendorCode": "string",
        "vendorTransactionId": "string",
        "returnUrl": "string",
        "redirectUrl": "string",
        "clientAuthToken": "string",
        "isMandateSetupOrder": "boolean",
        "amount": "number",
        "currency": "INR"
      }
    }
  }
}
```

## Internal Service Calls

| Service | Method | Purpose | Location |
|---------|--------|---------|----------|
| PaymentGroup | `createPaymentGroup` | Create payment groups for line items | `src/service/payment_plan/service` |
| PostSellingPreferences | `createPreferences` | Create post-selling preferences | `src/service/post_selling_preferences` |
| Invoice | `getLineItemSummary` | Get pricing and invoice details | `src/service/invoicing_engine` |
| CheckoutOrder | `createCheckoutOrder` | Create checkout order record | `src/service/checkout_order/service` |
| OptIn Service | `handleOptinCreationAndModification` | Create/update opt-ins | `src/service/orchestration_engine/checkout_steps/handle_optin_creation_and_modification.ts` |
| LineItems | `upsertLineItems` | Create/update line items | `src/service/lineItems/service` |
| PaymentRequirements | `createPaymentRequest` | Create payment collection requirement | `src/service/payment_requirements/service` |
| CustomerRepeatProgram | `initiateRepeatProgram` | Handle repeat program logic | `src/common/external/customer_repeat_program_service` |

## Events

### Produced Events

| Event Name | Topic | Payload Schema | Headers | Location |
|------------|-------|----------------|---------|----------|
| `line_items_updated` | `line_items_updated` | `LineItemsUpdatedEventParams` | `tenant`, `city`, `requestId`, `idempotencyKey` | `src/events/producer.ts:25-35` |
| `line_items_confirmed` | `line_items_confirmed` | `LineItemsConfirmedEventParams` | `tenant`, `city`, `requestId`, `idempotencyKey` | `src/events/producer.ts:37-47` |

### Consumed Events

| Event Name | Topic | Handler | Purpose |
|------------|-------|---------|---------|
| `payment_collection_requirement_success` | `payment_collection_requirement_success` | `handlePCRSuccess` | Complete checkout on payment success |
| `payment_collection_requirement_failed` | `payment_collection_requirement_failed` | `handlePCRFailure` | Handle payment failures |
| `line_items_confirmed` | `line_items_confirmed` | `handleLIsConfirmed` | Process confirmed line items |

## State Model

### ExecutionContext
```typescript
{
  executionContextId: string;
  taskName: "INITIATE_CHECKOUT";
  requestParams: InitiateCheckoutParams;
  referenceIds: [{id: string}]; // PaymentGroup IDs
  response: InitiateCheckoutResponse;
  contexts: StepContext[]; // Step execution history
  status: "INITIALIZED" | "COMPLETED" | "ROLLEDBACK";
  retryTimestamps: string[];
}
```

### LineItem States
- **CREATED**: New line item created
- **UPDATED**: Existing line item modified
- **CONFIRMED**: Payment confirmed, line item active
- **CANCELLED**: Line item cancelled

### PaymentGroup States
- **CREATED**: Payment group created
- **ACTIVE**: Payment group active
- **COMPLETED**: All payments collected
- **FAILED**: Payment collection failed

### CheckoutOrder States
- **CREATED**: Checkout order created
- **PROCESSING**: Payment in progress
- **COMPLETED**: Checkout completed
- **FAILED**: Checkout failed

### OptIn States
- **CREATED**: Opt-in created
- **ACTIVE**: Opt-in active
- **SUSPENDED**: Opt-in suspended
- **CANCELLED**: Opt-in cancelled

## Idempotency & Retry Mechanisms

### Idempotency Keys
- **Primary**: `executionContextId` (provided in request or auto-generated)
- **Secondary**: PaymentGroup IDs for distributed locking
- **Natural Keys**: LineItem IDs, CheckoutOrder ID

### Retry Strategy
1. **ExecutionContext Persistence**: Each step is persisted with success/failure status
2. **Resume from Last Step**: Failed executions resume from last successful step
3. **Distributed Locking**: Prevents concurrent execution on same entities
4. **Automatic Retry**: `RetrialMechanism.triggerTaskRetrialMechanism()` for transient failures

### Checkpoint Recovery
- **Step Guards**: Each orchestration step checks if already executed
- **Context Restoration**: Previous execution context restored on retry
- **Rollback Support**: Failed executions can be rolled back

## Validation & Business Rules

### Validators (`src/service/orchestration_engine/validations/checkout_initiate/`)

| Validator | Purpose | Location |
|-----------|---------|----------|
| `userValidation` | Validate user context | `user.ts` |
| `lineItemsValidation` | Validate line item data | `line_items.ts` |
| `arrearFeeValidation` | Validate arrear fees | `invoice_and_payments.ts` |
| `validatePaymentsParam` | Validate payment parameters | `src/service/invoicing_engine/validations/checkout_initiate.ts` |
| `validateCheckoutInitiate` | Validate payment plan | `src/service/payment_plan/validations/checkout_initiate.ts` |

### Business Rules
- **No Business Logic in Loop**: Orchestration calls services but doesn't branch on domain conditions
- **Current Payable Validation**: Ensures payable amount is valid
- **Slot Availability**: Validates slot availability for bookings
- **Coupon/Credits Validation**: Validates coupon codes and credit usage

## Error Handling & Retries

### Retry Policies

| Step | Max Attempts | Backoff | Circuit Breaker |
|------|--------------|---------|-----------------|
| PaymentGroup Creation | 3 | Exponential | Yes |
| Invoice Generation | 3 | Linear | Yes |
| LineItem Upsert | 3 | Exponential | Yes |
| PCR Creation | 3 | Exponential | Yes |
| Event Production | 2 | Linear | No |

### Failure Classification

| Error Type | Classification | Action |
|------------|----------------|--------|
| `PAYMENT_VALIDATION_ERROR_TYPE` | Terminal | Rollback transaction |
| `TRANSACTION_EXCEEDED_TIME_LIMIT_ERROR_CODE` | Terminal | Rollback transaction |
| Network timeouts | Transient | Retry with backoff |
| Database locks | Transient | Retry with backoff |
| External service failures | Transient | Retry with backoff |

### Compensations
- **Transaction Rollback**: MongoDB transactions for atomicity
- **Arrears Scheduling**: Failed payments scheduled for arrears recovery
- **Mid-tranche Settlement**: Partial payments handled via settlement

## Observability

### Logging
```typescript
// Key log points
Logger.info({
  methodName: 'initiateCheckout',
  key1: 'params',
  key1Value: JSON.stringify(request)
});

Logger.info({
  methodName: 'orchestrationEngine/initiateCheckout',
  message: 'execution initiated',
  key1: 'params',
  key1Value: JSON.stringify(params)
});
```

### Metrics
- **Execution Time**: Per step and total execution time
- **Success Rate**: Step-wise and overall success rates
- **Retry Count**: Number of retries per execution
- **Payment Success Rate**: PCR success/failure rates

### Traces
- **Correlation ID**: `executionContextId` for end-to-end tracing
- **Request ID**: For request-level tracing
- **PaymentGroup IDs**: For entity-level tracing

### Audit Records
- **ExecutionContext**: Complete execution history
- **Step Contexts**: Individual step execution details
- **Event Production**: Event publishing audit trail

## Examples

### Example 1: New Plan Purchase with Surge Slot Fee

**Request**:
```json
{
  "userId": "user_123",
  "userType": "CUSTOMER",
  "device": {"os": "android", "version": "12"},
  "dimensions": {
    "city": {"key": "delhi"},
    "category": {"key": "cleaning"}
  },
  "lineItems": {
    "created": [{
      "lineItemId": "li_456",
      "plan": {"planId": "plan_789"},
      "initialPostSellingPreferences": {
        "location": {"address": "123 Main St"}
      }
    }]
  },
  "sourceContext": {"id": "journey_101", "type": "CUSTOMER_JOURNEY"},
  "payments": {"userPaymentIntent": "PAY_NOW"},
  "paymentPreferences": {
    "isCashPayment": false,
    "selectedOnlinePaymentInfo": {
      "paymentMethodIdentifier": "pm_123",
      "mandateInfo": {
        "preference": {
          "referenceId": "optin_456",
          "referenceType": "OPT_IN"
        }
      }
    }
  }
}
```

**Success Response**:
```json
{
  "checkoutOrderId": "co_789",
  "lineItems": {
    "created": [{
      "lineItemId": "li_456",
      "preferenceId": "pref_123"
    }]
  },
  "sourceContext": {"id": "journey_101", "type": "CUSTOMER_JOURNEY"},
  "paymentDetails": {
    "vendorTransactionInfo": {
      "skipProcessCall": false,
      "processPaymentViaSDKParams": {
        "vendorCode": "razorpay",
        "vendorTransactionId": "txn_456",
        "amount": 1500,
        "currency": "INR",
        "clientAuthToken": "token_789"
      }
    }
  }
}
```

### Example 2: Tip Update Only (No Slot Change)

**Request**:
```json
{
  "userId": "user_123",
  "userType": "CUSTOMER",
  "device": {"os": "ios", "version": "15"},
  "dimensions": {
    "city": {"key": "mumbai"},
    "category": {"key": "cleaning"}
  },
  "lineItems": {
    "updated": [{
      "lineItemId": "li_existing_123",
      "preferences": {
        "location": {"address": "456 Oak Ave"},
        "tip": {"amount": 200}
      }
    }]
  },
  "sourceContext": {"id": "optin_456", "type": "OPT_IN"},
  "payments": {"userPaymentIntent": "PAY_NOW"}
}
```

### Example 3: Renewal with Payable=0 (No PCR)

**Request**:
```json
{
  "userId": "user_123",
  "userType": "CUSTOMER",
  "device": {"os": "android", "version": "11"},
  "dimensions": {
    "city": {"key": "bangalore"},
    "category": {"key": "cleaning"}
  },
  "lineItems": {
    "existing": [{"lineItemId": "li_renewal_789"}]
  },
  "sourceContext": {"id": "optin_789", "type": "OPT_IN"},
  "payments": {"userPaymentIntent": "PAY_NOW"},
  "invoice": {
    "summaryComponents": {
      "charges": [],
      "discounts": [{"type": "CREDIT", "value": 1000}]
    }
  }
}
```

## Edge Cases

### Critical Edge Cases

1. **No Mandate Present**
   - **Scenario**: User doesn't have payment mandate
   - **Handling**: Creates mandate setup order via `isMandateSetupOrder: true`

2. **COD Path**
   - **Scenario**: Cash on delivery payment
   - **Handling**: `isCashPayment: true`, skips PCR creation

3. **Partial Failures**
   - **Scenario**: Pricing succeeds, PCR fails
   - **Handling**: Transaction rollback, retry mechanism triggered

4. **Duplicate Submissions**
   - **Scenario**: Same request submitted multiple times
   - **Handling**: Idempotency via `executionContextId`

5. **Payment Confirmed Late**
   - **Scenario**: Payment webhook arrives after timeout
   - **Handling**: Event-driven confirmation via `payment_collection_requirement_success`

6. **LineItem Cancel After Pricing**
   - **Scenario**: Line item cancelled during processing
   - **Handling**: Rollback and compensation

7. **Tranche Settlement Scheduled**
   - **Scenario**: Mid-tranche settlement required
   - **Handling**: Settlement scheduling post-completion

8. **Post-booking Tip**
   - **Scenario**: Tip added after booking
   - **Handling**: Separate tip update flow

## Security & Data

### Required Auth Scopes/Roles
- **Customer**: `CUSTOMER` user type
- **Partner**: `PARTNER` user type
- **Admin**: System-level access

### PII Data Touched
- **User ID**: Customer identification
- **Location Data**: Address information in preferences
- **Payment Info**: Payment method details
- **Device Info**: OS and version for feature flags

### GDPR/Retention Notes
- **PII Encryption**: Controlled by `isLineItemPIIDataEncryptionEnabled` flag
- **Data Retention**: Execution context retained for retry purposes
- **Audit Trail**: Complete execution history maintained

### Write Boundaries
- **Line Items**: Only orchestration loop can mutate
- **Payment Groups**: Only payment plan service can modify
- **Checkout Orders**: Only checkout order service can update
- **Opt-ins**: Only opt-in service can modify

## Test Plan

### Test Matrix

| Test Category | Test Cases | Expected Outcome |
|---------------|------------|------------------|
| **Happy Path** | New plan purchase, tip update, renewal | Successful completion |
| **Retries** | Network timeout, DB lock, service unavailable | Automatic retry and recovery |
| **Idempotency** | Duplicate requests with same executionContextId | Return cached response |
| **Validator Failures** | Invalid user, missing line items, invalid pricing | Validation errors |
| **PCR Failure** | Payment gateway failure, insufficient funds | Rollback and retry |
| **Payment Confirmation Race** | Webhook arrives before timeout | Event-driven confirmation |
| **Arrears Creation** | Failed payment scenarios | Arrears scheduled |
| **Tranche Settlement** | Mid-tranche scenarios | Settlement scheduled |

### Jest Test Skeletons

```typescript
// test/unit/service/orchestration_engine/initiate_checkout.test.ts

describe('InitiateCheckout', () => {
  describe('execute', () => {
    it('should successfully complete new plan purchase', async () => {
      const mockRequest = {
        userId: 'user_123',
        userType: 'CUSTOMER',
        // ... complete request
      };
      
      const mockPaymentGroup = {
        lineItems: [{ lineItemId: 'li_123' }],
        paymentGroupIds: { created: ['pg_123'] }
      };
      
      jest.spyOn(PaymentGroup, 'createPaymentGroup').mockResolvedValue(mockPaymentGroup);
      jest.spyOn(OrchestratorManager, 'executeOrchestratorUnit').mockResolvedValue({});
      
      const result = await InitiateCheckout.execute(mockRequest);
      
      expect(result.checkoutOrderId).toBeDefined();
      expect(result.lineItems.created).toHaveLength(1);
    });

    it('should handle retry with existing execution context', async () => {
      const mockExecutionContext = {
        executionContextId: 'ctx_123',
        status: ExecutionContextStatus.COMPLETED,
        response: { checkoutOrderId: 'co_123' }
      };
      
      jest.spyOn(OrchestratorManager, 'getExecutionContext').mockResolvedValue(mockExecutionContext);
      
      const result = await InitiateCheckout.execute({ executionContextId: 'ctx_123' });
      
      expect(result).toEqual(mockExecutionContext.response);
    });

    it('should handle PCR failure and trigger retry mechanism', async () => {
      const mockRequest = { /* ... */ };
      const mockError = new Error('PCR_FAILED');
      mockError.generatedExecutionContextId = 'ctx_456';
      
      jest.spyOn(PaymentGroup, 'createPaymentGroup').mockRejectedValue(mockError);
      jest.spyOn(RetrialMechanism, 'triggerTaskRetrialMechanism').mockResolvedValue();
      
      await expect(InitiateCheckout.execute(mockRequest)).rejects.toThrow('PCR_FAILED');
      expect(RetrialMechanism.triggerTaskRetrialMechanism).toHaveBeenCalled();
    });
  });
});
```

## Gaps & TODOs

### Missing Contracts
- [ ] **Idempotency Keys**: Make idempotency keys explicit in all service calls
- [ ] **Event Headers**: Standardize event headers across all events
- [ ] **Error Codes**: Unify error codes across services

### Undocumented Headers
- [ ] **Correlation ID**: Add correlation ID to all external service calls
- [ ] **Request ID**: Standardize request ID format
- [ ] **Tenant Context**: Add tenant context to all operations

### Unclear State Transitions
- [ ] **OptIn States**: Document all OptIn state transitions
- [ ] **PaymentGroup States**: Clarify PaymentGroup lifecycle
- [ ] **LineItem States**: Document LineItem state machine

### Magic Constants
- [ ] **Retry Limits**: Make retry limits configurable
- [ ] **Timeout Values**: Externalize timeout configurations
- [ ] **Feature Flags**: Document all feature flag dependencies

### Proposed Improvements
1. **Outbox Pattern**: Implement outbox pattern for event publishing
2. **Circuit Breakers**: Add circuit breakers for all external service calls
3. **Metrics Dashboard**: Create comprehensive metrics dashboard
4. **Alerting**: Implement alerting for critical failures
5. **Documentation**: Auto-generate API documentation from schemas

## Operational Runbook: Debugging Stuck Checkouts

### Step 1: Identify the Issue
```bash
# Check execution context status
db.execution_context.findOne({executionContextId: "ctx_123"})

# Check payment group status
db.payment_groups.find({_id: {$in: ["pg_123", "pg_456"]}})

# Check line item status
db.line_items.find({lineItemId: {$in: ["li_123", "li_456"]}})
```

### Step 2: Analyze Execution Context
```bash
# Get step-by-step execution history
db.execution_context.findOne(
  {executionContextId: "ctx_123"},
  {contexts: 1, status: 1, retryTimestamps: 1}
)
```

### Step 3: Check Event Status
```bash
# Check if events were produced
db.event_logs.find({
  eventName: {$in: ["line_items_updated", "line_items_confirmed"]},
  correlationId: "ctx_123"
})
```

### Step 4: Manual Recovery Actions
1. **If stuck at PCR creation**: Check payment gateway status
2. **If stuck at event production**: Manually trigger event
3. **If stuck at line item update**: Check database locks
4. **If stuck at opt-in creation**: Verify opt-in service health

### Step 5: Retry Mechanism
```bash
# Trigger manual retry
POST /orchestration/checkout/initiate
{
  "executionContextId": "ctx_123",
  // ... original request params
}
```

### Step 6: Rollback if Necessary
```bash
# Mark execution context as rolled back
db.execution_context.updateOne(
  {executionContextId: "ctx_123"},
  {$set: {status: "ROLLEDBACK"}}
)
```

---

This comprehensive documentation provides a complete understanding of the Orchestration.checkout.initiate API, its flow, contracts, and operational aspects. The system is designed for high reliability with robust retry mechanisms, distributed locking, and event-driven architecture.
