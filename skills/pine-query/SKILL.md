---
name: pine-query
description: "Write pine/SBVR API queries for balena's backend. Use this skill whenever the user asks to write, create, or construct a pine query — including get, post, patch, or delete calls against resources like device, user, application, organization, subscription, etc. Trigger this skill when the user mentions \"pine query\", \"pine get/post/patch/delete\", \"resin api query\", \"sbvr query\", or asks to query/fetch/create/update/delete any balena resource using the pine client API, even if they don't explicitly say \"pine\"."
---

# Pine Query Skill

Write pine client API queries for balena's SBVR backend. Pine is the TypeScript OData client used by open-balena-api and pinejs to interact with resources like `device`, `application`, `release`, `user`, `api_key`, `image`, `service_install`, and many others.

## Query Structure

Every pine query is an object passed to a method on a pine client instance. The client can be `api.resin`, `rootApi`, `pineUser`, `sbvrUtils.api.Auth`, `api.tasks`, or similar — the name varies by context, but the query shape is always the same.

### Methods

| Method | Purpose |
|--------|---------|
| `.get()` | Read one or many resources |
| `.post()` | Create a resource |
| `.patch()` | Update resource(s) |
| `.delete()` | Delete resource(s) |
| `.prepare()` | Create a reusable parameterized query |
| `.request()` | Raw OData request (e.g. canAccess actions) |
| `.put()` | Upsert (create or replace) |

### Common Query Properties

```typescript
{
  resource: 'device',            // required — the resource type
  id: 123,                       // target a single resource by numeric id
  // OR
  id: { uuid: 'abc-123' },      // target by natural key
  // OR
  id: { device: 1, installs__service: 2 }, // composite natural key

  passthrough: {                 // execution context
    req: permissions.root,       // or permissions.rootRead for read-only
    tx,                          // database transaction
  },

  options: {
    $select: 'field',            // or ['field1', 'field2']
    $filter: { ... },            // where clause
    $expand: { ... },            // join related resources
    $orderby: { field: 'asc' }, // or array, or string
    $top: 10,                    // limit results
    $count: { $filter: { ... } }, // count matching resources
    returnResource: false,       // POST only — return just { id } instead of full resource
  },

  body: { ... },                 // data for POST/PATCH/PUT
}
```

## Filtering ($filter)

Filters are the most common and nuanced part of pine queries. Multiple conditions at the same level are implicitly ANDed.

### Comparison Operators

```typescript
// Equality (implicit)
$filter: { status: 'success' }

// Explicit operators
$filter: { cpu_temp: { $gt: 37 } }      // greater than
$filter: { cpu_temp: { $lt: 36 } }      // less than
$filter: { created_at: { $ge: date } }  // greater or equal
$filter: { created_at: { $le: date } }  // less or equal
$filter: { created_at: { $eq: date } }  // explicit equal
$filter: { revision: { $ne: null } }    // not equal / not null
$filter: { name: { $in: ['a', 'b'] } }  // in array
```

### String Operators

```typescript
// Ends with
$filter: { is_stored_at__image_location: { $endswith: path } }

// Case-insensitive comparison
$filter: {
  $eq: [
    { $tolower: { $: 'username' } },
    { $tolower: loginInfo },
  ],
}
```

### Logical Operators

```typescript
// $or — array form (any condition matches)
$filter: {
  $or: [
    { expiry_date: null },
    { expiry_date: { $gt: { $now: null } } },
  ],
}

// $or — object form (shorthand for field-level OR)
$filter: {
  $or: {
    os_version: { $ne: null },
    supervisor_version: { $ne: null },
  },
}

// $and — explicit (needed when same-level keys would collide)
$filter: {
  $and: [
    { is_scheduled_with__cron_expression: { $ne: null } },
    { is_scheduled_with__cron_expression: { $ne: cron } },
  ],
}

// $not — negate a block
$filter: {
  $not: {
    permission: { $in: perms },
  },
}
```

### Null Checks

```typescript
$filter: { supervisor_version: null }        // is null
$filter: { should_be_running__release: { $ne: null } } // is not null
```

### Current Timestamp ($now)

