<!-- Parent: sf-data/SKILL.md -->
# Bulk Operations Guide

When and how to use Salesforce Bulk API operations.

## Decision Matrix

| Record Count | Recommended API | Command |
|--------------|-----------------|---------|
| 1-10 | Single Record | `sf data create record` |
| 11-2000 | Standard API | `sf data query` + Apex |
| 2000-10M | Bulk API 2.0 | `sf data import bulk` |
| 10M+ | Data Loader | External tool |

## Bulk API 2.0 Commands

### Import (Insert)
```bash
sf data import bulk \
  --file accounts.csv \
  --sobject Account \
  --target-org myorg \
  --wait 30
```

### Update
```bash
sf data update bulk \
  --file updates.csv \
  --sobject Account \
  --target-org myorg \
  --wait 30
```

### Upsert (Insert or Update)
```bash
sf data upsert bulk \
  --file upsert.csv \
  --sobject Account \
  --external-id External_Id__c \
  --target-org myorg \
  --wait 30
```

### Delete
```bash
sf data delete bulk \
  --file delete.csv \
  --sobject Account \
  --target-org myorg \
  --wait 30
```

### Export
```bash
sf data export bulk \
  --query "SELECT Id, Name FROM Account" \
  --output-file accounts.csv \
  --target-org myorg \
  --wait 30
```

## CSV Format Requirements

- First row: Field API names
- UTF-8 encoding
- Comma delimiter (default)
- Max 100MB per file

## Bulk API Limits

| Limit | Value |
|-------|-------|
| Batches per 24 hours | 10,000 |
| Records per 24 hours | 10,000,000 |
| Max file size | 100 MB |
| Max concurrent jobs | 100 |

## Error Handling

```bash
# Check job status
sf data resume --job-id [job-id] --target-org myorg

# Get results
sf data bulk results --job-id [job-id] --target-org myorg
```

## Parallel Processing

Bulk API 2.0 automatically parallelizes record processing — no developer configuration needed:

- The platform splits your job into **up to 10 batches**, processed in parallel
- Batch size is determined automatically based on record complexity and org limits
- Each batch processes up to 10,000 records
- Serial mode is available if you need ordered processing (e.g., to avoid lock contention on parent records):

```bash
# Force serial processing when parallel causes UNABLE_TO_LOCK_ROW errors
sf data import bulk \
  --file accounts.csv \
  --sobject Account \
  --target-org myorg \
  --wait 30 \
  --async
```

**When to consider serial mode**: Parent-child inserts in the same job, records that trigger sharing recalculation on the same parent, or operations on objects with heavy automation (flows, triggers) that contend for the same resources.

---

## Partial Failure Handling

Bulk API 2.0 jobs can partially succeed — some records process while others fail. Always check results:

```bash
# Submit an upsert job
sf data upsert bulk --sobject Account --file accounts.csv --external-id External_Id__c --target-org myorg --wait 30

# Check job results (shows success/failure counts)
sf data bulk results --job-id <jobId> --target-org myorg

# Download failed records for retry
sf data bulk results --job-id <jobId> --target-org myorg --result-format csv > failed-records.csv
```

**Common failure patterns and fixes:**

| Error | Cause | Fix |
|-------|-------|-----|
| `DUPLICATES_DETECTED` | Duplicate rules blocked insert | Review duplicate rules or add `--allow-duplicates` |
| `UNABLE_TO_LOCK_ROW` | Parallel batches updating related records | Use serial mode or reorder CSV |
| `REQUIRED_FIELD_MISSING` | CSV missing required fields | Add missing columns to CSV |
| `INVALID_CROSS_REFERENCE_KEY` | Lookup target doesn't exist | Ensure parent records exist first |
| `STRING_TOO_LONG` | Field value exceeds max length | Truncate data before import |

**Retry pattern**: Filter the results CSV for failed rows, fix the data issues, and re-submit only the failed records as a new bulk job.

---

## Best Practices

1. **Chunk large files** - Split files >100MB
2. **Use --wait** - Monitor completion
3. **Handle partial failures** - Check result files
4. **Test in sandbox** - Validate before production

---
## Official References
- **Bulk API 2.0 Guide**: [Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.api_asynch.meta/api_asynch/)
- **Trailhead**: [Large Data Volumes](https://trailhead.salesforce.com/content/learn/modules/large-data-volumes)
- **Data Loader Guide**: [Salesforce Help](https://help.salesforce.com/s/articleView?id=sf.data_loader.htm)
