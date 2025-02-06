# ClickHouse User Access Restriction Troubleshooting and Resolution

## Issue Summary

The objective was to create a restricted user (`user_analytics`) in ClickHouse who should only see tables in `system.tables` where the database column contains the value 'analytics'. The default user should still have full visibility of all tables.

A challenge arose when, after implementing a row policy for `user_analytics`, no rows were returned when querying `system.tables`. Investigation revealed this was caused by missing explicit permissions and row policies applying globally to each user in older versions of ClickHouse.

## Troubleshooting Process

### 1. Understanding the Issue
- The user `user_analytics` was created with a row-level access policy (`user_analytics_filter`) restricting access to only rows where database = 'analytics'
- Despite applying the policy, `user_analytics` could not see any rows
- The default user (`default`) could see all tables without restrictions in some cases but was affected by row policies in older ClickHouse versions

### 2. Research and Findings
- **Row-Level Security on system.tables**: ClickHouse applies row policies globally per user in older versions. If a row policy is applied to `user_analytics`, it removes access for all users unless explicitly overridden
- **Permissions Required**: Even if a row policy is applied, the user must have explicit SELECT and SHOW TABLES privileges on `system.tables` to view relevant rows
- **GitHub Issue #12373** states:
  > "Row policies affect each user. Currently, there is no way to avoid that. To allow the default user to access your table without any filters, you have to define one more row policy."

### 3. Key Observations
- In ClickHouse 22.1.3.7-alpine, a default user policy is required (`default_user_filter`) to restore full access for other users
- In ClickHouse 24.3, the issue is fixed, and no additional policy is required

## Resolution

### Final Working Solution

#### 1. Cleaned Up Existing Configuration
```sql
DROP ROW POLICY IF EXISTS user_analytics_filter ON system.tables;
DROP ROW POLICY IF EXISTS default_user_filter ON system.tables;
DROP USER IF EXISTS user_analytics;
DROP ROW POLICY IF EXISTS role_analytics;
```

#### 2. Created the role_analytics Role
```sql
CREATE ROLE role_analytics SETTINGS 
    log_profile_events = 1 READONLY, 
    log_queries = 1 READONLY, 
    log_query_settings = 1 READONLY, 
    log_query_threads = 1 READONLY;
```

#### 3. Created the user_analytics User
```sql
CREATE USER user_analytics 
IDENTIFIED WITH plaintext_password BY 'password' 
DEFAULT ROLE role_analytics;
```

#### 4. Granted Necessary Permissions
```sql
GRANT SELECT ON system.* TO role_analytics;
GRANT SELECT ON analytics.* TO role_analytics;
GRANT SHOW TABLES ON system.* TO role_analytics;
GRANT SHOW TABLES ON analytics.* TO role_analytics;
```

✅ Tested and confirmed to work on ClickHouse version 22.1.3.7-alpine and 24.3.

#### 5. Created Row Policy for user_analytics
```sql
CREATE ROW POLICY user_analytics_filter ON system.tables
FOR SELECT 
USING database = 'analytics' 
TO user_analytics;
```
Ensures `user_analytics` only sees tables where database = 'analytics'.

#### 6. Additional Step Required for ClickHouse 22.1
In ClickHouse version 22.1, a default user policy must be created to restore access to system.tables for other users:

```sql
CREATE ROW POLICY default_user_filter ON system.tables 
FOR SELECT 
USING 1 = 1 
TO ALL EXCEPT user_analytics;
```

✅ Tested and confirmed that this step is required in ClickHouse 22.1.

#### 7. No Additional Policy Needed for ClickHouse 24.3
In ClickHouse version 24.3, this issue has been fixed, and the additional policy for the default user (`default_user_filter`) is no longer required.

## Version-Specific Observations

| ClickHouse Version | Issue Present? | Additional Policy Required? |
|-------------------|----------------|---------------------------|
| 22.1.3.7-alpine   | ✅ Yes         | ✅ CREATE ROW POLICY default_user_filter ... is required |
| 24.3              | ❌ No          | ❌ No additional policy is required |

## Conclusion

### Findings
- Older versions (22.1.3.7-alpine) require an additional row policy (`default_user_filter`) to restore system table visibility for other users
- Newer versions (24.3) have fixed this behavior, and no additional policy is needed
- The issue aligns with GitHub Issue #12373, which states that row policies apply to each user globally

### Additional Questions
- Should ClickHouse introduce role-based row policies instead of applying them per user?
- Could an explicit warning be provided when a row policy is created on system.tables in older versions?

## Customer Response Template

Subject: Resolution for Restricted ClickHouse User Access to system.tables

Hello [Customer Name],

We have identified and resolved the issue regarding the visibility of tables in system.tables for the user `user_analytics`. The root cause was missing SELECT and SHOW TABLES permissions and the way row policies apply globally to all users.

Key Updates:
- Granted SELECT and SHOW TABLES permissions to role_analytics
- Created a row policy ensuring user_analytics can only see tables in analytics
- For ClickHouse 22.1, an additional policy (`default_user_filter`) is required to restore full table visibility for other users
- For ClickHouse 24.3, this issue has been fixed, and no additional policy is required

Confirmed on ClickHouse Versions:
- ✅ 22.1.3.7-alpine (Requires default_user_filter)
- ✅ 24.3 (No additional policy required)

Reference: GitHub Issue #12373 confirms that row policies apply per user, so an explicit policy for the default user is required in older versions.

Next Steps (for 22.1 users only):
If you are using ClickHouse 22.1, apply the following SQL command:

```sql
CREATE ROW POLICY default_user_filter ON system.tables 
FOR SELECT 
USING 1 = 1 
TO ALL EXCEPT user_analytics;

GRANT SELECT ON system.* TO role_analytics;
GRANT SELECT ON analytics.* TO role_analytics;
GRANT SHOW TABLES ON system.* TO role_analytics;
GRANT SHOW TABLES ON analytics.* TO role_analytics;
```

If you are on ClickHouse 24.3 or later, no additional steps are required.

Let us know if you need further assistance.

Best regards,
[Your Name]
[Your Position]
