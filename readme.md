If we only had custom fields for a million records we would use a SQL database such as postgreSQL.

Now that we have custom fields, if they all were of the same data type with a SQL database we would have the option to have a second table for our custom fields.

```sql
create type status_types as enum ('pending','doing' 'done', 'tested');

create table task(
    id serial primary key,
    name text, 
    description text, 
    status status_types
);

create table task_field(
    id serial primary key, 
    task_id int references task(id),
    field_name text, 
    field_value text
)
```

Then Querying would be like:

```sql
    select t.id, t.name, t.description, t.status
    from task t 
    join task_field tf1 on tf1.task_id = tf1.id
    join task_field tf2 on tf2.task_id = tf2.id
    where t.status = 'done' 
    and tf1.field_name = 'field1' and tf1.field_value = 'value1'
    and tf2.field_name = 'field2' and tf2.field_value = 'value2'
```

## What are pros and cons of this approach?
### Pros:
1. The main tasks table is small:
if each record is around 1 KB, 1 million records gives 1 GB of uncompressed data so PostgreSQL can keep it inside RAM, which would be ideal for fast querying (if we do not involve custom fields)
2. This is a standard relational structure (also normalized) so we have relational integrity.
3. If we want to add a custom field we wouldn't change the main table.

### Cons:
1. Complex Querying: Each time we want access to a custom field would need to perform a join
2. Complex Indexing: Since we have `task_field.field_name and task_field.field_value` inside our queries, we would need to create index for this query. 

## What indexes do we need in this approach

1. Indexes for the static fields on `tasks` table (because they are used in the where clause)
2. Indexes for the custom fields in the query `task_field.field_name and task_field.field_value`
3. `tasks.id = task_field.task_id` this is used for joining using a foreign key, `tasks.id` is the primary key of the `tasks` table which already has an index, but the `task_field.task_id` needs to be indexed.


## What if custom fields are of different types (text, number, links)?
In the above approach we would use one table for each type giving the following tables:

```sql
create table task_field_text(
    id serial primary key, 
    task_id int references task(id),
    field_name text, 
    field_value text
)

create table task_field_integer(
    id serial primary key, 
    task_id int references task(id),
    field_name text, 
    field_value integer
)
```

## Is there any alternatives to the above design
My favorite approach is using `JSONB` fields in the PostgreSQL database. 
JSONB has the required performance and operators, also allows for varying fields.
Had we decided to use `JSONB` our schema would be like:


```sql
create type status_types as enum ('pending','doing' 'done', 'tested');

create table task(
    id serial primary key,
    name text, 
    description text, 
    status status_types,
    custom_fields JSONB
);
```

Then we can use JSONB operators for our queries.

```sql
select t.id, t.name, t.description, t.status
from task t 
where custom_fields @> '{"field1": "value1", "field2": "value2"}';
```
And this query would need indexes on the `custom_fields`:
```sql
create index idx_custom_fields_field1 ON task USING GIN ((custom_fields ->> 'field1'));
create index idx_custom_fields_field2 ON task USING GIN ((custom_fields ->> 'field2'));
```


And this is the solution I would go for, but this is not all.


# Method3: Using MongoDB
Since we have varying fields, we could also use MongoDB for this database.
Selecting MongoDB is not just about fields; the volume of data, and our backend technologies,
our deployment environment, our data sensitivity, the team expertise, could affect our decision.

This is the structure of our Documents in MongoDB:

```json
{
    "_id": ObjectId("..."),
    "name": "Task Name",
    "description": "Description of Task",
    "status": "Done",
    "customFields": {
        "field1": "value1",
        "field4": "user_id"
        "field2": 123,
        "field3": "2023-11-23",
        // Other custom fields...
    }
}
```

We can query by:
```js
    db.tasks.find({"status":"done", "field1": "value1", "field2": "value2"})
```

And the indexes we need:

```js
    db.tasks.createIndex({ "status": 1, "custom_fields.field1": 1, "custom_fields.field2": 1 })
```




