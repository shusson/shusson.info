# Deleting a subtree in a closure table the better way

__24/06/2019__

<!-- ![aquaduct](https://en.wikipedia.org/wiki/Pont_du_Gard#/media/File:Pont_du_Gard_BLS.jpg) -->

The [previous implementation](https://shusson.info/post/deletingasubtreeinaclosuretable) required custom logic to delete an element from the closure tree.
But there's a simpler way to implement delete using postgres `OnDelete` [constraints](https://www.postgresql.org/docs/9.5/ddl-constraints.html).

We want to add the following constraints:

- DELETE the query entity when a "query parentId" becomes null
- DELETE a "query_closure" table row when either the "id_ancestor" or "id_descendant" become null

A constraint in postgresql looks like:

```SQL
ALTER TABLE query
ADD CONSTRAINT "fk_parent_id"
FOREIGN KEY ("parent_id")
REFERENCES query(id)
ON DELETE CASCADE;
```

Unfortunately there is no way to currently add constraints like this to a tree entity in TypeORM. But we can run it as a migration before the app starts.

TypeORM stores the metadata about the table in the repository, we can use the metadata to define the variables in the SQL to make them less brittle.

```typescript
static async addOnDeleteCascades() {
    const queryRepository = getRepository(Query);
    const tableName = queryRepository.metadata.tableName;
    const treeRelationFks = queryRepository.metadata.treeParentRelation.foreignKeys[0];

    const parentIdColumnName = treeRelationFks.columnNames[0];
    const parentIdFkName = treeRelationFks.name;

    const closureTableMetadata = queryRepository.metadata.closureJunctionTable;

    const closureTableName = closureTableMetadata.tableName;
    const closureAncestorColumnName = closureTableMetadata.foreignKeys[0].columnNames[0];
    const closureAncestorFkName = closureTableMetadata.foreignKeys[0].name;
    const closureDescendantColumnName = closureTableMetadata.foreignKeys[1].columnNames[0];
    const closureDescendantFkName = closureTableMetadata.foreignKeys[1].name;

    await getConnection().query(`
            ALTER TABLE ${tableName}
            DROP CONSTRAINT "${parentIdFkName}",
            ADD CONSTRAINT "${parentIdFkName}"
            FOREIGN KEY ("${parentIdColumnName}")
            REFERENCES ${tableName}(id)
            ON DELETE CASCADE;

            ALTER TABLE ${closureTableName}
            DROP CONSTRAINT "${closureAncestorFkName}",
            ADD CONSTRAINT "${closureAncestorFkName}"
            FOREIGN KEY ("${closureAncestorColumnName}")
            REFERENCES ${tableName}(id)
            ON DELETE CASCADE;

            ALTER TABLE ${closureTableName}
            DROP CONSTRAINT "${closureDescendantFkName}",
            ADD CONSTRAINT "${closureDescendantFkName}"
            FOREIGN KEY ("${closureDescendantColumnName}")
            REFERENCES ${tableName}(id)
            ON DELETE CASCADE;
    `);
}
```

Then we just need to run `addOnDeleteCascades` during app initialization. In nestjs we can use `OnModuleInit` to hook into the database module lifecycle to do this.
