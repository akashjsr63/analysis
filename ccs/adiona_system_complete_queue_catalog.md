# Adiona System – Complete Queue Catalog

This document provides a comprehensive overview of the **Adiona Queue Architecture**, detailing all queues, their purposes, listeners, flows, retry mechanisms, and architectural patterns. It serves as a single reference for understanding how Adiona achieves high-throughput, fault-tolerant journey execution at scale.

---

## 1. Core Event Processing Queues ⚡

### A. External Event Queues

#### EXT_EVENTS_INGESTION_QUEUE
**Purpose:** First entry point for external events from outside systems  
**Listener:** `ExtEventsIngestionListener`

**Flow:**
- Receives raw external events (user actions, transactions, etc.)
- Validates and processes event ingestion
- Routes events to appropriate journey processing queues

**Retry Queue:** `EXT_EVENTS_INGESTION_DELAY_RETRY_QUEUE`  
**TTL:** ~16 minutes

---

#### EXT_EVENTS_QUEUE
**Purpose:** Individual external event processing  
**Listener:** `ExtEventsListener`

**Flow:**
- Processes one external event at a time
- Evaluates event relevancy for journeys
- Creates/updates journey state based on external events

**Retry Queue:** `EXT_EVENTS_DELAY_RETRY_QUEUE`  
**TTL:** ~16 minutes

---

#### EXT_EVENTS_BULK_QUEUE 🚀
**Purpose:** Bulk processing of external events  
**Listener:** `ExtEventsBulkListener`

**Flow:**
- Multiple external events in a single message
- Batch processing for multiple users
- Groups events by block ID for efficient execution
- Higher throughput than individual processing

**Retry Queue:** `EXT_EVENTS_BULK_DELAY_RETRY_QUEUE`  
**TTL:** ~16 minutes

---

### B. Internal Event Queues

#### INT_EVENTS_QUEUE
**Purpose:** Individual internal journey event processing  
**Listener:** `IntEventsListener`

**Flow:**
- Processes journey state transitions one at a time
- Executes blocks (wait, decision, engagement, etc.)
- Drives journey progression through the workflow

**Retry Queue:** `INT_EVENTS_DELAY_RETRY_QUEUE`  
**TTL:** ~16 minutes

---

#### INT_EVENTS_BULK_QUEUE 🚀
**Purpose:** Bulk internal journey event processing  
**Listener:** `IntEventBulkListener`

**Flow:**
- Multiple journey events batched together
- Groups events by block ID
- Processes incentives, engagements, and decisions in bulk
- **Primary queue for scalable journey execution**

**Retry Queue:** `INT_EVENTS_BULK_DELAY_RETRY_QUEUE`  
**TTL:** ~16 minutes  
**Consumer Concurrency:** Configurable via `internal_events_bulk_queue_max_concurrency`

---

## 2. Wait Block Notification Queues ⏰

#### NOTIFIER_EVENTS_QUEUE
**Purpose:** Time-based wait block notifications  
**Listener:** `NotifierEventsListener`

**Flow:**
- Triggered when wait block time expires
- Processes timed-out wait notifications
- Moves journeys from wait state to the next block
- Batches notifications by block ID

---

#### API_NOTIFIER_EVENTS_QUEUE
**Purpose:** API-triggered wait notifications  
**Listener:** `NotifierEventsListener`

**Flow:**
- Handles API-based wait block completions
- External systems trigger wait completion events
- Similar processing model to `NOTIFIER_EVENTS_QUEUE`

---

## 3. Audience Management Queues 👥

#### AUDIENCE_LIST_INGESTION_QUEUE
**Purpose:** Bulk audience/user list ingestion into journeys  
**Listener:** `AudienceIngestionListener`

**Flow:**
- Bulk import of users into journeys
- Creates journey instances for multiple users
- Entry point for audience segmentation

**Retry Queue:** `AUDIENCE_LIST_INGESTION_RETRY_QUEUE`  
**TTL:** 10 minutes

---

#### AUDIENCE_LIST_EGESTION_QUEUE
**Purpose:** Export/removal of users from journeys  
**Listener:** `AudienceEgestionListener`

**Flow:**
- Removes users from journeys in bulk
- Handles journey exits
- Exports audience data to external systems

**Retry Queue:** `AUDIENCE_LIST_EGESTION_RETRY_QUEUE`  
**TTL:** 10 minutes

---

