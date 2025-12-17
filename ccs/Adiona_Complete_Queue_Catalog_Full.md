# Adiona System – Complete Queue Catalog

This document provides a comprehensive overview of the **Adiona Queue Architecture**, including all queues, their responsibilities, listeners, retry mechanisms, and execution patterns. It also includes the **end-to-end queue flow diagram**.

---

## Queue Flow Summary 📊

```
External Events → EXT_EVENTS_INGESTION_QUEUE
                ↓
          [Validation & Routing]
                ↓
        ┌───────┴────────┐
        ↓                ↓
EXT_EVENTS_QUEUE   EXT_EVENTS_BULK_QUEUE
        ↓                ↓
  [Journey State Processing]
        ↓                ↓
INT_EVENTS_QUEUE   INT_EVENTS_BULK_QUEUE
        ↓                ↓
  [Block Execution: Incentives, Wait, Engagement, etc.]
        ↓
   [Next Block]
        ↓
  Loop back to INT_EVENTS_BULK_QUEUE
        ↓
  [Exit or FAILED_EVENTS_DLQ]
```

---

## 1. Core Event Processing Queues ⚡

### A. External Event Queues

#### EXT_EVENTS_INGESTION_QUEUE
**Purpose:** First entry point for external events from outside systems  
**Listener:** ExtEventsIngestionListener  

**Flow:**
- Receives raw external events (user actions, transactions, etc.)
- Validates and processes event ingestion
- Routes events to appropriate journey processing queues  

**Retry Queue:** EXT_EVENTS_INGESTION_DELAY_RETRY_QUEUE  
**TTL:** ~16 minutes  

---

#### EXT_EVENTS_QUEUE
**Purpose:** Individual external event processing  
**Listener:** ExtEventsListener  

**Flow:**
- Processes one external event at a time
- Evaluates event relevancy for journeys
- Creates/updates journey state  

**Retry Queue:** EXT_EVENTS_DELAY_RETRY_QUEUE  
**TTL:** ~16 minutes  

---

#### EXT_EVENTS_BULK_QUEUE 🚀
**Purpose:** Bulk processing of external events  
**Listener:** ExtEventsBulkListener  

**Flow:**
- Multiple external events in one message
- Batch processing for multiple users
- Groups by block ID
- High-throughput execution  

**Retry Queue:** EXT_EVENTS_BULK_DELAY_RETRY_QUEUE  
**TTL:** ~16 minutes  

---

### B. Internal Event Queues

#### INT_EVENTS_QUEUE
**Purpose:** Individual internal journey event processing  
**Listener:** IntEventsListener  

**Flow:**
- Processes journey transitions one at a time
- Executes blocks (wait, decision, engagement)
- Drives journey progression  

**Retry Queue:** INT_EVENTS_DELAY_RETRY_QUEUE  
**TTL:** ~16 minutes  

---

#### INT_EVENTS_BULK_QUEUE 🚀
**Purpose:** Bulk internal journey event processing  
**Listener:** IntEventBulkListener  

**Flow:**
- Batched journey events
- Grouped by block ID
- Incentives, engagements, decisions processed in bulk
- **Primary production execution queue**  

**Retry Queue:** INT_EVENTS_BULK_DELAY_RETRY_QUEUE  
**TTL:** ~16 minutes  
**Concurrency:** internal_events_bulk_queue_max_concurrency  

---

## 2. Wait Block Notification Queues ⏰

#### NOTIFIER_EVENTS_QUEUE
**Purpose:** Time-based wait block notifications  
**Listener:** NotifierEventsListener  

**Flow:**
- Triggered when wait expires
- Moves journeys to next block
- Batched by block ID  

---

#### API_NOTIFIER_EVENTS_QUEUE
**Purpose:** API-triggered wait notifications  
**Listener:** NotifierEventsListener  

**Flow:**
- External systems trigger wait completion
- Similar to NOTIFIER_EVENTS_QUEUE  

---

## 3. Audience Management Queues 👥

#### AUDIENCE_LIST_INGESTION_QUEUE
**Purpose:** Bulk audience ingestion  
**Listener:** AudienceIngestionListener  

**Flow:**
- Bulk import of users
- Creates journey instances  

**Retry Queue:** AUDIENCE_LIST_INGESTION_RETRY_QUEUE  
**TTL:** 10 minutes  

---

#### AUDIENCE_LIST_EGESTION_QUEUE
**Purpose:** Bulk user removal/export  
**Listener:** AudienceEgestionListener  

**Flow:**
- Removes users from journeys
- Handles exits and exports  

**Retry Queue:** AUDIENCE_LIST_EGESTION_RETRY_QUEUE  
**TTL:** 10 minutes  

---

#### AUDIENCE_LIST_RELOAD_NOTIFICATION_QUEUE
**Purpose:** Reload audiences on segment updates  
**Listener:** AudienceListReloadNotificationListener  

