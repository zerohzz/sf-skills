<!-- Parent: sf-soql/SKILL.md -->

# SOQL Query Optimization & Governor Limits

This guide covers Salesforce-specific query optimization — not generic SQL tuning. Every technique here addresses the platform's multitenant architecture, governor limits, and the query optimizer's behavior.

## Why Salesforce Query Optimization is Different

Salesforce runs on a **multitenant architecture** where your queries share compute resources with every other tenant. The platform enforces strict limits to protect all tenants:

- **100 SOQL queries per synchronous transaction** (200 asynchronous)
- **50,000 rows retrieved per transaction**
- **Non-selective queries on large objects (>200K records) are blocked by the query optimizer**

There is no `CREATE INDEX` command. You work with the platform's existing indexes and request custom indexes via Salesforce Support.

---

## Indexing: What's Automatically Indexed

**Standard Indexed Fields** (all objects):
- `Id`, `Name`, `OwnerId`, `CreatedDate`, `LastModifiedDate`, `RecordTypeId`, `SystemModstamp`
- External ID fields (custom fields marked as External ID)
- Master-Detail relationship fields
- Lookup fields (selectively indexed)

**Object-Specific Standard Indexes**:
| Object | Indexed Fields |
|--------|---------------|
| Account | AccountNumber, Site |
| Contact | Email |
| Lead | Email, Name |
| Case | CaseNumber |
| Opportunity | CloseDate |

**Custom Indexes** (request via Salesforce Support):
- Single-column custom index
- Two-column composite index
- Skinny table index (for high-volume objects >1M records)

---

## Selectivity: How the Query Optimizer Decides

The Salesforce query optimizer evaluates filters **before execution** and decides between an Index Scan or Table Scan.

### Selectivity Thresholds

```
A filter is SELECTIVE when it returns:
  - Less than 10% of total records (for first 1M records)
  - Less than 5% of total records (beyond 1M records)
  - AND the filter uses an indexed field

A filter is NON-SELECTIVE when:
  - It exceeds the threshold above
  - OR uses a non-indexed field
  - OR uses a leading wildcard (LIKE '%value')
  - OR uses a negative operator on a non-indexed field (!=, NOT IN, EXCLUDES)
```

### Selective vs Non-Selective Examples

```sql
-- NON-SELECTIVE: Status is not indexed, likely >10% of records
SELECT Id FROM Lead WHERE Status = 'Open'

-- SELECTIVE: CreatedDate is indexed + date range is selective
SELECT Id FROM Lead
WHERE Status = 'Open'
AND CreatedDate = LAST_N_DAYS:30

-- NON-SELECTIVE: Leading wildcard prevents index usage
SELECT Id FROM Account WHERE Name LIKE '%corp'

-- SELECTIVE: Trailing wildcard uses the Name index
SELECT Id FROM Account WHERE Name LIKE 'Acme%'

-- SELECTIVE: Id is always indexed
SELECT Id, Name FROM Account WHERE Id IN :accountIds

-- SELECTIVE: External ID is always indexed
SELECT Id FROM Account WHERE External_Id__c = 'EXT-12345'
```

---

## Query Plan Analysis

Use the Developer Console or the Tooling API to analyze query plans:

### Developer Console Method
1. Open Developer Console → Query Editor
2. Check "Use Tooling API"
3. Run: `EXPLAIN SELECT Id FROM Account WHERE Name = 'Test'`

### CLI Method (Tooling API)

```bash
# Query plan via Tooling API REST endpoint
sf data query \
  --query "EXPLAIN SELECT Id FROM Account WHERE Name = 'Test'" \
  --target-org my-org \
  --use-tooling-api
```

**Note**: The `EXPLAIN` keyword is used in the Tooling API's Query resource. This is different from standard SOQL.

### Interpreting Query Plan Output

| Field | Meaning | Good Value |
|-------|---------|------------|
| `Cardinality` | Estimated rows returned | Low relative to total records |
| `Cost` | Relative query cost | < 1.0 preferred |
| `Fields` | Index fields used | Should list your filter fields |
| `LeadingOperationType` | Index vs TableScan | **Index** = good, **TableScan** = bad |
| `SObjectCardinality` | Total records in object | Context for Cost evaluation |

---

## Governor Limits Reference

