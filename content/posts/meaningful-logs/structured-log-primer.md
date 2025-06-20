+++
date = '2025-06-20T14:44:40-07:00'
draft = false
title = 'Structured Log Primer'
series = ["Meaningful Logs"]
tags = ["logging", "development"]
weight = 2
+++
Background and best practices for structured logging.

# Context

Structured logging is the practice of attaching key value pairs to event logs for the purpose of simplifying log search and consumption. For more motivating details see the first article in the series Useful Logs.

# Anatomy

Structured logs can be thought of as dictionaries with a dedicated field holding the log `message` and the remaining fields adding `context`. That's really all there is to it!

```jsx
{
  "event": "Finished processing user data",
  "workflow_id": "64aef9ca51b691ce2d32a6cc",
  "task_id": "64aefa0455e6c5d7a1263d5e",
  "level": "info",
  "timestamp": "2023-07-12T19:16:39.864541Z"
}
```

# Best practices

While the format may be simple, effectively utilizing structured logging at a company or even project level is nuanced. Below are a few patterns to keep in mind as you get started.

1. Use consistent key names
2. Build your log context
3. Use descriptive names which are likely to be unique
4. Use primitive values for search fields

## **Consistent key names**

The primary benefit of structured logs is that they're searchable by the included fields. The simple task of searching by an indexed field becomes complex if the field you're interested in is captured across a variety of different keys.

Imagine the `id` of a session is attached to logs under several names: `id`, `session_id`, `sid`, and `s_uid`. Anyone who wants to query for all logs related to the session will need to both be aware of all variations of the field name and match each variation in their query. This process is both cumbersome and error prone.

### **How to**

Constants

The best way to enforce consistent fields names is to define a constant holding the key. Use this constant every time you log.

Example using Python `structlog` (later articles in the series will dig deeper into this library):

```python
import structlog
logger = structlog.stdlib.get_logger().new()

SESSION_ID_LOG_KEY = "session_id"
session_id = "64aef9ca51b691ce2d32a6cc"
logger.info("Testing constant field names", **{SESSION_ID_LOG_KEY: session_id})
```

Output

```python
2025-03-01T19:42:21.881139Z [Info    ] Testing constant field names      session_id="64aef9ca51b691ce2d32a6cc"
```

## Build your log context

Many structured logging libraries have utilities for building a running set of structured fields over the course of a process. This set of fields is generally knowns as the "log context" and can be attached automatically to future log lines.

Use these methods to set fields relevant to downstream logs as soon as they're available. This will both cut down on the potential for variability in key names and reduce the possibility of forgetting to attach keys to downstream logs.

### How to

When working with loggers you're likely to find yourself working with both `mutable` and `immutable` context. Each type has its merits as both are well suited to specific logging patterns and scenarios. You'll become familiar with both patterns throughout the series. For now, just know know that sometimes you'll make a copy of your context and other times you'll modify shared state.

Add to an immutable context

**Outcome:** New fields are only made available to downstream calls. 

As the original context is immutable "adding to the context" actually results in making a copy of the original context and including the new field. Use this style of logging when:

- a structured field is only relevant to a confined scope.
- a log object is `injected` into a downstream layer (e.g. a class or method expects to receive a logger).

```python
import structlog

logger = structlog.stdlib.get_logger().new()
# Adding a field through "bind" makes a copy of the context on `logger`
# then adds the new field to the copy.
log = logger.bind(service_name="user_store")
# This logger has the "service_name" field.
log.info("starting service")
# This logger has no fields.
logger.info("original logger")

# Process user data
data = {"user_name": "Joe"}
user = process_user_data(log, data)
# This line will have the "service_name" field.
log.info("completed processing")

def process_user_data(log: BoundLogger, data: dict[str, Any]) -> User:
    # Bind a field to a logger only accessible within this method.
    inner_log = log.bind(method="process_user_data")
    # This log will have both the "service_name" and "method" fields.
    inner_log.info("processing data for user")
    return User(**data)
 
```

Output

```python
2025-03-01T19:42:21.881139Z [Info    ] starting service                service_name="user_store"
2025-03-01T19:42:21.891139Z [Info    ] original logger
2025-03-01T19:42:21.911139Z [Info    ] processing data for user        service_name="user_store" method="process_user_data"
2025-03-01T19:42:22.001139Z [Info    ] completed processing            service_name="user_store"
```

Add to a mutable context

**Outcome:** New fields are made available to all calls for the remaining lifetime of the process.