**Flow:**
- Syncs journey audiences with updated segments  

**Retry Queue:** AUDIENCE_LIST_RELOAD_NOTIFICATION_RETRY_QUEUE  
**TTL:** 10 minutes  
**Concurrency:** Default 5  

---

## 4. A/B Testing & Control Group Queues 🧪

#### CONTROL_USER_QUEUE
**Purpose:** Control group assignment  
**Listener:** ControlUserListener  

**Flow:**
- Control vs treatment assignment
- Skips actions for control users  

**Retry Queue:** CONTROL_USER_RETRY_QUEUE  
**TTL:** 10 minutes  

---

#### ABTEST_WAIT_NOTIFIER_EXECUTION_QUEUE
**Purpose:** A/B test wait execution  
**Listener:** ABTestWaitNotifierListener  

**Flow:**
- Test-specific waits
- Duration and significance handling  

**Retry Queue:** ABTEST_WAIT_NOTIFIER_EXECUTION_RETRY_QUEUE  
**TTL:** 10 minutes  

---

#### PROCESSED_AB_TEST_ENTITY_QUEUE
**Purpose:** Post-test result processing  
**Listener:** ProcessedAbTestEntityListener  

**Flow:**
- Metric aggregation
- Winner determination  

**Retry Queue:** PROCESSED_AB_TEST_ENTITY_RETRY_QUEUE  
**TTL:** 10 minutes  

---

## 5. Journey Job Execution Queues ⚙️

#### JOURNEY_META_JOB_EXECUTION_QUEUE
**Purpose:** Meta-level journey jobs  
**Listener:** JourneyMetaJobExecutionListener  

**Flow:**
- Expiry, migration, cleanup
- Version transitions  

**Retry Queue:** JOURNEY_META_JOB_EXECUTION_RETRY_QUEUE  
**TTL:** 10 minutes  

---

#### JOURNEY_META_POST_APPROVAL_JOB_EXECUTION_QUEUE
**Purpose:** Post-approval jobs  

**Retry Queue:** JOURNEY_META_POST_APPROVAL_JOB_EXECUTION_RETRY_QUEUE  
**TTL:** 10 minutes  

---

#### JOURNEY_STATUS_UPDATE_JOB_EXECUTION_QUEUE
**Purpose:** Bulk status updates  

**Retry Queue:** JOURNEY_STATUS_UPDATE_JOB_EXECUTION_RETRY_QUEUE  
**TTL:** 10 minutes  

---

#### EMPIRICOR_JOB_EXECUTION_QUEUE
**Purpose:** Empiricor analytics jobs  
**Listener:** EmpiricorJobExecutionListener  

**Retry Queue:** EMPIRICOR_JOB_EXECUTION_RETRY_QUEUE  
**TTL:** 1 minute  

---

## 6. Alert & Monitoring Queues 📢

#### JOURNEY_EXPIRY_ALERT_QUEUE
**Purpose:** Pre-expiry alerts  

---

#### JOURNEY_ALERT_QUEUE
**Purpose:** Failure, SLA, and critical alerts  

---

## 7. Scheduled & Cron Queues 🕐

#### META_SCHEDULER_AND_EXPIRY_CRON_QUEUE
**Purpose:** Scheduled maintenance and expiry  

---

#### STORE_JOB_NOTIFICATION_QUEUE
**Purpose:** External scheduler integration  

---

## 8. Callback & Integration Queues 🔄

#### ADIONA_CALLBACK_QUEUE
**Purpose:** External callbacks  

**Retry Queue:** ADIONA_CALLBACK_RETRY_QUEUE  
**TTL:** 10 minutes  

---

#### API_RESPONSE_QUEUE
**Purpose:** Async API responses  

**Retry Queue:** API_RESPONSE_RETRY_QUEUE  
**TTL:** 5 minutes  

---

## 9. Validation & Checker Queues ✅

#### JOURNEY_CHECKER_QUEUE
**Purpose:** Journey validation  

**Retry Queue:** JOURNEY_CHECKER_RETRY_QUEUE  
**TTL:** 10 minutes  

---

## 10. Failed Events & DLQ ❌

#### FAILED_EVENTS_DLQ
**Purpose:** Final failure destination  
**TTL:** 7 days  
**Listener:** None  

---

## 11. Simulated Testing Queues 🧪

#### QUEUE_SIMULATED_VEYRON_TRIGGER_MESSAGE
**Purpose:** Simulated triggers  
**TTL:** 24 hours  

---

#### QUEUE_SIMULATED_VEYRON_BULK_TRIGGER_MESSAGE
**Purpose:** Bulk simulation and load testing  

---

## Key Insights 🔑

- Bulk queues handle most production load
- TTL-based retries ensure resilience
- Clear separation of ingestion, processing, and execution
- Built-in observability and graceful failure handling

---

🚀 Designed to support **millions of journeys** with **high throughput, scalability, and fault tolerance**.
