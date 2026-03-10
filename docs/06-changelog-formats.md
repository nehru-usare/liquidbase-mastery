# Liquibase ChangeLog Formats

Liquibase is not restricted to YAML. It supports several formats for defining your database changes (`ChangeLogs` and `ChangeSets`). Each format has its own strengths and use cases. Understanding how they work behind the scenes is crucial for an Architect or Senior Developer.

The core concept remains the same across all formats: you define a **ChangeSet** (identified by an `id` and an `author`), which contains **Changes** (like `createTable` or `addColumn`). Liquibase reads these definitions and translates them into the specific SQL dialect of your target database (PostgreSQL, MySQL, Oracle, etc.).

Here are the primary formats supported by Liquibase:

## 1. YAML Format

YAML is often the preferred choice in modern Spring Boot applications because it's human-readable, concise, and aligns well with Spring's own `application.yml` configuration files.

**How it works:**
Liquibase parses the YAML structure into its internal Java object model representing the ChangeSets and Changes.

**Example `changelog.yaml`:**
```yaml
databaseChangeLog:
  - changeSet:
      id: "create-users-table"
      author: "amit-developer"
      changes:
        - createTable:
            tableName: "users"
            columns:
              - column:
                  name: "id"
                  type: "bigint"
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: "username"
                  type: "varchar(50)"
                  constraints:
                    nullable: false
                    unique: true
```

**Pros:**
*   Highly readable and less verbose than XML.
*   Easy to write and maintain for developers familiar with modern configuration files.

**Cons:**
*   Strict indentation rules can sometimes cause hard-to-spot parsing errors.

---

## 2. XML Format

XML is the original and most mature format for Liquibase. It offers the most comprehensive validation and tooling support.

**How it works:**
Liquibase uses XML parsers to read the elements and attributes. Because it has an XSD (XML Schema Definition), your IDE can provide excellent autocompletion and validation as you type.

**Example `changelog.xml`:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <changeSet id="create-users-table" author="amit-developer">
        <createTable tableName="users">
            <column name="id" type="bigint" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="username" type="varchar(50)">
                <constraints nullable="false" unique="true"/>
            </column>
        </createTable>
    </changeSet>

</databaseChangeLog>
```

**Pros:**
*   **Strong Validation:** The XSD schema ensures you don't make structural mistakes before even running Liquibase.
*   **IDE Support:** Excellent autocompletion and error highlighting in tools like IntelliJ IDEA or Eclipse.
*   Most comprehensive documentation naturally stems from its XML roots.

**Cons:**
*   Verbose. It requires more typing and can be harder to scan visually compared to YAML or SQL.

---

## 3. Formatted SQL Format

Liquibase allows you to write raw SQL scripts while still getting the benefits of ChangeSet tracking (the `DATABASECHANGELOG` table). This is achieved using specific SQL comments to define the boundaries and metadata of a ChangeSet.

**How it works:**
Liquibase parses the SQL file, looking for specific comment patterns (like `--liquibase formatted sql` and `--changeset author:id`). It treats the SQL commands between these comments as the body of the ChangeSet. This format bypasses Liquibase's SQL generation engine and executes your SQL directly.

**Example `changelog.sql`:**
```sql
--liquibase formatted sql

--changeset amit-developer:create-users-table
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE
);
--rollback DROP TABLE users;

--changeset amit-developer:insert-default-user
INSERT INTO users (username) VALUES ('admin');
--rollback DELETE FROM users WHERE username = 'admin';
```

**Pros:**
*   **DBA Friendly:** Database administrators often prefer pure SQL over learning XML or YAML syntax.
*   **Database-Specific Features:** Gives you direct access to highly specific features of your database (e.g., specific index types or triggers) that Liquibase might not fully support in its abstract formats.

**Cons:**
*   **Not Agnostic:** The SQL you write is tied to your specific database engine (e.g., PostgreSQL syntax won't work on Oracle). You lose the ability to easily switch database vendors.
*   **Manual Rollbacks Required:** Because Liquibase doesn't know *what* the SQL does (it just runs it), it cannot automatically generate a rollback. You **must** provide the `--rollback` statement for operations that can be reversed.

---

## 4. JSON Format

Similar to YAML and XML, JSON is supported for defining abstract changes.

**How it works:**
Liquibase parses the JSON structure into its internal Java object model.

**Example `changelog.json`:**
```json
{
  "databaseChangeLog": [
    {
      "changeSet": {
        "id": "create-users-table",
        "author": "amit-developer",
        "changes": [
          {
            "createTable": {
              "tableName": "users",
              "columns": [
                {
                  "column": {
                    "name": "id",
                    "type": "bigint",
                    "autoIncrement": true,
                    "constraints": {
                      "primaryKey": true
                    }
                  }
                },
                {
                  "column": {
                    "name": "username",
                    "type": "varchar(50)",
                    "constraints": {
                      "nullable": false,
                      "unique": true
                    }
                  }
                }
              ]
            }
          }
        ]
      }
    }
  ]
}
```

**Pros:**
*   Useful if your tooling or deployment pipelines heavily rely on JSON generation or parsing.

**Cons:**
*   Verbose and strict syntax (requires quotes around everything, no trailing commas). It is generally harder for humans to write and read manually compared to YAML.

---

## Combining Formats (The Master File approach)

You don't have to choose just one format for your entire project! A common practice is to use a master file (often XML or YAML) that `includes` other files.

You might use YAML for your standard table creations but `include` SQL files for complex stored procedures or database-specific optimizations.

**Example Master File (`master-changelog.yaml`):**
```yaml
databaseChangeLog:
  - include:
      file: "db/changelog/01-create-tables.xml"  # Using XML for structure
  - include:
      file: "db/changelog/02-add-columns.yaml"   # Using YAML for simplicity
  - include:
      file: "db/changelog/03-complex-views.sql"  # Using SQL for database-specific logic
```

This flexibility allows your team to use the best tool for the specific task while keeping the overall execution ordered and tracked by Liquibase.
