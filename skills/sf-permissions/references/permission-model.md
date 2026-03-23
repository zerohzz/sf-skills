<!-- Parent: sf-permissions/SKILL.md -->
# Salesforce Permission Model

A guide to understanding how permissions work in Salesforce.

## Overview

Salesforce uses a layered permission model:

```
┌─────────────────────────────────────────────────────┐
│                      USER                           │
├─────────────────────────────────────────────────────┤
│                    PROFILE                          │
│  (Base permissions - one per user)                  │
├─────────────────────────────────────────────────────┤
│           PERMISSION SET GROUPS                     │
│  (Collections of Permission Sets)                   │
├─────────────────────────────────────────────────────┤
│              PERMISSION SETS                        │
│  (Additive permissions)                             │
└─────────────────────────────────────────────────────┘
```

## Key Concepts

### Profiles

- **One profile per user** (mandatory)
- Defines base-level access
- Can restrict or grant permissions
- Legacy approach - Salesforce recommends minimal profiles + Permission Sets

### Permission Sets (PS)

- **Additive only** - can grant access, cannot revoke
- Multiple PS can be assigned to a user
- Can include:
  - Object CRUD permissions
  - Field-Level Security (FLS)
  - Apex Class access
  - Visualforce Page access
  - Flow access
  - Custom Permissions
  - Tab visibility
  - System permissions

### Permission Set Groups (PSG)

- **Container for multiple Permission Sets**
- Assign one PSG instead of many individual PS
- Simplifies user provisioning
- Status can be "Active" or "Outdated"

## Permission Types

### Object Permissions

| Permission | Description |
|------------|-------------|
| Create | Insert new records |
| Read | View records |
| Edit | Update existing records |
| Delete | Remove records |
| View All | Read all records regardless of sharing |
| Modify All | Full access regardless of sharing |

### Field-Level Security (FLS)

| Permission | Description |
|------------|-------------|
| Read | View field value |
| Edit | Modify field value |

Note: Edit includes Read access.

### Setup Entity Access

Access to programmatic components:

| Entity Type | Examples |
|-------------|----------|
| ApexClass | Controller classes, utility classes |
| ApexPage | Visualforce pages |
| Flow | Screen flows, autolaunched flows |
| CustomPermission | Feature flags, custom access controls |

### System Permissions

Organization-wide permissions like:

- ViewSetup
- ModifyAllData
- ViewAllData
- ManageUsers
- ApiEnabled
- RunReports
- ExportReport

## Common Permission Patterns

### Sales User Pattern

```
Permission Set Group: Sales_Cloud_User
├── Account_Access (PS)
│   └── Account: CRUD
├── Opportunity_Access (PS)
│   └── Opportunity: CRUD
└── Report_Runner (PS)
    └── System: RunReports, ExportReport
```

### API Integration Pattern

```
Permission Set: Integration_User
├── System: ApiEnabled
├── Objects: Read on required objects
└── Custom Permission: API_Access_Enabled
```

### Admin Lite Pattern

```
Permission Set: Admin_Lite
├── System: ViewSetup (NOT ModifyAllData)
├── System: ManageUsers
└── Custom Permission: Can_Manage_Users
```

## Best Practices

### 1. Minimum Necessary Access

Grant only the permissions users actually need.

### 2. Use Permission Set Groups

Group related PS into PSGs for easier management:
- `Sales_Cloud_User` (PSG) instead of 5 individual PS
- `Service_Cloud_User` (PSG) for case management

### 3. Audit Regularly

Use sf-permissions to:
- Find PS with overly broad access (ModifyAllData)
- Identify unused PS
- Document permission structures

### 4. Naming Conventions

```
Permission Set:     [Department]_[Capability]_PS
Permission Set Group: [Department]_[Role]_PSG

Examples:
  - Sales_Account_Edit_PS
  - Sales_Manager_PSG
  - HR_Employee_Data_Access_PS
```

### 5. Document Custom Permissions

Custom Permissions should have clear names:
- `Can_Approve_Expenses`
- `View_Salary_Data`
- `Export_Customer_Data`

## Permission Set Licenses (PSL)

Permission Set Licenses control access to features that require additional licensing beyond the base user license.

- **Managed packages require PSL assignment before Permission Set assignment** — assigning the PS without the PSL will silently fail or throw an error
- Example: Salesforce CPQ requires the "Salesforce CPQ License" PSL assigned to the user before the CPQ Permission Set will function
- Other common PSLs: "Salesforce CMS Integration", "Identity Connect", "Pardot"
- Check assigned PSLs: Setup > Users > select user > Permission Set License Assignments
- PSLs are a limited, org-wide resource — monitor available vs. consumed counts

---

## Muting Permission Sets

Muting Permission Sets are used **within Permission Set Groups** to selectively REMOVE specific permissions that would otherwise be granted by the included Permission Sets.

**Pattern**: "Give everything in the group, except X"

```
Permission Set Group: Sales_Manager_PSG
├── Account_Full_Access_PS        (grants Account: Create, Read, Edit, Delete)
├── Opportunity_Full_Access_PS    (grants Opportunity: Create, Read, Edit, Delete)
└── 🔇 Muting_No_Delete_PS       (mutes Account: Delete, Opportunity: Delete)

Result: Sales Managers can Create, Read, Edit — but NOT Delete — Accounts and Opportunities
```

- Muting PS can only be added to Permission Set Groups, not assigned directly to users
- They can mute object permissions, field permissions, and system permissions
- They CANNOT mute Custom Permissions (this is a platform limitation)

---

## Delegated Administration

Delegated Administration allows you to assign admin-like privileges for specific objects or user groups without granting full System Administrator access.

- Configured in: Setup > Security > Delegated Administration
- A delegated admin can:
  - Create and edit users in specified roles and subordinate roles
  - Assign specified profiles
  - Assign specified permission sets
  - Manage custom objects
  - Administer specified user groups
- Delegated admins **cannot** modify org-wide settings, security controls, or any metadata

---

## Principle of Minimum Blast Radius

Design your permission architecture to minimize the impact of any single permission change:

1. **Permission Sets should be as narrow as possible** — one PS per feature or function, not per role
2. **Use Permission Set Groups to compose role-level access** — if a role needs Account access + Report access + Flow access, create three separate PS and group them into one PSG
3. **Avoid "god" Permission Sets** that grant broad access — splitting access means you can revoke one capability without affecting others
4. **Never put `ModifyAllData` or `ViewAllData` in a Permission Set** unless absolutely required — these bypass all sharing rules and FLS checks
5. **Audit regularly** — use SOQL on `PermissionSet`, `PermissionSetAssignment`, and `SetupEntityAccess` to find overly permissive configurations

---

## Related Salesforce Documentation

- [Permission Sets](https://help.salesforce.com/s/articleView?id=sf.perm_sets_overview.htm)
- [Permission Set Groups](https://help.salesforce.com/s/articleView?id=sf.perm_set_groups.htm)
- [Field-Level Security](https://help.salesforce.com/s/articleView?id=sf.users_fields_fls.htm)
- [Trailhead: Data Security](https://trailhead.salesforce.com/content/learn/modules/data_security)
- [Sharing Architecture: Architect Guide](https://architect.salesforce.com/decision-guides/data-access)
