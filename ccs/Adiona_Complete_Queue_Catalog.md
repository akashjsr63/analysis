# Adiona System – Complete Queue Catalog

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

This diagram illustrates the end-to-end lifecycle of external events through ingestion, validation, journey execution, block processing, retry loops, and final exit or DLQ handling.
