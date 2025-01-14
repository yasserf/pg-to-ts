# pg-to-ts

This is a personal fork of [PYST/schemats][pyst-fork], which is a fork of [SweetIQ/schemats][orig-repo].

Usage:

    npm install pg-to-ts
    pg-to-ts generate -c postgresql://user:pass@host/db -o dbschema.ts

The resulting file looks like:

```ts
// Table product
export interface Product {
  id: string;
  name: string;
  description: string;
  created_at: Date;
}
export interface ProductInput {
  id?: string;
  name: string;
  description: string;
  created_at?: Date;
}
const product = {
  tableName: 'product',
  columns: ['id', 'name', 'description', 'created_at'],
  requiredForInsert: ['name', 'description'],
} as const;

export interface TableTypes {
  product: {
    select: Product;
    input: ProductInput;
  };
}

export const tables = {
  product,
};
```

This gives you most of the types you need for static analysis and runtime.

## Features

### Comments

If you set a Postgres comment on a table or column:

```sql
COMMENT ON TABLE product IS 'Table containing products';
COMMENT ON COLUMN product.name IS 'Human-readable product name';
```

Then these come out as JSDoc comments in the schema:

```ts
/** Table containing products */
export interface Product {
  id: string;
  /** Human-readable product name */
  name: string;
  description: string;
  created_at: Date;
}
```

The TypeScript language service will surface these when it's helpful.

### Dates as strings

node-postgres returns timestamp columns as JavaScript Date objects. This makes
a lot of sense, but it can lead to problems if you try to serialize them as
JSON, which converts them to strings. This means that the serialized and de-
serialized table types will be different.

By default `pg-to-ts` will put `Date` types in your schema file, but if you'd
prefer strings, pass `--datesAsStrings`. Note that you'll be responsible for
making sure that timestamps/dates really do come back as strings, not Date objects.
See <https://github.com/brianc/node-pg-types> for details.

### JSON types

By default, Postgres `json` and `jsonb` columns will be typed as `unknown`.
This is safe but not very precise, and it can make them cumbersome to work with.
Oftentimes you know what the type should be.

To tell `pg-to-ts` to use a specific TypeScript type for a `json` column, use
a JSDoc `@type` annotation:

```sql
ALTER TABLE product ADD COLUMN metadata jsonb;
COMMENT ON COLUMN product.metadata is 'Additional information @type {ProductMetadata}';
```

On its own, this simply acts as documentation. But if you also specify the
`--jsonTypesFile` flag, these annotations get incorporated into the schema:

    pg-to-ts generate ... --jsonTypesFile './db-types' -o dbschema.ts

Then your `dbschema.ts` will look like:

```ts
import {ProductMetadata} from './db-types';

interface Product {
  id: string;
  name: string;
  description: string;
  created_at: Date;
  metadata: ProductMetadata | null;
}
```

Presumably your `db-types.ts` file will either re-export this type from elsewhere:

```ts
export {ProductMetadata} from './path/to/this-type';
```

or define it itself:

```ts
export interface ProductMetadata {
  year?: number;
  designer?: string;
  starRating?: number;
}
```

## Development Quickstart

    git clone https://github.com/danvk/schemats.git
    cd schemats
    npm install
    npm run build

    node bin/schemats.js generate -c postgresql://user:pass@host/db -o dbschema.ts

See [SweetIQ/schemats][orig-repo] for the original README.

[orig-repo]: https://github.com/SweetIQ/schemats
[pyst-fork]: https://github.com/PSYT/schemats