When this pattern is used, additions to the context are made available to all downstream and upstream logs.

```python
import structlog

logger = structlog.stdlib.get_logger().new()
# Adding a field through "bind" makes a copy of the context on `logger`
# then adds the new field to the copy.
log = logger.bind(service_name="user_store")
# This logger has the "service_name" field.
log.info("starting service")

# Process user data
data = {"user_name": "Joe"}
user = process_user_data(log, data)
# Upstream log includes "service_name" and "user_name".
log.info("completed processing")

def process_user_data(log: BoundLogger, data: dict[str, Any]) -> User:
    user = User(**data)
    # Add the user_name field to shared context.
    structlog.contextvars.bind_contextvars(user_name=user.name)
    # Downstream log includes "service_name" and "user_name
    log.info("processing user data")
    return
```

Output

```python
2024-04-01T19:42:21.881139Z [Info    ] starting service                service_name="user_store"
2024-04-01T19:42:21.911139Z [Info    ] processing user data            service_name="user_store" user_name="Joe"
2024-04-01T19:42:22.001139Z [Info    ] completed processing            service_name="user_store" user_name="Joe"
```

Use middleware

It's common for logging libraries to send logs through a chain of processing methods after the log call is executed. These processors are responsible for transforming the log context and message into a well formatted output. This processing step is a great place to automate the addition of common metadata fields to every log. Nearly every logging config you'll see in a production code base will include the addition of log `level` and `timestamp`.

In all of the output examples above both `level` and `timestamp` were included through a processor. We'll get into the processor chain and writing custom processors later in the series. For now know the pattern exists and is the suggested option for adding default fields to every log line.

## **Use descriptive names**

Logger contexts have a large scope. They may even span multiple libraries. The chance of collision between keys with simple names is high in complex ecosystems. In many libraries key collisions are handled by silently overwriting the previous key value making it difficult to know they occur. When keys collide context is lost as a context value is dropped and log trails get interrupted.

Anti-pattern example:

```python
"""
session.py
"""
from uuid import uuid4

from structlog.contextvars import bind_contextvars

from db.user import get_user_by_name
logger = structlog.stdlib.get_logger().new()

# Initialize session
session_id = str(uuid4())
structlog.contextvars.bind_contextvars(id=session_id) 
logger.info("Session initialized")

# Get user
user = get_user_by_name("james")
logger.info("Fetched user")

"""
db/user.py
"""

def get_user_by_name(name: str) -> User:
    # Some method to get a user
    user = db_client.get_user(name=name)
    structlog.contextvars.bind_contextvars(id=user.id)
```

Output

```python
2025-03-01T19:42:21.881139Z [Info    ] Session initialized        id=0e7467a0-83d5-4e9c-ab7c-c91702796242
2025-03-01T19:42:22.001139Z [Info    ] Session initialized        id=084a8e6a-2bc7-4661-a06f-06aa15747f56
```

### How to

Be descriptive

As a best practice make key names as descriptive as possible. Using the example above `session_id` and `user_id` would make better key names. In small systems descriptive names alone may be sufficient to avoid collisions. 

Prefix keys with a package identifier

In larger ecosystems it's useful to establish a pattern of prefixing keys with a service or package identifier on top of using descriptive key names. Descriptive key names prevent conflicts within the service while the identifier guarantees global uniqueness.

It's common either add prefixes directly to the key constant name or to automate the addition through middleware.

Example

```python
# service 1
# Initialize session
session_id = str(uuid4())
structlog.contextvars.bind_contextvars(**{"service_1.session_id": session_id}) 
logger.info("Session initialized")
```

```python
# service 2
# Initialize session
session_id = str(uuid4())
structlog.contextvars.bind_contextvars(**{"service_2.session_id": session_id}) 
logger.info("Session initialized")
```

## Use primitive values for search fields

Most logging libraries support the inclusion of complex objects in structured fields (e.g. classes, structs, lists). This functionality is very useful for attaching relevant context to a log. However it's generally hard to reason about how these object will be indexed, making them hard to search for. Strive to encode any field that's relevant to search as a primitive.

This point is especially important when considering variables which appear to hold simple primitive types but in fact hold a wrapper object. A great example of this is the python `bson.ObjectId` which holds a `id` that can be represented easily as a string but without an explicit cast is serialized as `ObjectId(<id>)`.

### How to

Test your logs during development

- Run your code and inspect important logs. Make sure structured fields appear as you expect them to.
- Add tests verifying log emission and context. We'll cover log testing strategies in a future article