```typescript
$filter: { expiry_date: { $gt: { $now: {} } } }
```

## Relationship Traversal ($any)

`$any` lets you filter a resource based on properties of its related resources. It translates to an OData `any()` lambda. The `$alias` names the loop variable, and `$expr` defines the condition.

Always use `$any` expressions when filtering on related resources — never use nested property filters directly. Pine does not support dot-notation or inline nested filters like `$filter: { belongs_to__application: { slug: 'myapp' } }`. The only correct way to filter across a relationship is through `$any` with `$alias` and `$expr`.

### One-level traversal

```typescript
// Get public keys where the related user has a specific username
$filter: {
  user: {
    $any: {
      $alias: 'u',
      $expr: { u: { username: 'josh' } },
    },
  },
}
```

### Multi-level (nested) traversal

Chain `$any` to traverse multiple relationships deep:

```typescript
// Get users where actor -> api_key -> key matches
$filter: {
  actor: {
    $any: {
      $alias: 'a',
      $expr: {
        a: {
          api_key: {
            $any: {
              $alias: 'k',
              $expr: { k: { key: apiKeyValue } },
            },
          },
        },
      },
    },
  },
}
```

### Existence checks with $any

Use `$any` with `$ne: null` to check that a relationship exists, or wrap in `$not` to find orphans:

```typescript
// Find permissions not attached to any role or api_key
$filter: {
  $not: {
    $or: [
      { is_of__role: { $any: { $alias: 'r', $expr: { r: { id: { $ne: null } } } } } },
      { is_of__api_key: { $any: { $alias: 'a', $expr: { a: { id: { $ne: null } } } } } },
    ],
  },
}
```

## Selection & Expansion

### $select

```typescript
$select: 'slug'                    // single field (string)
$select: ['id', 'slug', 'name']   // multiple fields (array)
```

### $expand

Expand loads related resources inline. Supports nested `$select`, `$filter`, `$expand`, `$top`, `$orderby`:

```typescript
// Simple expand with select
$expand: {
  is_for__device_type: { $select: 'slug' },
}

// Multiple expand targets
$expand: {
  is_of__user: { $select: ['id', 'username'] },
  is_of__application: { $select: ['id', 'slug'] },
  is_of__device: { $select: ['id', 'uuid'] },
}

// Deep expand with filter and orderby
$expand: {
  owns__release: {
    $top: 1,
    $select: 'id',
    $filter: { commit: releaseCommit, status: 'success' },
    $orderby: { revision: 'desc' },
  },
}

// Expand with nested $any filter
$expand: {
  service_install: {
    $select: 'installs__service',
    $filter: {
      installs__service: {
        $any: {
          $alias: 'is',
          $expr: { is: { application: { $ne: appId } } },
        },
      },
    },
  },
}
```

## Ordering

```typescript
// Single field — object
$orderby: { revision: 'desc' }

// Multiple fields — array of objects
$orderby: [
  { is_invalidated: 'asc' },
  { revision: 'desc' },
  { created_at: 'desc' },
]

// String form
$orderby: 'created_at asc'

// By related resource property (string path)
$orderby: ['is_pinned_on__release/commit desc']

// By filtered related resource property
$orderby: [`application_tag(tag_key='sorting-tag')/value asc`, { app_name: 'asc' }]

// By deep nested property
$orderby: { 'installs__image/is_a_build_of__service/service_name': 'asc' }

// By $count of related resource
$orderby: { application_tag: { $count: {} }, $dir: 'desc' }

// By filtered $count
$orderby: [
  { application_tag: { $count: { $filter: { value: '0' } } }, $dir: 'desc' },
  { app_name: 'asc' },
]
```

## Counting

```typescript
// Count resources matching a filter
options: {
  $count: {
    $filter: {
      id: { $in: deviceIds },
      supervisor_version: null,
    },
  },
}
```

## Prepared Queries (.prepare)

Prepared queries are reusable parameterized queries. Parameters use `{ '@': 'paramName' }` as placeholders. The second argument declares the parameter types.

