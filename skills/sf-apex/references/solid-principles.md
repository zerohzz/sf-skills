<!-- Parent: sf-apex/SKILL.md -->
# SOLID Principles on the Salesforce Platform

## Overview

SOLID principles in Apex must account for Salesforce's unique constraints: governor limits, multitenant architecture, `with sharing` / `without sharing` semantics, and the Trigger → Handler → Service → Selector layered pattern. This guide shows each principle through **Salesforce-specific violations and solutions** — not generic OOP theory.

| Principle | Salesforce-Specific Focus |
|-----------|--------------------------|
| **S**ingle Responsibility | Trigger logic vs business logic separation |
| **O**pen/Closed | Custom Metadata Type-driven extensibility |
| **L**iskov Substitution | Sharing keyword behavioral differences |
| **I**nterface Segregation | `Database.Batchable` composable interfaces |
| **D**ependency Inversion | Trigger → Handler → Service → Selector layers |

---

## S — Single Responsibility on the Salesforce Platform

> On Salesforce, the #1 SRP violation is putting business logic directly in triggers.

### Salesforce Anti-Pattern: The "God Trigger"

```apex
// BAD: Trigger does everything — violates SRP, impossible to test, governor limit nightmare
trigger AccountTrigger on Account (before insert, before update, after insert, after update) {
    if (Trigger.isBefore && Trigger.isInsert) {
        for (Account acc : Trigger.new) {
            // Business logic: normalize phone (reason 1)
            acc.Phone = acc.Phone?.replaceAll('[^0-9+]', '');
            // Validation logic (reason 2)
            if (String.isBlank(acc.Industry)) {
                acc.addError('Industry is required for new accounts');
            }
        }
    }
    if (Trigger.isAfter && Trigger.isUpdate) {
        // Integration logic (reason 3)
        List<Id> changedIds = new List<Id>();
        for (Account acc : Trigger.new) {
            if (acc.AnnualRevenue != Trigger.oldMap.get(acc.Id).AnnualRevenue) {
                changedIds.add(acc.Id);
            }
        }
        if (!changedIds.isEmpty()) {
            // SOQL in trigger context — consuming shared governor limits!
            List<Account> accs = [SELECT Id, Name FROM Account WHERE Id IN :changedIds];
            System.enqueueJob(new AccountSyncQueueable(accs));
        }
    }
}
```

**Problems beyond SRP**:
- Cannot unit test business logic without DML (insert/update fires the trigger)
- Governor limits are consumed in the trigger context — no isolation
- Adding new behavior requires modifying the trigger file directly

### Solution: One Trigger Per Object + Handler Delegation

```apex
// Trigger: ONLY routes events. Zero logic.
trigger AccountTrigger on Account (before insert, before update, after insert, after update) {
    AccountTriggerHandler.run();
}

// Handler: Orchestrates which services to call per event
public with sharing class AccountTriggerHandler {
    public static void run() {
        if (Trigger.isBefore && Trigger.isInsert) {
            AccountValidationService.validateNewAccounts(Trigger.new);
            AccountFieldService.normalizePhones(Trigger.new);
        }
        if (Trigger.isAfter && Trigger.isUpdate) {
            AccountIntegrationService.syncChangedRevenue(Trigger.new, Trigger.oldMap);
        }
    }
}

// Each service has ONE reason to change
public with sharing class AccountValidationService {
    public static void validateNewAccounts(List<Account> accounts) {
        for (Account acc : accounts) {
            if (String.isBlank(acc.Industry)) {
                acc.addError('Industry is required for new accounts');
            }
        }
    }
}
```

**Why this matters on Salesforce specifically**: Apex triggers share governor limits with the entire transaction. Separating logic into services makes it testable independently (with `Test.startTest()` / `Test.stopTest()` resetting limits) and prevents a single trigger from becoming a bottleneck.

