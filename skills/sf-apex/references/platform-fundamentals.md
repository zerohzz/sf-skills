<!-- Parent: sf-apex/SKILL.md -->
# Salesforce Platform Fundamentals for Apex Developers

This reference covers Salesforce-specific concepts that don't exist on other platforms. Understanding these is essential for writing correct Apex code.

---

## Multitenant Architecture: Why Governor Limits Exist

Salesforce runs a **multitenant architecture** — your org shares compute, storage, and database resources with thousands of other orgs on the same pod. Governor limits exist to prevent any single tenant from monopolizing shared resources.

**What this means for Apex developers:**
- You cannot optimize away governor limits — they are enforced by the platform
- There is no way to "upgrade the server" for your org's Apex code
- Every SOQL query, DML statement, and CPU cycle comes from a shared pool
- Background (async) contexts get higher limits because they run in dedicated threads

**Design implications:**
- Batch processing for large data volumes (Batch Apex, Queueable chains)
- Collection-based patterns instead of record-by-record processing
- Strategic use of Platform Cache to reduce SOQL queries
- Careful async boundary design to maximize available limits

---

## Trigger Execution Order

When a record is saved in Salesforce, the platform executes operations in a **strict order**. Understanding this order is critical for debugging and designing automation that doesn't conflict.

### The Complete Execution Order

```
1.  Load original record from database (or initialize for insert)
2.  Load new field values from request
3.  Execute all BEFORE triggers
4.  Run system validation rules (required fields, field formats)
5.  Execute duplicate rules
6.  Save record to database (but don't commit)
7.  Execute all AFTER triggers
8.  Execute assignment rules
9.  Execute auto-response rules
10. Execute workflow rules
11. If workflow field updates exist, fire BEFORE and AFTER triggers AGAIN
12. Execute escalation rules
13. Execute Flow automations (record-triggered)
    - Before-save flows run in step 3 (before validation)
    - After-save flows run here
14. Execute entitlement rules
15. Roll-up summary field calculations
16. Cross-object formula field calculations
17. Repeat steps 7-16 for cascading updates
18. Execute post-commit logic (sending emails, async jobs)
19. COMMIT transaction to database
```

### Critical Implications

| Implication | Why It Matters |
|-------------|----------------|
| Before triggers run BEFORE validation rules | You can fix data in before triggers to pass validation |
| After triggers run BEFORE Flows | Apex after triggers see the record before Flow changes |
| Workflow field updates re-fire triggers | Can cause infinite loops if not guarded |
| All automation shares one transaction | Governor limits are cumulative across triggers + flows + workflow |
| Record-triggered Flows fire after all triggers | Apex trigger changes are visible to Flows |
| Commit happens at the end | A failure anywhere rolls back everything |

### Preventing Infinite Loops

```apex
// Use a static variable to prevent re-entry
public class TriggerRecursionGuard {
    private static Set<Id> processedIds = new Set<Id>();

    public static Boolean hasBeenProcessed(Id recordId) {
        return processedIds.contains(recordId);
    }

    public static void markProcessed(Id recordId) {
        processedIds.add(recordId);
    }

    public static void markProcessed(Set<Id> recordIds) {
        processedIds.addAll(recordIds);
    }
}

// In trigger handler:
public static void afterUpdate(List<Account> newAccounts, Map<Id, Account> oldMap) {
    List<Account> toProcess = new List<Account>();
    for (Account acc : newAccounts) {
        if (!TriggerRecursionGuard.hasBeenProcessed(acc.Id)) {
            toProcess.add(acc);
            TriggerRecursionGuard.markProcessed(acc.Id);
        }
    }
    if (!toProcess.isEmpty()) {
        // Process only unprocessed records
    }
}
```

---

## Asynchronous Apex Decision Matrix

Salesforce provides four async mechanisms, each with different governor limits, use cases, and constraints.

