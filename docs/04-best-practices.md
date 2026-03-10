# Liquibase Best Practices

To maintain a healthy database lifecycle block, you need to follow these best practices. Treating Liquibase as an arbitrary dump of SQL scripts will lead to unmaintainable nightmares.

## 1. Never Modify an Executed ChangeSet!
This is the golden rule of Liquibase. 

If a ChangeSet has been executed against a database (like Testing or Production), **DO NOT EDIT IT.** 

Liquibase calculates an MD5 Checksum for every ChangeSet and stores it in the `DATABASECHANGELOG` table. If you modify the YAML/XML after it has run, the checksum changes. On the next deployment, Liquibase will crash with a `ValidationFailedException: MD5 Checksum error`.

**How to Fix a Mistake:**
If you made a typo (e.g., named a column `emial` instead of `email`), you must:
1. Leave the bad ChangeSet alone.
2. Create a **NEW** ChangeSet that renames the column from `emial` to `email`.

**Exception (runOnChange):**
If you have views or stored procedures defined in raw SQL files, you can use `runOnChange: true`.
```yaml
  - changeSet:
      id: "create-user-view"
      author: "dev"
      runOnChange: true
      changes:
        - createView: ...
```
This tells Liquibase: "If the checksum changes, don't crash. Just run it again."

## 2. One Change = One ChangeSet
Keep your ChangeSets atomic. Don't create 5 tables and insert 100 rows in a single ChangeSet.

If a massive ChangeSet fails halfway through (e.g., table 1 created, table 2 created, but table 3 failed due to syntax error), your database is in a broken, partial state. 

Smaller ChangeSets ensure better failure isolation and easier rollbacks.

## 3. Naming Conventions (id, author, filenames)
**File Names:**
Use sequential numbering or timestamps to guarantee execution order.
- `2023-10-27-01-create-users.yml`
- `2023-10-27-02-add-roles.yml`

**IDs:**
IDs must be unique within a file. Don't use random strings or numbers. Use descriptive names.
- Good: `create-table-orders-v1`
- Bad: `1`

**Authors:**
Don't use `author: dev`. Use the developer's actual name, Git username, or Jira ticket number.
- `author: jsmith`
- `author: TICKET-1234`

## 4. Always Test Rollbacks
If you are using raw SQL or custom execution blocks, you must write a manual `<rollback>` section. Always test the rollback locally before merging your Pull Request. Run `mvn liquibase:rollback` to ensure your database cleanly reverts.

## 5. Idempotent ChangeSets
While Liquibase prevents running a ChangeSet twice via the tracking table, it's best to write idempotent migrations (i.e., operations that can be run multiple times safely).

Use preconditions extensively:
```yaml
  - preConditions:
      - onFail: MARK_RAN
      - not:
          tableExists:
            tableName: "employee"
```
This ensures that if the DBA manually created the table yesterday, Liquibase won't crash trying to create it again today.

## 6. Structure Large Projects cleanly
Don't put everything in one `db.changelog-master.yaml`. Use `includeAll` or `include` logically.

```yaml
databaseChangeLog:
  - include:
      file: db/changelog/releases/1.0.0/changelog-1.0.0.yaml
  - include:
      file: db/changelog/releases/1.1.0/changelog-1.1.0.yaml
```

## 7. Version Control is King
Liquibase files belong in the same Git repository as the application code. A specific git commit hash should represent both the exact state of the Java code and the exact state of the database schema required for that code to run.
