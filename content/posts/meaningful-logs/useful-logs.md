+++
date = '2025-06-20T14:41:45-07:00'
draft = false
title = 'Useful Logs'
series = ["Meaningful Logs"]
tags = ["logging", "development"]
weight = 1
+++
_Part 1 of 5 in the series [Meaningful Logs]({{< ref "/series/meaningful-logs" >}})_

From printing `"hello world"` to debugging your first program, nearly everyone starts their coding journey with a log statement.

In my experience, developers have an intuitive understanding of how powerful logs are as an introspection tool. What's not as obvious is that the simple form of logging that works so well during development loses most its utility in productionized code. In production, logs are emitted from many processes over a large time period. Without a well thought out logging strategy you'll find it nearly impossible to rebuild the sequential logflow that printed out so naturally on a single machine. 

All too often I've seen production systems operating with development style logs. I've found this shortcoming tends to stem from an unawareness both of how to log appropriately and how much better things could be.

In this series I'll cover several aspects of production logging in detail with a deep dive into various custom utilities for logging in Python. But before we get there we first need to shift our mindset. To move away from logging for ourselves as developers on a single computer toward logging for who (or more likely *what*) is consuming our logs.

# Mindset shift

In almost all cases logs from productionized codebases are consumed by either:

- A log aggregation system
- A person who is a user of your product
- A developer contributing to the product

The mindset shift comes in acknowledging that to maximize their usefulness logs should be constructed in the format ***most useful to the main type of expected consumer***. 

Fortunately there's a style of logging which lends itself well to all three (spoiler: it's structured logging!). Let's take a closer look at each type of consumer.

## **Logging for aggregation systems (most common consumer)**

Log aggregators combine logs from a variety of sources into an easily searchable index. Aggregators are a fantastic tool for debugging issues and inspecting the trail of information produced by a process. Common examples include Splunk and AWS Cloudwatch.

### Use cases

Aggregators are great for:

- Searching for events that happened across different process at different times.
- Constructing a full trail of events across multiple services.
- Searching for specific events based on key words.

Any process which is not executed and monitored directly by a user (e.g. a CLI) will likely benefit from an aggregation system.

If you have ever owned an application which wrote directly to a log file on a machine rather than an aggregator you know the pain of manually combing through logs to do a root cause analysis. With a log aggregator there's no need to connect directly to a host machine or to manually search across files. All logs are centralized in an easy to use application.

### How to log for aggregation systems

These systems are most useful when the ingested logs are ***structured*** meaning they contain key-value pairs which can be used to index messages. It's also generally useful to include ***metadata*** (e.g. `environment`, `session_id`) within logs to facilitate searches and the construction of dashboards based on log content.

The {{< ref "structured-log-primer" >}} will cover structured logging in detail.

### Useful practices

**Canonical log lines.**

It's worth mentioning the practice of canonical log lines here as they're especially useful in aggregation systems. The idea of a canonical log line is to capture all information relevant to a complete process flow in a single log. This cuts down on the time and effort required to construct a log trail. For details see this article: [Using Canonical Log Lines for Online Visibility](https://brandur.org/canonical-log-lines).

## **Logging for people**

When building tools used directly by humans the ***emphasis of logs should be on readable, actionable information***. That being said readable and actionable does not necessarily mean a log needs to be a cohesive sentence. Rather it just needs to be in a form that's easily understood.

### Use cases

- CLI
- Utility used with Jupyter notebooks

*Note: This section is geared toward human users of code not wrapped in a application. Error logs displayed on in application are a different story.*

### How to log for people

In many cases ***structured*** logs can be formatted in a way that is still human readable and generally more actionable than a sentence. For example this structured log is easier to interpret and reference as needed (for example to find `workflow_id`) than a sentence with the same information.

```jsx
{
  "event": "Calculation succeeded",
  "workflow_id": "64aef9ca51b691ce2d32a6cc",
  "task_id": "64aefa0455e6c5d7a1263d5e",
}
```

vs

```bash
"Calculation succeeded for workflow with id 64aef9ca51b691ce2d32a6cc in task 64aefa0455e6c5d7a1263d5e"
```

## **Logging for developers**

It's important that developers get meaningful, readable outputs while running and testing code. While developing locally more information tends to be better, with id's and stack traces playing a critical role in debugging. Along the same lines as human consumers developers will come to appreciate the consistency of structured logs. Each log message will contain all relevant identifiers and the ids are easy to parse visually.

### How to log for developers

While structured information may be useful, the raw format most useful to aggregation systems isn't always easy to interpret. Generally log aggregators are sent information in the form of a raw json string. This can be read but it's not a great experience.

Fortunately there's a ton of tooling for converting structured logs into messages which are easier for a human to read. When possible use a separate logging config during development which pretty prints outputs. There are configurable tools and utilities for styling console outputs in every language.

# **Logging Errors**

Error logging comes with a whole suite of extra challenges. At its core logging errors is similar to info logging. The purpose of the log is to provide an indication that an event took place and context on the state when the event occurred. 

### How to log errors

The structured logging pattern is an ideal method for building the context associated with the event. However in most cases error logs are the log form of an exception raised during a program. Exceptions serve a dual purpose of providing context on an unexpected state and exiting the control flow of a program. Logging errors appropriately requires accommodating both purposes while maintaining the ability to include structured information describing the environment at the time of the exception.

Errors will be covered in [a later article]({{< ref "logging-errors" >}}) where I'll motivate and introduce a custom exception library.

## Onward!

With the motivation out of the way, join me in the [next article]({{< ref "structured-log-primer" >}}) where I dig deeper into best practices and use cases for structured logs