| Feature | Queueable | Batch Apex | Schedulable | Platform Events |
|---------|-----------|------------|-------------|-----------------|
| **Max records** | 50,000 rows/job | 50M rows (QueryLocator) | N/A (launches other jobs) | 10M events/day (standard) |
| **Governor limits** | Async (double sync) | Async per execute() | Sync in execute() | Async per trigger |
| **Chaining** | Yes (1 child job) | No | Can launch Batch/Queueable | Trigger fires new transaction |
| **Callouts** | Yes (with `Database.AllowsCallouts`) | Yes (with `Database.AllowsCallouts`) | No (launch async instead) | No (in trigger context) |
| **Delay/schedule** | No built-in delay | Can schedule via Schedulable | Cron expression | Near-real-time |
| **State** | Instance variables | `Database.Stateful` interface | None (stateless) | Event payload fields |
| **Transaction isolation** | New transaction | New transaction per execute() | New transaction | New transaction |
| **Retry on failure** | No auto-retry | No auto-retry | Re-schedules on schedule | Platform retries failed subscriptions |
| **Use when** | Standard async work, callouts | Large data volume processing | Recurring scheduled jobs | Decoupled event-driven processing |

### Decision Flowchart

```
Is it a one-time operation?
├── YES: Does it need to process >50K records?
│   ├── YES → Batch Apex
│   └── NO: Does it need HTTP callouts?
│       ├── YES → Queueable (with Database.AllowsCallouts)
│       └── NO → Queueable
└── NO (recurring):
    ├── Recurring on a schedule → Schedulable (launches Batch or Queueable)
    └── Event-driven / decoupled → Platform Events
```

### Governor Limit Comparison

| Limit | Synchronous | Queueable / Batch execute() |
|-------|-------------|----------------------------|
| SOQL Queries | 100 | 200 |
| DML Statements | 150 | 150 |
| CPU Time | 10,000 ms | 60,000 ms |
| Heap Size | 6 MB | 12 MB |
| Callout Time | 120s total | 120s total |
| Callout Count | 100 | 100 |

---

## Static Variables and Transaction Lifecycle

On Salesforce, **static variables persist for the entire transaction** but are reset between transactions. This is fundamentally different from application-wide singletons in traditional applications.

```apex
public class TransactionState {
    // This persists across ALL trigger executions in the SAME transaction
    // But resets when the transaction commits
    public static Integer queryCount = 0;

    // This also resets between transactions — no application-level state
    private static Map<String, Object> cache = new Map<String, Object>();
}
```

**Implications:**
- Singleton patterns in Apex are transaction-scoped, not application-scoped
- Static variables are ideal for recursion guards and in-transaction caching
- Platform Cache (`Cache.Org` / `Cache.Session`) is needed for cross-transaction state
- Custom Settings (`getOrgDefaults()`) are cached per-transaction automatically

---

## Platform Cache

For data that shouldn't be re-queried every transaction:

```apex
// Org-level cache — shared across all users (up to 10MB standard, 30MB with add-on)
Cache.OrgPartition orgPart = Cache.Org.getPartition('local.MyPartition');
orgPart.put('key', value, 7200);  // TTL in seconds
Object cached = orgPart.get('key');

// Session-level cache — per user session (up to 8MB)
Cache.SessionPartition sessionPart = Cache.Session.getPartition('local.MyPartition');
sessionPart.put('userPrefs', prefs);
```

**When to use Platform Cache vs Custom Settings vs Custom Metadata:**
| Data Type | Best Storage | Why |
|-----------|-------------|-----|
| Org configuration (rarely changes) | Custom Metadata Type | Deployable, cached, no SOQL cost |
| User-specific settings | Custom Setting (Hierarchy) | Cached per-transaction, user-specific |
| Frequently accessed reference data | Platform Cache (Org) | Cross-transaction, configurable TTL |
| Session-specific state | Platform Cache (Session) | Per-user, survives page navigations |

---

## Official References

- **Execution Governors and Limits**: [Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- **Trigger Execution Order**: [Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm)
- **Asynchronous Apex**: [Trailhead](https://trailhead.salesforce.com/content/learn/modules/asynchronous_apex)
- **Platform Cache**: [Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_cache_namespace_overview.htm)
- **Multitenant Architecture**: [Architect Guide](https://architect.salesforce.com/fundamentals/platform-multitenant-architecture)