| Limit | Synchronous | Asynchronous |
|-------|-------------|--------------|
| Total SOQL Queries | 100 | 200 |
| Records Retrieved (total per transaction) | 50,000 | 50,000 |
| SOQL FOR loop batch size (queryMore) | 200 per batch | 200 per batch |
| Query Locator Rows (Batch Apex start()) | 50 million | 50 million |
| SOSL Queries | 20 | 20 |
| DML Statements | 150 | 150 |
| DML Rows | 10,000 | 10,000 |
| CPU Time | 10,000 ms | 60,000 ms |
| Heap Size | 6 MB | 12 MB |

---

## Optimization Patterns

### 1. Never Query Inside a Loop

```apex
// BAD: N SOQL queries (1 per contact) — hits 100 limit fast
for (Contact c : contacts) {
    Account a = [SELECT Name FROM Account WHERE Id = :c.AccountId];
}

// GOOD: 1 SOQL query regardless of list size
Set<Id> accountIds = new Set<Id>();
for (Contact c : contacts) {
    accountIds.add(c.AccountId);
}
Map<Id, Account> accountMap = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN :accountIds WITH USER_MODE]
);
for (Contact c : contacts) {
    Account a = accountMap.get(c.AccountId);
}
```

### 2. Use SOQL FOR Loops for Large Datasets

```apex
// BAD: Loads all records into heap — risks 6MB heap limit
List<Account> allAccounts = [SELECT Id, Name FROM Account WHERE Industry = 'Technology'];
// If 50,000 records, each ~200 bytes = ~10MB → HEAP OVERFLOW

// GOOD: Processes 200 records at a time via internal queryMore
for (List<Account> batch : [SELECT Id, Name FROM Account WHERE Industry = 'Technology']) {
    // batch.size() is up to 200
    // Old batch is garbage collected before next batch loads
    processBatch(batch);
}
```

**Platform detail**: SOQL FOR loops use `queryMore()` internally, fetching 200 records per batch. The previous batch is eligible for garbage collection, keeping heap usage constant.

### 3. Query Only the Fields You Need

```apex
// BAD: SELECT * equivalent — wastes heap and transfer time
List<Account> accounts = [SELECT FIELDS(ALL) FROM Account LIMIT 200];

// GOOD: Only the fields you'll actually use
List<Account> accounts = [
    SELECT Id, Name, Industry, AnnualRevenue
    FROM Account
    WHERE Industry = 'Technology'
    WITH USER_MODE
    LIMIT 200
];
```

### 4. Use Aggregate Queries to Avoid Row Limits

```apex
// BAD: Retrieve all records just to count them
List<Opportunity> opps = [SELECT Id FROM Opportunity WHERE StageName = 'Closed Won'];
Integer count = opps.size();  // Wastes 50,000 row budget

// GOOD: Aggregate doesn't count against 50,000 row limit (counts as 1 row)
Integer count = [SELECT COUNT() FROM Opportunity WHERE StageName = 'Closed Won'];

// GOOD: GROUP BY for multi-dimensional aggregation
List<AggregateResult> results = [
    SELECT StageName, COUNT(Id) cnt, SUM(Amount) total
    FROM Opportunity
    WHERE CloseDate = THIS_FISCAL_YEAR
    GROUP BY StageName
    WITH USER_MODE
];
```

### 5. Relationship Queries to Reduce SOQL Count

```apex
// BAD: 2 separate queries (wastes 1 of your 100 SOQL budget)
List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :ids];
List<Contact> contacts = [SELECT Id, FirstName, AccountId FROM Contact WHERE AccountId IN :ids];

// GOOD: 1 query with child subquery
List<Account> accounts = [
    SELECT Id, Name,
        (SELECT Id, FirstName, LastName, Email FROM Contacts)
    FROM Account
    WHERE Id IN :ids
    WITH USER_MODE
];

// GOOD: Parent-to-child field access (no extra SOQL)
List<Contact> contacts = [
    SELECT Id, FirstName, Account.Name, Account.Industry
    FROM Contact
    WHERE AccountId IN :ids
    WITH USER_MODE
];
```

---

## Security Modes (API 60.0+)

### WITH USER_MODE (Recommended Default)

```apex
// Enforces CRUD, FLS, AND sharing rules — the most secure option
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue
    FROM Account
    WHERE Industry = 'Technology'
    WITH USER_MODE
];
// Throws System.QueryException if user lacks field/object access
```

**Use `WITH USER_MODE` for**:
- All user-facing queries (LWC controllers, Aura, Visualforce)
- Service methods handling user requests
- Any query where the running user's permissions should apply

### WITH SYSTEM_MODE (Use with Justification)