```typescript
const query = api.resin.prepare(
  {
    resource: 'device',
    passthrough: { req: permissions.root },
    id: { uuid: { '@': 'uuid' } },
    options: { $select: ['id', 'is_frozen'] },
  },
  { uuid: ['string'] },
);

// Later, execute with:
const result = await query({ uuid: 'abc-123' });
```

Parameters can appear in `id`, `$filter`, or nested inside `$any`/`$expand`:

```typescript
api.resin.prepare(
  {
    resource: 'application',
    options: {
      $top: 1,
      $select: 'id',
      $expand: {
        owns__release: {
          $top: 1,
          $select: 'id',
          $filter: { commit: { '@': 'commit' }, status: 'success' },
          $orderby: { revision: 'desc' },
        },
      },
      $filter: {
        owns__device: {
          $any: {
            $alias: 'd',
            $expr: { d: { id: { '@': 'deviceId' } } },
          },
        },
      },
    },
  },
  { deviceId: ['number'], commit: ['string'] },
)
```

## Common Patterns

### Optimistic Update (skip no-op writes)

Only update if current values differ from the body. Avoids unnecessary writes and triggers:

```typescript
api.patch({
  resource: 'device',
  id: deviceId,
  options: { $filter: { $not: deviceBody } },
  body: { ...deviceBody, ...metricsBody },
})
```

### Guarded Update (status check)

Only update if a precondition holds:

```typescript
rootApi.patch({
  resource: 'scheduled_job_run',
  id: scheduledJobRun.id,
  options: { $filter: { status: { $ne: 'success' } } },
  body: { status: 'error', end_timestamp },
})
```

### Get-or-Insert

```typescript
const existing = await apiTx.get({
  resource,
  options: { $select: 'id', $filter: body },
} as const);
if (existing != null) return existing.id;
const { id } = await apiTx.post({
  resource,
  body,
  options: { returnResource: false },
});
return id;
```

### canAccess Check

```typescript
api.resin.request({
  method: 'POST',
  url: `device(uuid=@uuid)/canAccess?@uuid='${uuid}'`,
  passthrough: { req },
  body: { action: 'cloudlink' },
})
```

### Task Queue

```typescript
sbvrUtils.api.tasks.post({
  resource: 'task',
  passthrough: { req: permissions.root, tx },
  body: {
    is_executed_by__handler: 'create_service_installs',
    is_executed_with__parameter_set: { devices: deviceIds },
    is_scheduled_with__cron_expression: null,
    attempt_limit: 5,
  },
})
```

### Actions (beginUpload / commitUpload)

```typescript
api.post({
  resource: 'organization',
  action: 'beginUpload',
  id: org.id,
  body: {
    logo_image: {
      filename: 'logo.png',
      content_type: 'image/png',
      size: 6291456,
      chunk_size: 6000000,
    },
  },
})
```

## TypeScript Typing

### `as const` for type narrowing

Append `as const` to the query object so TypeScript narrows the return type based on `$select`/`$expand`:

```typescript
const result = await pineUser.get({
  resource: 'service_install',
  options: {
    $expand: { installs__service: { $select: ['id', 'service_name'] } },
    $filter: { device: deviceId },
  },
} as const);
// result is typed with only the selected/expanded fields
```

### Generic type parameter

```typescript
const dt = await pineUser.get<DeviceType['Read']>({
  resource: 'device_type',
  id: { slug: 'intel-nuc' },
  options: { $select: 'id' },
});
```

## Passthrough Context

- `permissions.root` — full read/write access, bypasses permission checks
- `permissions.rootRead` — read-only root access
- `tx` — database transaction for atomic operations
- `custom: { ... }` — pass additional context (e.g. `clientIP`)

Use `permissions.rootRead` instead of `permissions.root` for read-only operations (`.get()` calls). This follows the principle of least privilege — granting only the access level the operation actually needs reduces the blast radius if a query is accidentally misused or modified later.

```typescript
passthrough: { req: permissions.root, tx }
passthrough: { req: permissions.rootRead }
passthrough: { tx, req: permissions.root, custom: { clientIP } }
```

## API Prefix

Target a different model/vocabulary by specifying `apiPrefix`:

```typescript
api.post({
  apiPrefix: '/example/',
  resource: 'device',
  body: { name: 'test', type: 'rpi4' },
})
```
