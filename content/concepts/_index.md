---
title: "Concepts"
draft: false
chapter: false
pre: "<b>4. </b>"
weight: 4
icon: ""
---

### Overview

In Evolve, each sql script is called a **migration**. A migration is composed of a unique version, an informative description and a checksum.

At startup Evolve will collect all migrations located in [Evolve.Locations](/configuration/#options), searching recursively for files with a specific file name structure. All files are then sorted by version in ascending order, regardless their initial directory. **Each version of a migration must be unique**. If not, the validation phase fails.

Once executed, the name and checksum of the migration are saved to a metadata table in your database. This table is checked every time Evolve is run, to see if the migration script has already been applied.

### Migration

A migration is composed of a *version*, a *description* and a *checksum*.

<i class="fa fa-hand-o-right"></i> The version must be unique. Migrations are applied in the order of their versions. *The first numbers that make up the version of your migration may be the release version of your application followed by a counter.*

To be processed by Evolve your migration scripts must follow this file name structure: *V1_3_1_1__Create_table.sql*:

- **prefix**: configurable, default: **V**
- **version**: numbers separated by **_** (one underscore)
- **separator**: configurable, default: **__** (two underscores)
- **description**: words separated by single underscores
- **suffix**: configurable, default: **.sql** 

<i class="fa fa-hand-o-right"></i> The description is informative and allow you to remember what the migration was about.

<i class="fa fa-hand-o-right"></i> The checksum is stored in the Evolve metadata table and used to detect accidental changes.

### Commands

Evolve has 3 execution commands to interact with your database:

<i class="fa fa-hand-o-right"></i> **migrate**: applies the migrations. It's the main command. 

<i class="fa fa-hand-o-right"></i> **erase**: erases the database schema(s) if Evolve has created it or has found it empty (cf. [Metadata table](#metadata-table)). Otherwise Evolve will not do anything. This command is intended to be use in development to start with a new clean database.

<i class="fa fa-hand-o-right"></i> **repair**: updates checksums of previously applied migrations with those of the currently available migration scripts.

### Transactions

Each migration is executed in a separate database transaction. Thus each script will either succeed or fail completely and Evolve will stop on the first error. If your database supports DDL statements within a transaction, failed migrations will always be rolled back, otherwise you will have to manually fix your database state.

### Placeholders

Placeholders are strings prefixed by: *Evolve.Placeholder.* that will be replaced in sql migrations before their execution. They make it possible to get dynamic migrations depending on the environment.

```xml
<add key="Evolve.Placeholder.schema1" value="my_schema" />
<add key="Evolve.Placeholder.database" value="my_db" />
```

```sql
SELECT * FROM ${database}.${schema1}.TABLE_1; -- SELECT * FROM my_db.my_schema.TABLE_1;
```

### Metadata table

During its initial execution, Evolve creates a table with a default name of **changelog**, to keep track of all migrations (applied or failed) and to store their checksums. Example:

| id | type | version | description | name | checksum | installed_by | installed_on | success |
|----|:----:|---------|-------------|------|----------|-------------|--------------|:-------:|
| 1 | 2 | 0 | Empty schema found: dbo. | dbo | | sa | 22/02/2018 20:45:15 | True |
| 2 | 0 | 1.0.0.0 | create table calendrier and constraints | V1_0_0_0__create_table_calendrier_and_constraints.sql | D4AAF08FBF70D3B327A9A3D3B4E0E21A | sa | 22/02/2018 20:45:15 | True |
| 3 | 0 | 1.0.0.1 | create triggers | V1_0_0_1__create_triggers.sql | A4AA367C92B99C56E881324726882B9B | sa | 22/02/2018 20:45:16 | True |

<i class="fa fa-hand-o-right"></i> The `type` column is used to identify which type of metadata is stored:

- 0: migration
- 1: new schema. Indicates that Evolve created the schema and thus can **drop** it if `Evolve.Command = "erase"`.
- 2: empty schema. Indicates that the schema was empty before Evolve applied the first migration and thus can **erase** it if `Evolve.Command = "erase"`. 
- 3: start version. Used by Evolve to store the version from which to take migrations (cf. `Evolve.StartVersion`).