> **Industry Pattern**: [Trigger Actions Framework (TAF)](https://github.com/mitchspano/apex-trigger-actions-framework) takes this further — handlers are registered via Custom Metadata, so adding new behavior requires zero code changes to the trigger or handler.

---

## O — Open/Closed on the Salesforce Platform

> On Salesforce, the best way to be "open for extension" is **Custom Metadata Types** — configuration-driven behavior that admins can change without code deployment.

### Salesforce Anti-Pattern: Hardcoded Business Rules

```apex
// BAD: Every new discount type requires Apex code change + deployment
public with sharing class DiscountService {
    public static Decimal calculateDiscount(Account acc, Decimal amount) {
        if (acc.Type == 'Enterprise') {
            return amount * 0.15;
        } else if (acc.Type == 'SMB') {
            return amount * 0.05;
        } else if (acc.Type == 'Partner') {  // Added 3 months later
            return amount * 0.20;
        }
        // Every new type = PR + deploy + test
        return 0;
    }
}
```

### Solution: Custom Metadata Type-Driven Extension

```apex
// Custom Metadata Type: Discount_Rule__mdt
// Fields: Account_Type__c (Text), Discount_Percentage__c (Number)
// Records configured by admins — no code deployment needed

public with sharing class DiscountService {
    // Cache Custom Metadata (free from governor limits — not counted as SOQL)
    private static final Map<String, Discount_Rule__mdt> RULES;

    static {
        RULES = new Map<String, Discount_Rule__mdt>();
        for (Discount_Rule__mdt rule : Discount_Rule__mdt.getAll().values()) {
            RULES.put(rule.Account_Type__c, rule);
        }
    }

    public static Decimal calculateDiscount(Account acc, Decimal amount) {
        Discount_Rule__mdt rule = RULES.get(acc.Type);
        if (rule == null) return 0;
        return amount * (rule.Discount_Percentage__c / 100);
    }
}
```

**Why Custom Metadata is the Salesforce-native OCP solution**:
- `getAll()` doesn't count against SOQL governor limits
- Deployable via metadata API (unlike Custom Settings, which are data)
- Admins can add new types without developer involvement
- Works in test context without `SeeAllData=true`

---

## L — Liskov Substitution on the Salesforce Platform

> On Salesforce, LSP violations often come from **sharing keyword differences** — a `without sharing` subclass behaves fundamentally differently from a `with sharing` parent.

### Salesforce Anti-Pattern: Sharing Keyword Breaks Substitutability

```apex
// Parent: expects all subclasses to respect record-level security
public virtual with sharing class RecordService {
    public virtual List<Account> getAccounts() {
        // Returns only accounts the user can see
        return [SELECT Id, Name FROM Account WITH USER_MODE];
    }
}

// Subclass: SILENTLY bypasses sharing — violates the security contract
public without sharing class AdminRecordService extends RecordService {
    public override List<Account> getAccounts() {
        // Returns ALL accounts — user sees data they shouldn't!
        return [SELECT Id, Name FROM Account];
    }
}

// Consumer trusts the parent's contract
public with sharing class AccountController {
    @AuraEnabled(cacheable=true)
    public static List<Account> getAccounts() {
        RecordService service = getService();  // Could return AdminRecordService!
        return service.getAccounts();  // User might see unauthorized records
    }
}
```

### Solution: Explicit Security Contract

```apex
// Option 1: Use inherited sharing — behavior follows the caller
public virtual inherited sharing class RecordService {
    public virtual List<Account> getAccounts() {
        return [SELECT Id, Name FROM Account WITH USER_MODE];
    }
}

// Option 2: Enforce security at the interface level
public interface SecureDataAccess {
    // Contract: implementations MUST use WITH USER_MODE
    List<SObject> querySecure(String query);
}

// Option 3: Document and enforce the sharing contract
public virtual with sharing class RecordService {
    // Subclasses that change sharing behavior must be in the ALLOWLIST
    private static final Set<Type> ALLOWED_ELEVATED_SERVICES = new Set<Type>{
        AdminRecordService.class  // Explicitly approved
    };

    public virtual List<Account> getAccounts() {
        return [SELECT Id, Name FROM Account WITH USER_MODE];
    }
}
```

**Salesforce-specific LSP rule**: When overriding a `with sharing` method, **never change to `without sharing`** without explicit documentation and Custom Permission checks. The caller expects the parent's security contract.

---

## I — Interface Segregation on the Salesforce Platform

> Salesforce's `Database.Batchable` is the canonical ISP example — you only implement the interfaces you need.

### The Platform's Built-In ISP Design

```apex
// Salesforce provides composable interfaces for batch jobs
// Implement ONLY what you need

// Minimal: just the batchable interface
public class SimpleBatch implements Database.Batchable<SObject> {
    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator('SELECT Id FROM Account');
    }
    public void execute(Database.BatchableContext bc, List<Account> scope) {
        // Process records
    }
    public void finish(Database.BatchableContext bc) { }
}

// Need state between batches? Add Database.Stateful
public class StatefulBatch implements Database.Batchable<SObject>, Database.Stateful {
    private Integer totalProcessed = 0;  // Persists across execute() calls

    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator('SELECT Id FROM Account');
    }
    public void execute(Database.BatchableContext bc, List<Account> scope) {
        totalProcessed += scope.size();
    }
    public void finish(Database.BatchableContext bc) {
        System.debug('Total processed: ' + totalProcessed);
    }
}

// Need HTTP callouts? Add Database.AllowsCallouts
public class CalloutBatch implements Database.Batchable<SObject>, Database.AllowsCallouts {
    // Now can make HTTP callouts in execute()
    // Without this interface, callouts throw System.CalloutException
}

// Need to schedule it? Also implement Schedulable
public class ScheduledBatch implements Database.Batchable<SObject>, Schedulable {
    public void execute(SchedulableContext sc) {
        Database.executeBatch(this, 200);
    }
    // ... batchable methods
}
```

### Applying ISP in Your Own Apex

```apex
// BAD: Fat interface forces unnecessary implementations
public interface RecordHandler {
    void beforeInsert(List<SObject> records);
    void afterInsert(List<SObject> records);
    void beforeUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap);
    void afterUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap);
    void beforeDelete(List<SObject> records);
    void afterDelete(List<SObject> records);
    void afterUndelete(List<SObject> records);
}
// Handler that only needs afterInsert must implement 6 empty methods!

// GOOD: Segregated interfaces — implement only what you handle
public interface BeforeInsertHandler {
    void handleBeforeInsert(List<SObject> records);
}

public interface AfterUpdateHandler {
    void handleAfterUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap);
}

// Implement only what you need
public class AccountRevenueHandler implements AfterUpdateHandler {
    public void handleAfterUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap) {
        // Only fires on after update — no empty methods needed
    }
}
```

> **Platform Insight**: This is exactly how Trigger Actions Framework works — each action class implements only the trigger events it handles (`TriggerAction.BeforeInsert`, `TriggerAction.AfterUpdate`, etc.).

---

## D — Dependency Inversion on the Salesforce Platform

> On Salesforce, DIP is expressed through the **layered architecture**: Trigger → Handler → Service → Selector/Domain. Each layer depends on abstractions, not concrete implementations.

### Salesforce Anti-Pattern: Trigger Directly Calls External API

```apex
// BAD: Trigger (high-level) depends on HTTP details (low-level)
trigger AccountTrigger on Account (after update) {
    for (Account acc : Trigger.new) {
        if (acc.Status__c != Trigger.oldMap.get(acc.Id).Status__c) {
            // Direct dependency on HTTP callout — can't test without mock
            // Also: callouts are NOT allowed in trigger context!
            HttpRequest req = new HttpRequest();
            req.setEndpoint('callout:ERP_System/accounts');
            req.setMethod('POST');
            new Http().send(req);  // Throws System.CalloutException!
        }
    }
}
```

### Solution: Layered Architecture with Dependency Injection

```apex
// Layer 1: Trigger — routes only
trigger AccountTrigger on Account (after update) {
    AccountTriggerHandler.afterUpdate(Trigger.new, Trigger.oldMap);
}

// Layer 2: Handler — orchestrates
public with sharing class AccountTriggerHandler {
    public static void afterUpdate(List<Account> newAccounts, Map<Id, Account> oldMap) {
        List<Account> statusChanged = new List<Account>();
        for (Account acc : newAccounts) {
            if (acc.Status__c != oldMap.get(acc.Id).Status__c) {
                statusChanged.add(acc);
            }
        }
        if (!statusChanged.isEmpty()) {
            // Depends on service abstraction, not HTTP details
            // Uses Queueable to avoid "callout from trigger" governor limit
            System.enqueueJob(new AccountSyncService(statusChanged));
        }
    }
}

// Layer 3: Service — business logic with injected dependencies
public with sharing class AccountSyncService implements Queueable, Database.AllowsCallouts {
    private List<Account> accounts;
    private IERPClient erpClient;

    // Production constructor
    public AccountSyncService(List<Account> accounts) {
        this(accounts, new ERPClient());
    }

    // Test constructor — dependency injection
    @TestVisible
    private AccountSyncService(List<Account> accounts, IERPClient client) {
        this.accounts = accounts;
        this.erpClient = client;
    }

    public void execute(QueueableContext ctx) {
        for (Account acc : accounts) {
            erpClient.syncAccount(acc);  // Depends on abstraction
        }
    }
}

// Abstraction
public interface IERPClient {
    void syncAccount(Account acc);
}

// Low-level implementation
public class ERPClient implements IERPClient {
    public void syncAccount(Account acc) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:ERP_System/accounts');
        req.setMethod('POST');
        req.setBody(JSON.serialize(acc));
        new Http().send(req);
    }
}
```

> **Industry Standard**: The [fflib-apex-common](https://github.com/apex-enterprise-patterns/fflib-apex-common) library (Apex Enterprise Patterns) provides a complete implementation of this layered architecture with `fflib_Application` for dependency injection, `fflib_SObjectSelector` for the selector layer, `fflib_SObjectDomain` for the domain layer, and `fflib_SObjectUnitOfWork` for transactional DML.

---

## Summary: SOLID on Salesforce

| Principle | Generic Textbook | Salesforce Platform Reality |
|-----------|----------------|-----------------------------|
| **SRP** | "One reason to change" | One trigger per object, business logic in services, not triggers |
| **OCP** | "Use interfaces" | Use **Custom Metadata Types** for config-driven extension |
| **LSP** | "Subtypes substitute" | **Sharing keywords** must preserve parent's security contract |
| **ISP** | "Small interfaces" | Platform models this: `Database.Batchable` + `Stateful` + `AllowsCallouts` |
| **DIP** | "Depend on abstractions" | Trigger → Handler → Service → Selector layered architecture |

---

## Official References

- **Apex Developer Guide**: [Trigger Best Practices](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_bestpract.htm)
- **Trailhead**: [Apex Triggers](https://trailhead.salesforce.com/content/learn/modules/apex_triggers)
- **Trailhead**: [Asynchronous Apex](https://trailhead.salesforce.com/content/learn/modules/asynchronous_apex)
- **Apex Enterprise Patterns**: [fflib-apex-common](https://github.com/apex-enterprise-patterns/fflib-apex-common)
- **Trigger Actions Framework**: [TAF GitHub](https://github.com/mitchspano/apex-trigger-actions-framework)
- **Custom Metadata Types**: [Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_methods_system_custom_metadata_types.htm)
- **Sharing and Security**: [Apex Security Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_security_sharing_chapter.htm)
