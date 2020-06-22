# Deleting a subtree in a closure table

__14/08/2018__

![tree](https://imgs.xkcd.com/comics/tree.png)

At CTcue we model a users search query as a tree and store it in postgres using a closure table. Why and how we model it as a a tree deserves it's own blog post, but suffice to say, we think there are advantages. We use [TypeORM](https://github.com/typeorm/typeorm) and it provides a [closure table implementation](https://github.com/typeorm/typeorm/blob/master/docs/tree-entities.md#closure-table) except it doesn't support deletion.

## Implementation

### Entity

For the example I've removed all the specifics.

```typescript
import { Column, Entity, OneToMany, OneToOne, Tree, Treenodes, TreeParent } from "typeorm";

@Entity()
@Tree("closure-table")
export class Query {

    @PrimaryGeneratedColumn("uuid") readonly id: string;

    @TreeParent() parent: Query;

    @Treenodes()
    groups: Query[];
}
```

## Deletion

A lot of this implementation was based on the slides in [models for hierarchical data](https://www.slideshare.net/billkarwin/models-for-hierarchical-data).

When using a closure table, the tree is stored in two tables, a table which describes the entity (entity table) and a table which describes the ancestor/descendant relationships (closure table).

### Entity table

|id|parent|
|-|-|
|1|null|
|2|1|
|3|1|
|4|2|

### Closure table

|ancestor|descendant|
|-|-|
|1|1|
|1|2|
|1|3|
|1|4|
|2|2|
|2|4|
|4|4|
|3|3|

To delete a subtree we have to remove the relevant rows from the closure table, remove the parent foreign key from the entity table and then finally remove the entities themselves.

*Note*: Some implementations of the `closure table pattern` don't use a parent foreign key in the entity table, but TypeORM does, so we'll work with it.

So if we were to remove the subtree beginning with id `2`, the resulting tables would look like:

|id|parent|
|-|-|
|1|null|
|3|1|

|ancestor|descendant|
|-|-|
|1|1|
|1|3|
|3|3|

Notice that we had to remove two rows from the entity table and 5 rows from the closure table. Also notice that the closure table holds the ids of the entity table, so we can use it to delete the rows from the entity table. Finally also notice that the closure table holds a self-relationships, e.g 1->1 or 2->2.

In pseudo code:

```text
let ids = descendants where ancestor = 2
delete rows from the closure table where descendant in ids
set parent key to null where the parent key in ids
delete rows from the entity table where id in ids
```

And the code will look like:

```typescript
@EntityRepository(Query)
@Loggable()
export class QueryRepository extends TreeRepository<Query> {

    async delete(id: string): Promise<DeleteResult> {
        const result = await getManager().transaction(async (tem: EntityManager) => {
            const entity = await tem.findOneOrFail(Query, id, { relations: ["parent"] });

            if (!entity.parent) {
                // throw error
                ...
            }

            const table = this.metadata.closureJunctionTable.ancestorColumns[0].entityMetadata
                .tableName;
            const ancestor = this.metadata.closureJunctionTable.ancestorColumns[0].databasePath;
            const descendant = this.metadata.closureJunctionTable.descendantColumns[0].databasePath;

            const nodes = await tem
                .createQueryBuilder()
                .select(descendant)
                .from(table, "closure")
                .where(`${ancestor} = :id`, { id: id })
                .getRawMany();

            const nodeIds = node.map((v) => v[descendant]);

            // delete all the nodes from the closure table
            await tem
                .createQueryBuilder()
                .delete()
                .from(table)
                .where(`${descendant} IN (:...ids)`, { ids: nodeIds })
                .execute();

            // delete the parent foreign key from the queries
            // otherwise we'll get a fk constraint when trying to delete
            await tem
                .createQueryBuilder()
                .relation(Query, "parent")
                .of(nodeIds)
                .set(null);

            // delete the queries
            await tem.delete(Query, nodeIds);

            return nodeIds;
        });

        return { raw: result };
    }
}
```
