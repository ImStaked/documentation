# Advanced Access to the SubQuery Store

The SubQuery store is an injected class that allows users to interact with records in the database from within mapping functions. This will come handy when user demands using multiple entity records as the parameters in the mapping function, or create/update multiple records in a single place.

::: info Note
Note that there are additional methods autogenerated with your entities that also interact with the store. Most users will find those methods sufficient for their projects.
:::

Following is a summary of the `Store` interface:

```typescript
export interface Store {
  get(entity: string, id: string): Promise<Entity | null>;
  getByField(
    entity: string,
    field: string,
    value: any,
    options?: { limit?: number, offset?: number }
  ): Promise<Entity[]>;
  getOneByField(
    entity: string,
    field: string,
    value: any
  ): Promise<Entity | null>;
  set(entity: string, id: string, data: Entity): Promise<void>;
  bulkCreate(entity: string, data: Entity[]): Promise<void>;
  bulkUpdate(entity: string, data: Entity[], fields?: string[]): Promise<void>;
  remove(entity: string, id: string): Promise<void>;
}
```

## Get Record

`get(entity: string, id: string): Promise<Entity | null>;`

This allows you to get a record of the entity with its `id`.

```typescript
const id = block.block.header.hash.toString();
await store.get(`StarterEntity`, id);
```

## Get All Records by Field

`getByField(entity: string, field: string, value: any, options?: { limit?: number; offset?: number }): Promise<Entity[]>;`

This returns matching records for the specific entity that matches a given search. By default it will return the first 100 results.
The number of results can be changed via the `query-limit` flag for the node or via the options field. If you need more than the number of results provided you can also specify an `offset` and page your results.

```typescript
// Get all records with field1 == 50
await store.getByField(`StarterEntity`, "field1", 50);
```

Please note, the third parameter also accepts array, you can consider this similar like `bulkGet` with OR search.
To get a list of records with `field1` equal to 50, 100 or 150:

```typescript
// Get all records with field1 == 50 OR field1 == 100 OR field1 == 150
await store.getByField("StarterEntity", "field1", [50, 100, 150]);
```

## Get First Record by Field

`getOneByField(entity: string, field: string, value: any): Promise<Entity | null>;`

This returns the first matching record for the specific entity that matches a given search.

```typescript
const field1Value = 50;
await store.getOneByField(`StarterEntity`, `field1`, field1Value);
```

## Upsert (Create and Update) Record

`set(entity: string, id: string, data: Entity): Promise<void>;`

This allows user to create a single record, if the record already exist this will overwrite its record.

```typescript
const id = block.block.header.hash.toString();
await store.set(`StarterEntity`,id, {field1: 50, ...})
```

## Bulk Create Records

`bulkCreate(entity: string, data: Entity[]): Promise<void>;`

This allows to create multiple records for specified entity, but it will not overwrite existing records.

```typescript
await store.bulkCreate(`StarterEntity`,[
    {id: 1, field1: 50, ...},
    {id: 2, field1: 100, ...},
    {id: 3, field1: 150, ...}
])
```

## Bulk Upsert (Create and Update) Records

`bulkUpdate(entity: string, data: Entity[], fields?: string[]): Promise<void>;`

This allows to update multiple records for specified entity, it will create the records if they are not exist.

```typescript
await store.bulkUpdate(`StarterEntity`,[
    {id: 1, field1: 99, ...},
    {id: 2, field1: 199, ...},
    {id: 3, field1: 299, ...}
])
```

The 3rd parameter is optional, and allows user to provide a list of fields they wish to be updated, and other fields will be ignored.

For example, only `field5` will be updated.

```typescript
await store.bulkUpdate(`StarterEntity`,[
    {id: 1, field1: 99, field5: true, ...},
    {id: 2, field1: 199, field5: false,...},
    ['field5']
])
```

Please note, this fields feature is not working currently with any automated historical indexing. It will overwrite all attributes. To disable automated historical indexing, please enable `--disable-historical=true` parameter on `subql-node`.

## Remove Record

`remove(entity: string, id: string): Promise<void>;`

This allows to remove a single record of the entity with its `id`.

```typescript
const id = block.block.header.hash.toString();
await store.remove(`StarterEntity`, id);
```