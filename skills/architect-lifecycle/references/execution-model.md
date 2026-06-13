# Lifecycle Function Execution Model

Reference for the server-side runtime that executes lifecycle functions: the
sandbox, the `services` object, retry behavior, and error types.

> The specific limits below (timeouts, memory, retry counts) describe the
> lifecycle runtime's intended contract. Verify them against your deployment
> before relying on exact numbers — they may change as the runtime evolves.

## Runtime Environment

Lifecycle functions execute in an **isolated V8 sandbox**:

- Separate from the main Architect process
- No access to Node.js APIs
- No access to file system
- Controlled network access via `services.http`

## Constraints

| Constraint             | Value                     | Behavior on Violation            |
| ---------------------- | ------------------------- | -------------------------------- |
| **Max execution time** | 10 seconds                | Killed, error returned           |
| **Memory limit**       | 128 MB                    | Killed, error returned           |
| **Network access**     | `services.http` only      | Direct fetch/axios not available |
| **File system**        | None                      | fs operations not available      |
| **Child processes**    | None                      | spawn/exec not available         |
| **Timers**             | setTimeout only (max 10s) | setInterval not available        |

## Available Globals

```javascript
// Available
console.log()    // Alias for services.log()
JSON             // Parse/stringify
Date             // Date operations
Math             // Math functions
Object           // Object methods
Array            // Array methods
String           // String methods
Promise          // Async operations
RegExp           // Regular expressions

// NOT Available
fetch            // Use services.http instead
require          // No module system
import           // No ES modules
process          // No Node.js
fs               // No file system
Buffer           // No Node.js buffers
```

## Services Object

### services.http

Make HTTP requests to external services.

```javascript
// GET request
const response = await services.http.get('https://api.example.com/data')
// response: { status: 200, data: {...}, headers: {...} }

// POST request
const result = await services.http.post(
  'https://api.example.com/webhook',
  { event: 'created', data: entry.data },
  { headers: { Authorization: 'Bearer token' } }
)

// PUT request
await services.http.put('https://api.example.com/sync/' + entry.id, entry.data)

// DELETE request
await services.http.delete('https://api.example.com/cache/' + entry.id)
```

**Constraints:**

- Timeout: 5 seconds per request
- Max redirects: 3
- Allowed protocols: HTTPS only (HTTP blocked)
- Response size limit: 1 MB

### services.entries

Query and modify entries within the same organization/environment.

```javascript
// Find entries
const products = await services.entries.find({
  modelId: 'product',
  filter: { category: 'electronics' },
  limit: 10,
})

// Get single entry
const author = await services.entries.get('entry_author_123')

// Create entry (use carefully - can trigger infinite loops!)
const newEntry = await services.entries.create({
  modelId: 'auditLog',
  data: { action: 'created', entryId: entry.id },
})

// Update entry
await services.entries.update('entry_xxx', {
  data: { lastSynced: new Date().toISOString() },
})
```

**Warning:** Creating/updating entries from lifecycle functions can trigger other lifecycle functions. Guard against infinite loops:

```javascript
// Bad - infinite loop
function handler(entry, context, services) {
  await services.entries.update(entry.id, { counter: entry.data.counter + 1 });
  return { entry };
}

// Good - guard against recursion
function handler(entry, context, services) {
  if (entry.data._triggeredByLifecycle) {
    return { entry }; // Skip if already processed
  }
  entry.data._triggeredByLifecycle = true;
  // ... rest of logic
  return { entry };
}
```

### services.log

Write to function execution logs.

```javascript
services.log('Processing entry:', entry.id)
services.log('User context:', context.user)
services.log('Validation passed')
```

Logs are stored and viewable in the Admin UI under the function's execution history.

### services.kv

Simple key-value storage scoped to the function.

```javascript
// Store a value
await services.kv.set('lastRunTime', Date.now())

// Retrieve a value
const lastRun = await services.kv.get('lastRunTime')

// Delete a value
await services.kv.delete('lastRunTime')
```

**Constraints:**

- Key size: max 256 characters
- Value size: max 64 KB (JSON serialized)
- Total storage per function: 1 MB

## Retry Semantics

Different events have different retry behavior:

| Event          | Retry on Failure | Reason                           |
| -------------- | ---------------- | -------------------------------- |
| `beforeSave`   | No               | Synchronous, blocks save         |
| `afterSave`    | Yes (3x)         | Async, should eventually succeed |
| `beforeDelete` | No               | Synchronous, blocks delete       |
| `afterDelete`  | Yes (3x)         | Async, cleanup should complete   |

### Retry Schedule

For `afterSave` and `afterDelete`:

1. **Immediate retry** after initial failure
2. **30 seconds** after first retry failure
3. **5 minutes** after second retry failure
4. **Marked as failed** after third retry

### Idempotency

Since `afterSave`/`afterDelete` may retry, write idempotent code:

```javascript
// Bad - may send duplicate notifications
function handler(entry, context, services) {
  await services.http.post('https://slack.com/webhook', {
    text: `Entry ${entry.id} created`
  });
  return { success: true };
}

// Good - use idempotency key
function handler(entry, context, services) {
  await services.http.post('https://slack.com/webhook', {
    text: `Entry ${entry.id} created`
  }, {
    headers: {
      'X-Idempotency-Key': `${entry.id}-${context.event}-${entry.version}`
    }
  });
  return { success: true };
}
```

## Error Types

### ValidationError

Thrown by your code to block an operation:

```javascript
throw new Error('Price cannot be negative')
```

Result: Operation blocked, error message shown to user.

### TimeoutError

Function exceeded the 10 second limit:

Result:

- `beforeSave`/`beforeDelete`: Operation blocked
- `afterSave`/`afterDelete`: Retry scheduled

### RuntimeError

Uncaught exception in your code:

```javascript
// This will throw RuntimeError
const x = entry.data.nonexistent.property
```

Result: Same as TimeoutError.

### NetworkError

HTTP request failed:

```javascript
// May throw NetworkError
await services.http.get('https://unreachable-host.com')
```

Result: Caught in your try/catch, or bubbles up as RuntimeError.

## Debugging

### View Logs

In the Admin UI: Models → [Model] → Lifecycle Functions → [Function] → Execution History.

### Log Output

```javascript
function handler(entry, context, services) {
  services.log('=== Start ===')
  services.log('Entry ID:', entry.id)
  services.log('Event:', context.event)
  services.log('Is New:', context.isNew)

  try {
    // ... logic
    services.log('Success')
  } catch (error) {
    services.log('Error:', error.message)
    throw error
  }

  return { entry }
}
```

### Test Before Relying On It

Lifecycle functions run on real content events, so validate against a
throwaway entry first: create or update an entry for the model via
`architect entries` (or the Admin UI) with data that should trigger the
handler, then check the function's execution history for the result and any
`services.log` output before depending on it in production.
