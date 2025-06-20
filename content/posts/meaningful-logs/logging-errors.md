+++
date = '2025-06-20T14:46:17-07:00'
draft = false
title = 'Logging Errors'
description = "What it takes to log errors well"
series = ["Meaningful Logs"]
tags = ["logging", "development"]
weight = 4
+++
_Part 4 of 5 in the series [Meaningful Logs]({{< ref "/series/meaningful-logs" >}})_

Error logging is truly hard to get right. Even with the right tools and best practices it still takes conscious effort to construct meaningful errors. 

In this post, I'll dig into why error logging is so difficult and what it takes to do it well. In the next article, I'll walk through how to log errors in Python using a custom exception library.

# What makes error logging difficult

So far in this series, we've focused on designing logs that describe events and capture state. The goal has been to enable introspection through rich context representative of a specific point in time.

Logging errors presents a complex extension to the task of logging. In addition to everything we expect from a standard log, error logs must also: 

- Accommodate program `Exceptions` , which both provide context and interrupt control flow.
- Integrate with Error Aggregation systems
- Support debugging

Balancing all of these requirements effectively is where many systems fall short. The result is a system held back by noise and errors that lack the context needed to act.

# What to expect

A robust error logging solution must do exactly what we laid out above. It must:

- Accommodate `Exceptions` gracefully
- Integrate with error and log aggregation systems
- Support debugging

## Accommodate Exceptions

Most error logs describe exceptions. Exceptions serve two distinct purposes:

1. **Interrupt control flow**: Exceptions are raised whenever something goes wrong and the program can't (or shouldn't) continue. They propagate up the call stack, giving each layer a chance to handle the failure.
2. **Provide context**: Exceptions describe an issue. This includes both a stack trace and ideally structured information about the state of the program at the time of failure.

A good exception logging strategy provides space for both roles. It should integrate cleanly with the language's standard error-handling flow, while offering a mechanism to attach the kind of rich context we've described throughout this series.

To preserve clean control flow and avoid noise, exceptions are generally logged only at the highest layers of a program. As such, it's especially important to attach context to an exception when it's raised, since relevant information is often only available at this lower layer. A strong system must provide a way for context to travel alongside exceptions as they propagate.

### Example

To highlight the requirements for handling exceptions, consider the following example where an exception is raised at a low-level of a program:

- An important parameter `user_id` is known only within the method that raises the exception.
- The exception is logged at a higher layer with limited native access to context

To preserve context, the custom `UserFetchError` includes a structured details field. This allows `user_id` to be attached to the exception and included in the log emitted  when the exception is caught. 

Note that access to the `details` field is limited to only the dedicated exception block for the `UserFetchError`. The generic exception block has no access to additional context. We'll discuss a generic solution to this problem in the next post.

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

**Output**

Result of calling `handle_request("req-456")`

```python
{
  "event": "UserFetchError",
  "error": "Failed to fetch user",
  "user_id": "abc123",
  "level": "error",
  "timestamp": "..."
}
```

## Integrate with error aggregators

The first post in this series introduced the idea of logging for aggregation systems. There, we emphasized the importance of creating a searchable, centralized index for on demand log retrieval.

Error logs share these goals, but introduce new challenges. In addition to discovery and traceability, we also care about **volume**, **trends**, and **grouping**. While the ability to look up individual errors and dive into stack traces is still essential, it's equally important to know how often an error is occurring.

