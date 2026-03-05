---
name: drizzle-orm
description: Examples for drizzle-orm and guides for how to get the latest documentation and API references for drizzle-orm.
alwaysApply: false
---

# Drizzle ORM Skill
## Latest Documentation
Read https://orm.drizzle.team/llms.txt and see where to find the latest documentation and API references for drizzle-orm.

# Our Best Practices
## DB Connection with Bun SQL via Unix Socket
```ts
import { drizzle } from 'drizzle-orm/bun-sql'
import { SQL } from 'bun'

const dbClient = new SQL({
  path: "/var/run/postgresql/.s.PGSQL.5432",
  username: "db_user",
  database: "db_name",
  // maximum number of connections in the pool
  max: 4,
  idleTimeout: 60,
  maxLifetime: 3600,
  connectionTimeout: 10,
  onconnect: () => {
    console.log('Connection to PostgreSQL established')
  }
})

export const db = drizzle({ client: dbClient })
```
