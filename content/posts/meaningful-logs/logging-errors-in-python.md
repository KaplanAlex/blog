+++
date = '2025-06-20T14:46:23-07:00'
draft = false
title = 'Logging Errors in Python'
series = ["Meaningful Logs"]
tags = ["logging", "development"]
weight = 5
+++
_Part 5 of 5 in the series [Meaningful Logs]({{< ref "/series/meaningful-logs" >}})_

In the [previous article]({{< ref "logging-errors" >}}), we explored the many challenges of logging errors—how to capture them effectively, group them meaningfully, and support fast debugging.

This article grounds that discussion in practical examples and utilities that can be used as the foundation for robust error logging in Python. We'll cover:

- A pattern for structured logging in Python that satisfies all three requirements.
- Direct integration with `structlog` and [Sentry](https://sentry.io/).
- Logging examples

# Custom Python Exceptions

The previous article outlined three core expectations for any robust error logging solution:

1. **Gracefully handle exceptions** – preserve control flow while capturing context
2. **Enable meaningful grouping** – support aggregation systems like Sentry
3. **Support debugging** – through structure, clarity, and consistent naming

In discussing these requirements we found that much of meeting them comes down to how we define and raise exceptions. Specifically we showed the custom exception classes provide substantial benefits including:

- **Structured context**: Exceptions are objects—making it easy to attach relevant fields
- **Natural grouping**: Tools like Sentry can group errors by class name
- **Improved traceability**: Custom names make errors easier to search, recognize, and debug

However as you may have noticed in some of the previous examples, there are still a few points of friction in exception logging. To solve this, we'll design a small abstraction that simplifies structured error logging and standardizes how context is captured and surfaced.

To begin, let's revisit the code from the [*Accommodate Exceptions*]({{<ref "logging-errors#accommodate-exceptions">}}) section of the last post. It successfully captures context (`user_id`) but doing so required manual plumbing:

```python
# my_package.__init__ configures a logger as described in the last section.
from my_package import logger

########################
# Custom Exception class
########################
class UserFetchError(Exception):
    def __init__(self, user_id: str, message: str = "Failed to fetch user"):
        super().__init__(message)
        self.user_id = user_id
        self.details = {"user_id": user_id}

############
# Server
############
def fetch_user(request_id: str) -> str:
    # Pretend there is a database call here to fetch a id
    # That call returns "abc123"
    user_id = my_db.fetch_session_user_info(request_id)
    
    # Simulate a failure
    raise UserFetchError(user_id=user_id)

def handle_request(request_id: str):
    try:
        fetch_user(request_id)
    # Log once at the application boundary.
    except UserFetchError as e:
        # Access structured context from the special error class.
        logger.exception(
            "UserFetchError",
            request_id=request_id,
            error=str(e),
            **e.details
        )
    except Exception as e:
        # No access to additional context in this block.
        logger.exception("Generic failure", error=str(e))

# Call the handler
handle_request("req-456")
```

While this pattern works for this use case, there is a strong coupling between the structure of this one-off exception and  logger expectations. This strategy will not scale well, especially as our system matures and we delve into more complex logging scenarios like exception chaining.

## Exception Base Class

To standardize this pattern and hide complexity during day-to-day development we can define a custom base exception class `CustomException`. This class will:

- Standardize how context is stored and accessed
- Enable automatic context propagation through exception chains
- Reduce friction when logging at the top level

```python
class CustomException(Exception):
    def __init__(self, message: str = "", ctx: Optional[dict] = None) -> None:
        """
        Base class for custom exceptions. Offers a standard interface for attaching
        context to exceptions.

        Args:
            message (str, optional): Additional context to associate with the error. Defaults to "".
            ctx (Optional[dict], optional): Metadata associated with the error. Defaults to None.
        """
        if ctx is None:
            ctx = {}
        self.ctx = ctx
        self.message = message
        if self.message != "":
            self.ctx.update({"message": self.message})
        super().__init__(self.message)

    def get_context_chain(self) -> dict:
        """
        Retrieves the exception context from the entire exception chain.
        """
        ctx = self.ctx.copy()
        if self.__cause__ and isinstance(self.__cause__, CustomException):
            for k, v in self.__cause__.get_context_chain().items():
                ctx[f"inner.{k}"] = v
        return ctx

```

This utility gives us a clean, composable way to attach and collect structured fields associated with any error. When paired with a logging wrapper, we can automatically extract this context and emit it with our log.

## Usage Guidelines

Use `CustomException` as the base class for any error you plan to send to aggregators or include in structured logs.

### Key Features

- **message** *(optional)*: A human-readable description of the specific failure
- **ctx** *(optional)*: A dictionary of metadata to attach to the log or error report
- **Chaining support**: When exceptions are chained, context from the full chain is preserved via get_context_chain()

### Best Practices

- **Subclass** `CustomException` for each meaningful error type
- Give each class a stable, descriptive name that reflects the root cause (e.g. `UserFetchError`, `InvalidRequestError`)
- Reuse these classes across your codebase for consistent grouping and easier triage
- Use `ctx` to attach all relevant metadata—e.g., `request_id`, `user_id`, environment state, etc.

### Example

```python
# Exception class
class UserFetchError(CustomException):
    pass

# Failing method
def fetch_user(user_id: str):
    # ...
    raise UserFetchError("Failed to fetch user", ctx={"user_id": user_id})

# Invoke
try:
    fetch_user("abc123")
except CustomException as e:
    logger.error("UserFetchError", **e.get_context_chain())
```

# Integrations

To make error logging seamless, we'll now integrate our custom error classes into two tools we've discussed extensively: `structlog` for structured logs and [Sentry](https://sentry.io/) for error aggregation.

## Structlog

We've talked a lot about the flexibility and power of `structlog`. Now we'll leverage the power of this library to eliminate the last bits of boilerplate. We'll  build a custom processor that automatically extracts metadata from our `CustomException` class.

```python
class LOG_KEYS(str, Enum):
    """
    Consistent names for recurring keys.
    """

    # Holds the relevant environment.
    ENV = "env"
    # Holds an exception.
    ERR = "err"

def process_custom_exceptions(_logger: WrappedLogger, _method_name: str, event_dict: EventDict) -> EventDict:
    """
    Processor to extract context from CustomException and attach it to the log.

    Requirements:
    - Error must appear under the "err" field in the event_dict.
    - Error must be a subclass of CustomException.
    """
    # Check if there's an error in the event dictionary.
    if LOG_KEYS.ERR not in event_dict:
        return event_dict

    err = event_dict[LOG_KEYS.ERR]
    if not isinstance(err, CustomException):
        return event_dict

    # Set the event field to the error class when omitted (e.g. an empty string).
    if not event_dict.get("event"):
        event_dict["event"] = err.__class__.__name__

    # Flatten the error context into structured fields, namespaced under "err." to
    # avoid conflicts with existing keys.
    for k, v in err.get_context_chain().items():
        event_dict[f"err.{k}"] = v
    return event_dict

```

This processor can be added to a `structlog` processor chain to automatically enrich logs whenever an exception is passed via the `err` keyword.

#### Example — Even Simpler

```python
# Exception class
class UserFetchError(CustomException):
    pass

# Failing method
def fetch_user(user_id: str):
    # ...
    raise UserFetchError("Failed to fetch user", ctx={"user_id": user_id})

# Invoke
try:
    fetch_user("abc123")
# Handle all exceptions the same way
except Exception as e:
    logger.error("", err=e)
```

No manual unpacking of error fields. No custom naming logic. Just raise your error with context and let the processor do the rest.

## Sentry

We can take our solution a step further by integrating Sentry directly into the processor chain. This allows us to automatically forward errors (along with structured context) to our error aggregator.

Following error logging best practices, we want:

- **Control over when Sentry is active** (e.g. don't send from local)
- **Rich structured context** on every event
- **Reliable grouping** using our custom exception classes

We'll manage all of this by extending our `LogConfig` class. 

### Update 1: Enable Sentry Conditionally

To begin we'll setup support for conditional Sentry enablement based on a dedicated environment variable `SENTRY_LOG_LEVEL` . This will ensure data is not sent accidentally from local development.

```python
from pydantic_settings import BaseSettings

class LogConfig(BaseSettings):
    ###############################
    # Configuration parameters
    ###############################
    env: ENV_VALUES = Field(default=ENV_VALUES.LOCAL, validation_alias="ENV")
    # NEW FIELD: parses the lowest error level to send to sentry from an environment variable.
    set_sentry_log_level: Optional[LOG_LEVEL_STR] = Field(default=None, validation_alias="SENTRY_LOG_LEVEL")

		# ...existing fields

    ###############################
    # Derived configuration state
    ###############################
    # ...existing fields
    @computed_field
    def sentry_enabled(self) -> bool:
        return self.sentry_level != SENTRY_LOG_LEVEL.DISABLED

    @computed_field
    def sentry_log_leve(self) -> Optional[LOG_LEVEL_STR]:
        """Minimum log level that can be sent to sentry"""
        return self.set_sentry_log_level

```

### Update 2: Add the SentryProcessor

Next we'll add the default `SentryProcessor` to the processor chain when Sentry is enabled. This processor captures and forwards structured logs, using the configured log level as a filter.

```python
from pydantic_settings import BaseSettings
from structlog_sentry import SentryProcessor

class LogConfig(BaseSettings):
    """
    Logging configuration
    """

    def _build_structlog_processors(self) -> list[Processor]:
        processors = [
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            process_custom_exceptions,
            # Other processors...
        ]

    # Forward logs of priority "event_level" or higher to sentry.
    # Must come after "add_log_level" and "process_custom_exceptions".
    # Adds all structured fields as sentry tags and event data.
        if self.sentry_enabled:
            processors.append(
                SentryProcessor(
                    event_level=as_logging_level(self.sentry_log_level),
                    tag_keys="__all__"  # Include all structured fields as Sentry tags
                )
            )

        return processors
```

### Update 3: Initialize the Sentry SDK.

Add the Sentry SDK initialization to the log configuration call to provide the library with details on where and how to send messages. Behavior is customized through the custom `before_send` hook.

```python
from pydantic_settings import BaseSettings
from structlog_sentry import SentryProcessor

class LogConfig(BaseSettings):
    """
    Logging configuration
    """

    def get_logger(self) -> structlog.stdlib.BoundLogger:
    """
    Fetch a new logger configured based on the current class state.
    """
    structlog.configure(
        # ...existing configuration
    )

    # UPDATE: Configure the sentry sdk when enabled. Provide your sentry project URL (DSN)
    # and the custom before_send method which modifies the fingerprint.
    if self.sentry_enabled:
        sentry_sdk.init(
            dsn=SENTRY_DSN,
            environment=config.env,
            before_send=before_send, # custom grouping logic
        )

    # Use structlog.stdlib.get_logger used type hinting
    return structlog.stdlib.get_logger(package).bind()
```

### Update 4: Implement `before_send`

We've been careful to use dedicated exception classes to describe meaningful failure types. This hook tells Sentry to group events based on the outermost exception's class name, rather than its message or traceback.

```python

SENTRY_HINT_EXEC_INFO = "exc_info"

def before_send(event: dict[str, any], hint: dict[str, any]) -> dict[str, any]:
    """
    Sets the Sentry fingerprint to the exception class name for custom grouping.
    """
    if SENTRY_HINT_EXEC_INFO not in hint:
        return event

    exception = hint[SENTRY_HINT_EXEC_INFO][1]
    if isinstance(exception, CustomException):
		    event["fingerprint"] = [exception.__class__.__name__]

    return event
```

# Wrapping up

Thank you for joining me in this series! Logging is a nuanced practice—one that I've become passionate about after seeing the transformative impact of getting it right.

Through these posts, we've explored:

- The different types of log consumers—and how to support each
- The value of structured logging and how to implement it effectively
- Best practices for both standard and error-specific logging
- And how to pull it all together in Python using `structlog` and Sentry

With the `CustomException` class and lightweight `structlog` integrations introduced in this final post, you now have a complete toolkit for clean, consistent, and effortless error logging.

Happy logging!