```apex
// Bypasses CRUD, FLS, and sharing rules
// ALWAYS document why this is needed
List<Account> accounts = [
    SELECT Id, Name, Sensitive_Field__c
    FROM Account
    WITH SYSTEM_MODE
];
```

**Use `WITH SYSTEM_MODE` only for**:
- Background/batch jobs that must process all records
- System integrations where no user context applies
- Administrative utilities (with Custom Permission guard)

### Legacy: WITH SECURITY_ENFORCED (Pre-API 60.0)

```apex
// Older approach — enforces FLS but not sharing rules
// Use WITH USER_MODE instead for new code
List<Account> accounts = [
    SELECT Id, Name
    FROM Account
    WITH SECURITY_ENFORCED
];
```

### Legacy: Security.stripInaccessible()

```apex
// For pre-API 60.0 or when you need to silently strip fields instead of throwing
SObjectAccessDecision decision = Security.stripInaccessible(
    AccessType.READABLE,
    [SELECT Id, Name, SecretField__c FROM Account]
);
List<Account> safeAccounts = decision.getRecords();
```

---

## SOQL Injection Prevention

### The Threat

```apex
// VULNERABLE: User input directly concatenated into SOQL
public static List<Account> search(String userInput) {
    String query = 'SELECT Id, Name FROM Account WHERE Name = \'' + userInput + '\'';
    return Database.query(query);
    // Attack: userInput = "x' OR Name LIKE '%"  → returns ALL accounts
}
```

### Prevention: Bind Variables (Always Preferred)

```apex
// SAFE: Bind variables are never interpreted as SOQL
public static List<Account> search(String userInput) {
    return [SELECT Id, Name FROM Account WHERE Name = :userInput WITH USER_MODE];
}
```

### Prevention: String.escapeSingleQuotes() (Dynamic SOQL)

```apex
// SAFE: For dynamic SOQL when bind variables aren't possible
public static List<SObject> dynamicSearch(String objectName, String userInput) {
    // Validate object name against allowlist
    Set<String> allowedObjects = new Set<String>{'Account', 'Contact', 'Lead'};
    if (!allowedObjects.contains(objectName)) {
        throw new IllegalArgumentException('Invalid object: ' + objectName);
    }

    String sanitized = String.escapeSingleQuotes(userInput);
    String query = 'SELECT Id, Name FROM ' + objectName +
                   ' WHERE Name LIKE \'%' + sanitized + '%\' WITH USER_MODE LIMIT 100';
    return Database.query(query);
}
```

### Prevention: Allowlist for Dynamic Field Names

```apex
// SAFE: Validate field names against Schema describe
public static List<Account> sortedQuery(String sortField) {
    Map<String, Schema.SObjectField> fieldMap =
        Schema.SObjectType.Account.fields.getMap();

    if (!fieldMap.containsKey(sortField.toLowerCase())) {
        throw new IllegalArgumentException('Invalid field: ' + sortField);
    }

    String query = 'SELECT Id, Name FROM Account ORDER BY ' + sortField + ' WITH USER_MODE LIMIT 100';
    return Database.query(query);
}
```

---

## Large Data Volume (LDV) Strategies

For objects with >1M records:

| Strategy | When to Use | How to Request |
|----------|-------------|----------------|
| **Custom Index** | Frequently filtered custom field | Salesforce Support case |
| **Two-Column Index** | Queries filtering on 2 fields together | Salesforce Support case |
| **Skinny Table** | High-volume queries on subset of fields | Salesforce Support case |
| **Archive to Big Objects** | Historical data rarely queried | Implementation project |
| **Batch Apex** | Processing all records | Use `Database.QueryLocator` (50M rows) |

---

## Official References

- **SOQL/SOSL Reference**: [Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.soql_sosl.meta/soql_sosl/)
- **Trailhead**: [Apex Basics & Database](https://trailhead.salesforce.com/content/learn/modules/apex_database)
- **Trailhead**: [Search Solution Basics](https://trailhead.salesforce.com/content/learn/modules/search_solution_basics)
- **Query Plan Tool**: [Tooling API Reference](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_queryplan.htm)
- **Large Data Volumes**: [Best Practices Guide](https://developer.salesforce.com/docs/atlas.en-us.salesforce_large_data_volumes_bp.meta/salesforce_large_data_volumes_bp/)
- **Governor Limits**: [Execution Governors and Limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- **WITH USER_MODE**: [User Mode SOQL](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_enforce_usermode.htm)
