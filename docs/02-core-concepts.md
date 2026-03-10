# Liquibase Core Concepts

Once you understand *why* you need Liquibase, you need to know how to use its core features effectively to structure database migrations.

## 1. Change Types (The "What")
Liquibase provides an abstraction over raw SQL. This means you can write a ChangeSet in YAML or XML, and Liquibase will translate it into the specific SQL dialect for PostgreSQL, MySQL, Oracle, etc.

**Common Change Types:**
- `createTable`, `dropTable`, `renameTable`
- `addColumn`, `dropColumn`, `renameColumn`, `modifyDataType`
- `addPrimaryKey`, `addForeignKeyConstraint`, `addUniqueConstraint`
- `createIndex`, `dropIndex`
- `insert`, `update`, `delete`, `loadData` (for loading CSVs)

**Using Raw SQL:**
If Liquibase doesn't support a specific database feature (like advanced functions or triggers), you can always fall back to raw SQL using the `sql` or `sqlFile` change types.

## 2. Preconditions (The "If")
Sometimes, you only want a ChangeSet to run if a certain condition is met. Preconditions allow you to dictate the state of the database before a ChangeSet is executed.

Examples:
- Only create a table if it doesn't already exist.
- Only drop an index if the index exists.
- Only run if the database is PostgreSQL.

```yaml
  - changeSet:
      id: "add-email-column"
      author: "dev"
      preConditions:
        - onFail: MARK_RAN
        - not:
            columnExists:
              tableName: "users"
              columnName: "email"
      changes:
        - addColumn: ...
```
**`onFail` actions**: `HALT` (stop migration), `CONTINUE` (skip this precondition but run the changeset), `MARK_RAN` (skip the changeset but mark it as executed so it never tries again).

## 3. Contexts (The "Where")
Contexts allow you to tag ChangeSets so they only run in specific environments (e.g., `dev`, `test`, `prod`).

Suppose you want to insert dummy test data only in your local development environment:

```yaml
  - changeSet:
      id: "insert-dev-users"
      author: "qa"
      context: "dev, test"
      changes:
        - insert:
            tableName: "users"
            columns:
              - column: { name: "username", value: "testuser1" }
```
When running Liquibase, you pass the active contexts. If you run it with `--contexts="prod"`, this changeset will be completely ignored.

## 4. Labels
Similar to Contexts but used for tagging features or releases.
For example, you might tag all ChangeSets related to a new billing feature with `label: billing_v2`. This allows architects to control executions based on logical feature groupings rather than just environments.

## 5. Rollbacks (The "Undo")
What happens if a deployment is successful, but the feature is broken, and you need to revert the application version? You might also need to revert the database schema.

Liquibase supports **Rollbacks**.

**Automatic Rollbacks:**
Many Liquibase change types have automatic rollbacks. If your ChangeSet is `createTable`, Liquibase knows the rollback is `dropTable`. It does it automatically.

**Manual Rollbacks:**
If your ChangeSet uses raw SQL, or a destructive operation like `dropTable` (it can't magically recreate the data), you must provide a manual rollback instruction.

```yaml
  - changeSet:
      id: "raw-sql-update"
      author: "dev"
      changes:
        - sql: "UPDATE users SET status = 'ACTIVE' WHERE status IS NULL"
      rollback:
        - sql: "UPDATE users SET status = NULL WHERE status = 'ACTIVE'"
```

## 6. Include & IncludeAll (Structuring the Project)
You don't put everything in one file. A standard approach is to have a `master-changelog.yaml` which simply imports other files.

```yaml
databaseChangeLog:
  - include:
      file: "db/changelog/versions/v1.0.0/01-create-users.yml"
  - include:
      file: "db/changelog/versions/v1.0.0/02-create-orders.yml"
  - include:
      file: "db/changelog/versions/v1.1.0/01-add-email.yml"
```
This keeps migrations organized by application versions or feature sets.
