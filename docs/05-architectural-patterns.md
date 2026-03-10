# Advanced Architectural Patterns

This section is targeted toward Tech Leads and Architects who are designing complex systems and orchestrating database deployments at scale.

## 1. Zero-Downtime Deployments (Blue-Green)
When aiming for continuous availability, you cannot lock the database and bring the application down while migrations run. 

**The Rule of Backward Compatibility:**
Every database migration must be backward compatible with the **previous** version of the application code.

**The Workflow (Replacing a column `first_name` and `last_name` with `full_name`):**
1. **Release 1 (Database Only):** 
   - Add the new `full_name` column.
   - Deploy code that writes to *both* old and new columns, but reads from the old columns.
2. **Backfill:**
   - Run a script (or Liquibase ChangeSet) to migrate existing data from the old columns to the new column.
3. **Release 2 (Application Shift):**
   - Deploy code that reads and writes exclusively from the new `full_name` column.
4. **Release 3 (Cleanup):**
   - Deploy a Liquibase ChangeSet that drops the old `first_name` and `last_name` columns.

This 3-phase rollout guarantees zero downtime. Liquibase handles the structural changes in Release 1 and 3.

## 2. De-coupling Liquibase from Spring Boot Startup
Relying on Spring Boot auto-configuration (`spring.liquibase.enabled=true`) works great for small apps. But at scale, it introduces problems:

**The Problems:**
1. **Startup Latency:** If you have 50 microservice instances scaling up simultaneously, they will all try to acquire the `DATABASECHANGELOGLOCK`. Instance 1 gets it, the other 49 wait. This dramatically increases application startup time.
2. **Permissions:** The Spring Boot application needs `DDL` (Data Definition Language) permissions to alter tables. Security policies often dictate that the runtime application should only have `DML` (Select, Insert, Update, Delete) permissions.

**The Architect's Solution:**
Run Liquibase as a separate step in your CI/CD pipeline.

1. Pipeline Phase 1: Build Java App.
2. Pipeline Phase 2: Run Liquibase migrations (CLI or Maven Plugin) against the production database using highly privileged DBA credentials.
3. Pipeline Phase 3: Start the Spring Boot application using lower privileged Application credentials with `spring.liquibase.enabled=false`.

## 3. Multi-Tenant Architectures
If your application provides a separate schema or database per customer (Tenant), you need a strategy to upgrade hundreds of schemas reliably.

**Approach A: Spring Boot Multi-Tenant Execution**
Custom components loop through all tenant datasources and execute `Liquibase(datasource, changelog)` programmatically at startup.

**Approach B: CLI Iteration in CI/CD (Preferred)**
Write a pipeline script that queries the central registry of tenants, loops over their connection strings, and executes the Liquibase CLI against each one. 

## 4. Managing Large Data Migrations
Liquibase is a schema migration tool, not an ETL tool. 

If you need to update 50 million rows in a table (e.g., standardizing a status column string formatting), doing this inside a Liquibase ChangeSet via `<sql>` or `<update>` can cause immense locks, blowing up your transaction log.

**The Solution:**
1. Use Liquibase to add the new column or table structural requirements.
2. Run a background batch job (e.g., Spring Batch) to migrate the data in chunks of 1000 without locking the main tables for hours.
3. When the batch job finishes, deploy a new Liquibase ChangeSet to apply tight constraints (`NOT NULL`) to the newly populated data.

## 5. Lock Table Management
In clustered environments, a node may crash or be killed while holding the `DATABASECHANGELOGLOCK`. 

If this happens, subsequent Liquibase executions will hang indefinitely.

**Handling dead locks:**
- Use the Liquibase CLI `releaseLocks` command.
- Set a sensible timeout property or write a custom Spring context listener to forcibly break locks older than a certain threshold, though this is risky and CLI intervention is preferred to investigate *why* it crashed.