#### AUDIENCE_LIST_RELOAD_NOTIFICATION_QUEUE
**Purpose:** Reload audience lists when updated  
**Listener:** `AudienceListReloadNotificationListener`

**Flow:**
- Triggered when audience segment data changes
- Reloads user membership in journeys
- Synchronizes journeys with updated segments

**Retry Queue:** `AUDIENCE_LIST_RELOAD_NOTIFICATION_RETRY_QUEUE`  
**TTL:** 10 minutes  
**Concurrency:** Configurable (default: 5)

---

## 4. A/B Testing & Control Group Queues 🧪

#### CONTROL_USER_QUEUE
**Purpose:** Control group user management  
**Listener:** `ControlUserListener`

**Flow:**
- Assigns users to control vs treatment groups
- Ensures control users skip certain actions (e.g., incentives)
- Handles A/B test randomization

**Retry Queue:** `CONTROL_USER_RETRY_QUEUE`  
**TTL:** 10 minutes

---

#### ABTEST_WAIT_NOTIFIER_EXECUTION_QUEUE
**Purpose:** A/B test-specific wait notification execution  
**Listener:** `ABTestWaitNotifierListener`

**Flow:**
- Processes wait blocks for A/B tests
- Manages test duration
- Performs statistical significance checks

**Retry Queue:** `ABTEST_WAIT_NOTIFIER_EXECUTION_RETRY_QUEUE`  
**TTL:** 10 minutes

---

#### PROCESSED_AB_TEST_ENTITY_QUEUE
**Purpose:** Post-processing of A/B test results  
**Listener:** `ProcessedAbTestEntityListener`

**Flow:**
- Calculates A/B test metrics
- Aggregates test performance data
- Determines winning variants

**Retry Queue:** `PROCESSED_AB_TEST_ENTITY_RETRY_QUEUE`  
**TTL:** 10 minutes

---

## 5. Journey Job Execution Queues ⚙️

#### JOURNEY_META_JOB_EXECUTION_QUEUE
**Purpose:** Journey metadata-level job execution  
**Listener:** `JourneyMetaJobExecutionListener`

**Flow:**
- Executes scheduled jobs at the journey meta level
- Handles journey expiration, migration, and cleanup
- Manages version upgrades and transitions

**Retry Queue:** `JOURNEY_META_JOB_EXECUTION_RETRY_QUEUE`  
**TTL:** 10 minutes

---

#### JOURNEY_META_POST_APPROVAL_JOB_EXECUTION_QUEUE
**Purpose:** Post-approval job execution  
**Listener:** `JourneyMetaJobExecutionListener`

**Flow:**
- Executes jobs triggered after journey approval
- Activation workflows
- Post-deployment tasks

**Retry Queue:** `JOURNEY_META_POST_APPROVAL_JOB_EXECUTION_RETRY_QUEUE`  
**TTL:** 10 minutes

---

#### JOURNEY_STATUS_UPDATE_JOB_EXECUTION_QUEUE
**Purpose:** Journey status update jobs  
**Listener:** `JourneyMetaJobExecutionListener`

**Flow:**
- Bulk journey status changes (pause, resume, stop)
- Lifecycle state transitions
- Status synchronization

**Retry Queue:** `JOURNEY_STATUS_UPDATE_JOB_EXECUTION_RETRY_QUEUE`  
**TTL:** 10 minutes

---

#### EMPIRICOR_JOB_EXECUTION_QUEUE
**Purpose:** Empiricor (A/B testing engine) job execution  
**Listener:** `EmpiricorJobExecutionListener`

**Flow:**
- Empiricor-specific analytics jobs
- Test result calculations
- Statistical processing

**Retry Queue:** `EMPIRICOR_JOB_EXECUTION_RETRY_QUEUE`  
**TTL:** 1 minute

---

## 6. Alert & Monitoring Queues 📢

#### JOURNEY_EXPIRY_ALERT_QUEUE
**Purpose:** Alerts for expiring journeys  
**Listener:** `JourneyExpiryAlertListener`

**Flow:**
- Triggers alerts before journey expiration
- Sends admin notifications
- Enables proactive expiry management

---

#### JOURNEY_ALERT_QUEUE
**Purpose:** General journey alerts  
**Listener:** `JourneyAlertListener`

**Flow:**
- Journey failure alerts
- SLA breach notifications
- Critical system and journey alerts

---

## 7. Scheduled & Cron Queues 🕐

#### META_SCHEDULER_AND_EXPIRY_CRON_QUEUE
**Purpose:** Scheduled cron jobs for journey management  
**Listener:** `MetaScheduledAndExpiryCronEventsProcessor`