This is where **error aggregators** come in. These tools (e.g. [Sentry](https://sentry.io/)) are purpose-built for understanding and managing errors at scale. They support two critical workflows:

1. **Discovery**: Related error events are intelligently grouped to produce meaningful volume metrics. Events are indexed by all structured fields, enabling search and aggregation by tags like `user_id`, `service`, or `environment`. Most tools integrate directly with alerting platforms so that critical issues surface quickly. It's common to configure integrations to alerting systems based on these aggregated results.
2. **Root Cause Analysis**: Aggregators provide a detailed, point-in-time snapshot of individual error events—including stack traces, request metadata, environment variables, and all attached structured context.

### The Core Challenge: Grouping

Effective integration with an error aggregator comes down to **grouping.** Similar events must be recognized as part of the same issue to take advantage of aggregation and limit noise.

Most error aggregators rely on a concept called a `fingerprint` to determine which events should be grouped together. For more details, see [Sentry's guide to event grouping](https://docs.sentry.io/concepts/data-management/event-grouping/). The easiest and most reliable way to influence the `fingerprint` is to raise **custom exception classes with clear, consistent names**.

Some suggestions for grouping effectively:

- Use **custom exception classes** with stable, descriptive names.
    - I almost never raise standard library exceptions directly.
    - I'm liberal about introducing new error classes—typically one per distinct failure mode.
- Avoid embedding dynamic values in exception messages (e.g. "UserFetchError for user abc123"). These messages are difficult to group meaningfully.
- Attach all relevant context as **structured fields** instead of embedding them in the error message.

#### Example

```python
class UserFetchError(Exception):
    def __init__(self, user_id: str):
        super().__init__("UserFetchError")
        self.details = {"user_id": user_id}
```

This allows Sentry (or any other aggregator) to group all `UserFetchError` exceptions together providing easily interpretable volume metrics.

### Outcome

A well-integrated error aggregator ensures that if something does break you'll know and have the tools to respond quickly.

## Support debugging

Errors tell us something went wrong. But for that message to be useful, it must help someone understand what happened and how to fix it.

A well-constructed error makes this easy. A vague or generic one leaves you lost within the noise.

Good errors are:

- **Clear**: They describe exactly what failed.
- **Actionable**: They point to the failing code path or decision.
- **Context-rich**: They carry structured fields that explain *why* the error occurred—e.g. IDs, input values, or system state.

As mentioned throughout this series, I strongly recommend using **dedicated error classes** to represent specific failure types. These errors are easier to interpret, easier to search for, and far more effective when integrated with an aggregator.

#### Example:

```python
class UserNotFoundError(Exception):
    def __init__(self, user_id: str):
        super().__init__("UserNotFoundError")
        self.user_id = user_id
        self.details = {"user_id": user_id}

def fetch_user(user_id: str):
    user = db.get(user_id)
    if user is None:
        raise UserNotFoundError(user_id)
```

Compare this to raising a generic exception like a `KeyError`. On its own, it tells you very little. With enough stack trace and surrounding context, you might be able to piece things together - but tools like Sentry will struggle to group and index these reliably.

```python
def fetch_user(user_id: str):
    user = db.get(user_id)
    if user is None:
        raise KeyError(f"User {user_id} not found")
```

Be deliberate when designing debuggable errors. Every time you raise an exception, ask: *Will this be easy to understand later?* With consistent use of clear names and structured context, debugging becomes a traceable, repeatable process.

# Best practices

We've already covered a lot of ground in this article. For convenience, I've laid out a summary of best practices below. Many of these will probably feel natural to you by now after following the principles throughout this series.

### Summary

- [Log **exceptions** at the highest level]({{<ref "#log-exceptions-at-the-highest-level">}}).
- [Log errors once]({{<ref "#log-errors-once">}})—if you raise an exception, don't also log it mid-stack.
- [Use predefined names for error types]({{<ref "#use-predefined-names-for-error-types">}}).
- [Keep dynamic values out of error *fingerprints*]({{<ref "#keep-dynamic-values-out-of-error-fingerprints">}}). Attach them as structured fields instead.
- [Send errors to both a log aggregator and an error aggregator]({{<ref "#send-errors-to-both-a-log-aggregator-and-an-error-aggregator">}}).
- [Don't send errors to aggregators from local systems]({{<ref "#dont-send-errors-to-aggregators-from-local-systems">}}).
- [Use appropriate log levels]({{<ref "#make-use-of-different-log-levels">}}) (e.g., error vs warning) to guide attention.

### Log exceptions at the highest level

Plan to log the majority of exceptions at the boundary of your application. Let exceptions bubble up naturally, and only log them once they reach a top-level handler.

Avoid logging exceptions that you plan to re-raise. This ensures clean control flow and prevents duplication in your logs and aggregators.

If your code is intended to be embedded or reused (rather than run as a standalone app), be especially mindful about logging. In many cases, it's better to raise and let the caller decide how to handle it.

### Log errors once

Repeated logging of a single error event interferes with aggregation and inflates volume metrics. Duplicate logging is unfortunately very common. Logging exceptions at app boundaries is the best way to avoid this trap.

### **Use predefined names for error types**

Use stable, descriptive exception class names that clearly indicate the nature of the problem. These names are essential for effective grouping and deduplication in error aggregators.

You can reuse error types across similar failure conditions, but be intentional. Each class should represent a distinct issue.

### Keep dynamic values out of error *fingerprints*

Avoid embedding dynamic values (e.g., IDs, paths, or inputs) into the error message itself. These break fingerprinting and can scatter identical issues across many groups in your aggregator.

Instead, attach dynamic values as structured fields, so they remain queryable without interfering with grouping.

### Send errors to both a log aggregator and an error aggregator

Error aggregators like Sentry make errors actionable: they show trends, alert your team, and surface contextual details. But errors are still logs. They need to be present in your log aggregator so that full traces and workflows can be reconstructed. 

Make sure your system sends errors to both.

### Don't send errors to aggregators from local systems

Errors raised in local development or Jupyter notebooks can pollute your error aggregator with noise. These environments often trigger edge cases or incomplete setups that aren't production-relevant. By default, prevent error reporting from local environments. 

### Make use of different log levels

Log levels help indicate the severity of an issue and guide how logs are filtered, prioritized, and escalated. Use them intentionally to convey the nature of the problem.

As a general rule of thumb:

- **Error**: Use for breaking issues that indicate a failed state or task. These are actionable problems that typically require investigation.
- **Warning**: Use for non-breaking issues that the system can recover from or tolerate. Warnings are still useful to track—especially if you want to monitor their frequency or catch early signs of degradation.

Consistent use of log levels improves both local debugging and integration with alerting systems.

→ *Join me in the [final post]({{< ref "logging-errors-in-python">}}) for a discussion of how to log errors effectively in Python*