**Flow:**
- Periodic journey expiry checks
- Scheduled job triggers
- Time-based journey maintenance

---

#### STORE_JOB_NOTIFICATION_QUEUE (`adiona_api_srv_triggers`)
**Purpose:** Store job notifications from external systems

**Flow:**
- Integration with external job schedulers
- External trigger processing
- Cross-system job coordination

---

## 8. Callback & Integration Queues 🔄

#### ADIONA_CALLBACK_QUEUE
**Purpose:** Callbacks from external systems  
**Listener:** `AbTestEventListener`

**Flow:**
- Webhook responses
- External acknowledgments
- Asynchronous operation completions

**Retry Queue:** `ADIONA_CALLBACK_RETRY_QUEUE`  
**TTL:** 10 minutes

---

#### API_RESPONSE_QUEUE
**Purpose:** API response processing  
**Listener:** `ApiResponseProcessor`

**Flow:**
- Processes async API call responses
- Handles external service replies
- Enables decoupled API communication

**Retry Queue:** `API_RESPONSE_RETRY_QUEUE`  
**TTL:** 5 minutes

---

## 9. Validation & Checker Queues ✅

#### JOURNEY_CHECKER_QUEUE
**Purpose:** Journey validation and health checks  
**Listener:** `JourneyCheckerListener`

**Flow:**
- Journey consistency checks
- Data integrity validation
- Journey state verification

**Retry Queue:** `JOURNEY_CHECKER_RETRY_QUEUE`  
**TTL:** 10 minutes

---

## 10. Failed Events & Dead Letter Queue ❌

#### FAILED_EVENTS_DLQ
**Purpose:** Final destination for failed events

**Details:**
- Events that exhaust all retries
- Requires manual intervention
- Used for audit, debugging, and recovery

**TTL:** 7 days (604,800,000 ms)  
**Listener:** None (manual processing only)

---

## 11. Simulated Testing Queues 🧪

#### QUEUE_SIMULATED_VEYRON_TRIGGER_MESSAGE
**Purpose:** Simulated communication triggers for testing

**Flow:**
- Test environment message simulation
- Veyron communication system testing

**TTL:** 24 hours

---

#### QUEUE_SIMULATED_VEYRON_BULK_TRIGGER_MESSAGE
**Purpose:** Bulk simulated communication triggers

**Flow:**
- Bulk message testing
- Performance and load testing

---

## Queue Architecture Patterns 🎯

### Primary–Retry Pattern
- Each main queue has a corresponding retry queue
- Retry queues use TTL-based delayed retries
- Dead Letter Exchange routes messages back after TTL
- Handles transient failures (network issues, service unavailability)

### Bulk vs Individual Processing

**Individual Queues** (`EXT_EVENTS_QUEUE`, `INT_EVENTS_QUEUE`):
- One message per user/journey
- Lower throughput
- Simpler, user-specific logic

**Bulk Queues** (`EXT_EVENTS_BULK_QUEUE`, `INT_EVENTS_BULK_QUEUE`):
- One message for multiple users/journeys
- High throughput and batch efficiency
- Grouped by block ID
- Primary production execution model

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

## Queue Flow Summary 📊

```
External Events
  → EXT_EVENTS_INGESTION_QUEUE
      → [Validation & Routing]
          → EXT_EVENTS_QUEUE / EXT_EVENTS_BULK_QUEUE
              → [Journey State Processing]
                  → INT_EVENTS_QUEUE / INT_EVENTS_BULK_QUEUE
                      → [Block Execution: Incentives, Wait, Engagement]
                          → Next Block
                              → INT_EVENTS_BULK_QUEUE (loop)
                                  → Exit or FAILED_EVENTS_DLQ
```

---

## Key Insights 🔑

- **Bulk Queues are King:** `INT_EVENTS_BULK_QUEUE` and `EXT_EVENTS_BULK_QUEUE` handle most production traffic
- **Resilient by Design:** Retry queues with TTL enable graceful recovery
- **Clear Separation of Concerns:** Ingestion → Validation → Processing → Execution
- **A/B Testing First-Class Support:** Dedicated queues for experimentation and analytics
- **Observability Built-In:** Alert queues provide proactive monitoring
- **Graceful Degradation:** DLQ ensures failures are isolated and debuggable

---

🚀 This queue architecture enables **millions of journey executions** with **high throughput, fault tolerance, and maintainability**.

