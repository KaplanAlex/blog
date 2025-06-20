+++
date = '2025-06-20T14:46:03-07:00'
draft = false
title = 'Structured Logging in Python'
description = 'Kicking off our Journey into Python structured logs'
series = ["Meaningful Logs"]
tags = ["logging", "development"]
weight = 3
+++
_Part 3 of 5 in the series [Meaningful Logs]({{< ref "/series/meaningful-logs" >}})_

Now that we're fully motivated to structure all of our logs we can dive into the specifics of structured logging through examples. 

The remainder of this series will focus on structured logging in Python. We'll use packages specific to Python and write several custom Python utilities. In many ways this series will serve as a comprehensive guide for structured logging specifically in Python. That being said, the ideas and practices covered are generic. The following articles will still be useful to anyone hoping to get a better grasp on structured logging.

# Open Source Library: `structlog`

All popular languages have at least one well maintained library for structured logging. I highly recommend building your logging infrastructure on top of one of these libraries. You'll be able to take advantage of the active development from a large talented community.

In Python my library of choice is `structlog` [[Github](https://github.com/hynek/structlog/)] [[Docs](https://www.structlog.org/en/stable/)]

`structlog`provides two main functions. The ability to:

1. Add structured information to logs in the form of key value pairs.
2. Apply a set of processors to logs. Examples include:
    1. Transformations: removing sensitive fields
    2. Enrichments: Adding date time, and filters

The `structlog` [documentation](https://www.structlog.org/en/stable/) is awesome and likely has the answers to all your questions. This article will distill highlights from the documentation and my own experience using `structlog`.

# `structlog` concepts

- [BoundLogger]({{<ref "#boundlogger">}})
- [Context]({{<ref "#context">}})
- [Processors]({{<ref "#processors">}})

### BoundLogger

https://www.structlog.org/en/stable/bound-loggers.html

The core building block in struct log is a "bound logger". This is the logger object that's used to run log commands (`log.info(...)`) and to attach structured information to messages (`log.bind(…)`). `structlog` always wraps a logger and provides access to the logger through a `BoundLogger` instance.

A`BoundLogger` ***binds*** key/value context pairs to a log object. 

The logger holds:

- Context: A dictionary of the key value pairs to be included in the output.
- Processors: The transformations to apply before output.
- A wrapped logger: Which actually logs the output. Commonly a logger from the `structlog` library (e.g. `PrintLoggger`).

### Context

The act of adding a structured field to a logger is known as "binding". Each bind action returns a new, immutable bound logger with an updated `context` commonly referred to as the `event_dict`. The `context` holds the key value pairs to associate with log lines.

`context` values can be added with `bind` and removed with `unbind`. `bind` can also be called with no arguments to initialize a new, isolated logger instance.

### Processors

https://www.structlog.org/en/stable/processors.html

Processors are methods which transform the contents of `context`. Processors run sequentially in a chain with each method receiving the `context` constructed by the last step. Common examples of `processors` include adding log level, timestamps, and filters.

The `structlog` library has many pre-built processors which can be easily dropped into your processor chain. I always include the following in my set of processors:

```python
processors = [
    # Combines variables defined in the execution local global
    # context with variables bound to a specific logger.
    structlog.contextvars.merge_contextvars,
    # Adds a variable indicating the log level.
    structlog.processors.add_log_level,
    # Adds the stack trace to exceptions
    structlog.processors.StackInfoRenderer(),
    # Adds a timestamp
    structlog.processors.TimeStamper(fmt="iso"),
]
```

Custom processors are easy to write and add to the chain as well. We'll get to a real example of this in the article on Error Logging. As a simple example, imagine you wanted to ensure that all context keys were lower case. Doing so is as simple as defining a method that takes in and outputs the event dictionary.

```python
def lower_event_keys(_logger: WrappedLogger, _method_name: str, event_dict: EventDict) -> EventDict:
    """
    Structlog process to convert all event keys to lowercase.

    Args:
        _logger (WrappedLogger): Unused.
        _method_name (str): Unused.
        event_dict (EventDict): structlog EventDict containing the structured data to be logged.

    Returns:
        EventDict: The modified event dictionary.
    """
    new_event_dict = {}
    for key in event_dict:
        new_event_dict[key.lower()] = event_dict[key]
    return new_event_dict
```

Simply add this method to the appropriate place in the list of processors and you're good to go!

# Logging in practice

With that background out of the way, let's dive into a few conventions I've found useful when logging. They are:

1. [Use a dedicated class to manage log configuration]({{<ref "#use-a-dedicated-log-configuration-class">}})
2. [Use a combination of logger access methods as appropriate]({{<ref "#use-a-combination-of-logger-access-methods">}})
3. [Build your log context]({{<ref "#build-your-log-context">}})

The following sections cover these conventions in detail, grounding the conversation in tangible examples.

## Use a dedicated log configuration class

Logging configurations tend to be complicated. Fortunately they are not changed often and are generally consistent across an ecosystem. I've found one log config class to be enough to support dozens of packages with diverse usage patterns.

In designing a log configuration class look to support:

- Customization. Especially across environments.
- Centralization of all core configuration logic.
- A simple interface for a consumer to configure a logger. Ideally in one line.

### Customization

Configuration classes have two separate sets of responsibilities:

1. Accepting input parameters
2. Exposing the final configuration state

In designing for flexibility I've found it valuable to **separate the fields that serve each purpose**. That is, maintain one set of fields for raw input parameters and a completely separate set of properties exposing the final configuration state.  The key benefit here is that the final config state can be calculated from **any combination of inputs and custom logic.** The example config below demonstrates this pattern—highlighting the utility of derived properties like `log_format`, which are calculated based on multiple input values.

My preferred approach for supplying input parameters is through environment variables. This offers sufficient flexibility for a single config to be used across different software packages and environments (our main goal).

For exposing the final configuration state, I rely on **immutable, clearly named fields.** Pydantic's `@computed_field` decorator [[see docs here](https://docs.pydantic.dev/2.0/usage/computed_fields/)] is a great tool for defining this type of property.

```python
from pydantic_settings import BaseSettings

class LogConfig(BaseSettings):
    """
    Logging configuration
    """

    # Pydantic configuration to enable providing the input values
    model_config = {"populate_by_name": True, "extra": "allow"}
    ###############################
    # Configuration parameters
    ###############################
    # NOTE that "env" is actually both an input and parameter and final
    # configuration property. Because it is both required and simple, there is
    # little benefit to adding a dedicated property field.
    env: ENV_VALUES = Field(default=ENV_VALUES.LOCAL, validation_alias="ENV")
    set_log_format: Optional[LOG_OUTPUT_FORMAT] = Field(default=None, validation_alias="LOG_FORMAT")
    set_log_level: Optional[LOG_LEVEL_STR] = Field(default=None, validation_alias="LOG_LEVEL")
    set_human_user: Optional[bool] = Field(default=None, validation_alias="CONFIGURE_FOR_HUMAN_USER")

    ###############################
    # Derived configuration state
    ###############################
    @computed_field
    def human_user(self) -> bool:
        """Whether the main consumer is a human"""
        if self.set_human_user is not None:
            return self.set_human_user
        return in_notebook() or sys.stderr.isatty()

    @computed_field
    def log_format(self) -> LOG_OUTPUT_FORMAT:
        """The log output format to use."""
        if self.set_log_format is not None:
            return self.set_log_format
        return LOG_OUTPUT_FORMAT.PRETTY if self.human_user else LOG_OUTPUT_FORMAT.JSON

    @computed_field
    def log_level(self) -> LOG_LEVEL_STR:
        """The log level to use."""
        if self.set_log_level is not None:
            return self.set_log_level
        return LOG_LEVEL_STR.DEBUG if self.human_user else LOG_LEVEL_STR.INFO

    @computed_field
    def stdlib_log_level(self) -> int:
        """The log level in stdlib format."""
        return as_logging_level(self.log_level)
```

*Python specifics*

This configuration class leverages the Pydantic BaseSettings class which offers built in support for setting instance variables from environment state. The extra config at the top allows configuration parameters to be defined either based on the field name (e.g. `set_log_format`) or the `validation_alias` (e.g. `LOG_FORMAT`).

### Centralization

**Log configuration classes** are a great container for the initialization logic that's frequently required when setting up a logger, especially since this setup is typically  customized based on the current configuration state.

With `structlog` this customization mainly involves defining the `processor` chain and `logger` class. The section below extends the `LogConfig` class to include a method for retrieving a fully configured `structlog` logger instance.

```python
import structlog

from pydantic_settings import BaseSettings

class LogConfig(BaseSettings):
		"""
    Logging configuration
    """
    # ... input parameters (see above)
    # ... derived state fields (see above)
    
    def _build_structlog_processors(self) -> list[Processor]:
        """
        Return the set of processors to include in a structlog configuration.
        """
        # Base configuration
        processors = [
            # Combines variables defined in the execution local global
            # context with variables bound to a specific logger.
            structlog.contextvars.merge_contextvars,
            # Adds a variable indicating the log level.
            structlog.processors.add_log_level,
            # Custom processor to convert keys to lower case (see example above).
            lower_event_keys,
            # Adds the stack trace to exceptions
            structlog.processors.StackInfoRenderer(),
            # Adds a timestamp
            structlog.processors.TimeStamper(fmt="iso"),
        ]

        # Render
        if self.log_output_format == LOG_OUTPUT_FORMAT.PRETTY:
            colors = self.force_no_color_logs is None
            # Render normal log lines.
            # pretty print in terminal sessions when "rich" is installed.
            processors += [
                structlog.dev.set_exc_info,
                structlog.dev.ConsoleRenderer(colors=colors),
            ]
        else:
            # JSON logs with structured tracebacks.
            processors += [
                structlog.processors.dict_tracebacks,
                structlog.processors.JSONRenderer(),
            ]

        return processors

    def get_logger(self) -> structlog.stdlib.BoundLogger:
        """
        Fetch a new logger configured based on the current class state.
        """
        structlog.configure(
            processors=self._build_structlog_processors(),
            wrapper_class=structlog.make_filtering_bound_logger(self.stdlib_log_level),
            context_class=dict,
            # This is the logger that ultimately renders the output.
            # Many types of loggers are supported including the stdlib logger.
            # PrintLogger is a simple logger which just writes to stdout.
            logger_factory=structlog.PrintLoggerFactory(),
            cache_logger_on_first_use=False,
        )
        # Use structlog.stdlib.get_logger used type hinting
				return structlog.stdlib.get_logger(package).bind()
```

### Simple interface

Flexible support for input parameters and centralized configuration logic come together to yield an intuitive user interface. In the example `LogConfig` class it's easy to provide configuration parameters either directly or through environment variables.

Example notebook usage:

```python
#%%
# Set envirionment variables before importing the log config package
import os
os.environ["LOG_OUTPUT_FORMAT"] = "PRETTY"

#%%
# Create config
from my_log_lib.log_config import LogConfig

# Set a configuration parameter directly
config = LogConfig(set_log_level="INFO")

# %%
print(config.log_level) # Output: "INFO"
print(config.log_format) # Output: "PRETTY"

# %%
# Fetch logger
logger = config.get_logger()
```

## Use a combination of logger access methods

Now that your logger is initialized, the next challenge is making it accessible everywhere you need to log. There are two logger access patterns I rely on frequently:

1. **Injection**
2. **Module-wide access (singleton)**

### Injection

Dependency injection is one of my favorite software engineering principles. It's a simple concept that limits the scope of software, making it easier to reason about. If you're unfamiliar, *injection* just means passing dependencies as explicit inputs rather than having components reach out to the global context.

#### Quick Injection example: passing a database client

Suppose you have a class that manages the execution of a job. Rather than initializing a database client inside the class you pass it in:

```python
class JobRunner:
    def __init__(self, db_client: DBClient):
        self._db_client = db_client

    def run(self):
        self._db_client.select_lots_of_data()
```

Passing in the client makes the class easier to test and reuse. The `JobRunner` doesn't need to know how to configure a `DBClient`, only how to use it. In tests you can swap in a mock then focus on testing the core logic of the `JobRunner` class.

```python
def test_runner_db_call():
    mock_db_client = MagicMock()
    runner = JobRunner(db_client=mock_db_client)
    runner.run()
    mock_db_client.select_lots_of_data.assert_called_once()
```

#### Injecting loggers

Loggers can be injected in the same way. The benefits are similar:

- The logger arrives fully configure with relevant upstream context attached
- Testing is simple

```python
class JobRunner:
    def __init__(self, log: structlog.stdlib.BoundLogger, job_id: str, db_client: DBClient):
        # Bind a job-specific context to the logger
        self._log = log.bind(job_id=job_id)
        self._db_client = db_client

    def execute(self):
		    start_time = datetime.now()
        self._log.info("execution_started")
        self._db_client.select_lots_of_data()
        self._log.info("execution_finished", execution_time=datetime.now() - start_time)
```

```python
def test_runner_logs_job_flow():
    mock_logger = MagicMock()
    mock_db_client = MagicMock()

    runner = JobRunner(log=mock_logger, job_id="abc123", db_client=mock_db_client)
    runner.execute()

    mock_logger.info.assert_any_call("execution_started")
    mock_logger.info.assert_any_call("execution_finished")
    mock_db_client.select_lots_of_data.assert_called_once()
```

### Module-wide access

The second common pattern is to pull a logger from the global scope. In general, I caution against using global state -  it's hard to reason about and tricky to test. However when it comes to logging, **module-scoped loggers** are often a reasonable choice.

In Python in particular, the use of "module level loggers" is widespread. It's common to configure a logger once at package initialization in the top-level `__init__.py`  and import it wherever it's needed.

#### Example: package-wide logger

```python
# my_package/__init__.py
from my_package.log_config import LogConfig
import structlog

# Configure structlog
LogConfig().configure_logger()

# Create a base logger for the package
logger = structlog.stdlib.get_logger().bind(package="my-package")
```

Any module within `my_package` can now log without extra setup:

```python
# my_package/workflow.py
from my_package import logger

def match_name(raw: str):
    logger.debug("looking for name", raw=raw)
    # ...
```

Benefits of this approach:

- **Simplifies usage**: The logger is directly accessible everywhere in the module. Any code path is free to log.
- **Ensures consistency**: All log statements share the same processors, format, and base context (e.g., `package="my-package"`).
- **Works well with functional libraries like `structlog`**: `structlog` loggers are immutable, so binding new context in individual modules won't affect others. Each call to `.bind()` returns a new logger instance with the added context.

### When to use each pattern

Most packages will work fine with either approach. My advice is to:

- Inject loggers when:
    - Your program has a clear set of entry points.
    - You want to provide a base log context or configuration to a class.
- Use a global logger when passing loggers explicitly is too cumbersome.
    - This pattern is especially useful in utility packages where this is no clear entry point.

## Build your log context

There are many patterns for attaching context to logs. Combine these patterns in whatever way best enables you to track to information that matters most when you log.

### Pattern 1: Bind to a base logger

As shown throughout the series, context can be set directly on a logger. When working with a base logger—whether it's created during startup or injected into a class—it's a good idea to bind context that won't change throughout the logger's lifetime right away.

#### Example: Initialization

```python
# Application startup
logger = structlog.stdlib.get_logger().bind(app="user-api", env="prod")
```

**Example: Injection**

```python
class WorkflowRunner:
    def __init__(self, logger: structlog.stdlib.BoundLogger, workflow_id: str):
        self._log = logger.bind(workflow_id=workflow_id)

    def run(self):
        self._log.info("workflow_started")
```

### Pattern 2: Build log context as a dictionary

When in doubt, a dictionary can always be used as a container for log context. Manual maintenance of a dictionary gives you tight control over the content and scope of your log context. This pattern is especially useful when working with a global logger. 

```python
from my_package import logger

class BatchProcessor:
    def __init__(self, job_id: str):
        self._ctx = {"job_id": job_id}

    def run(self, batch_id: str):
        context = {**self._ctx, "batch_id": batch_id}
        logger.info("batch_started", **context)
        updated_ctx = self._process(context)
        logger.info("batch_finished", **updated_ctx)

    def _process(self, ctx: dict) -> dict:
        proc_data_id = self._do_some_work()
        return {**ctx, "proc_data_id": proc_data_id}
        
```

```python
from my_package import logger

# Instantiate the processor with a job ID
processor = BatchProcessor(job_id="job-123")

# Run the processor on a batch
processor.run(batch_id="batch-456")
```

**Output:**

```python
{"event": "batch_started", "job_id": "job-123", "batch_id": "batch-456"}
{"event": "batch_finished", "job_id": "job-123", "batch_id": "batch-456", "proc_data_id": "proc-789"}
```

### Pattern 3: Update the global, thread-local context

Some log libraries support storing context in a dedicated global memory space unique to each process. This functionality is incredibly powerful as it allows you to add context to log lines from anywhere within a process flow! `structlog` offers this functionality through the [context vars](https://www.structlog.org/en/stable/contextvars.html) module.

As an example, consider a scenario where you want to log a summary event at the end of a process that includes context fields gathered from the entire process. Aggregating this context across process layers poses a significant challenge.

#### Manual Dictionary Maintenance

One option is to leverage Pattern 2 and pass a dictionary with context around everywhere. This is effective but cumbersome.

```python

logger = structlog.get_logger()

def lookup_db_user(ctx: dict) -> tuple[str, dict]:
    """
    Call the database.
    
    Args:
         ctx (dict): Current process context.
         
     Returns:
         str: user id.
         dict: Updated process context.
    """
    # Do something ...
    user_id = "user-456"
    return user_id, {"user_id": user_id}

def process_user_req(ctx: dict) -> dict:
    """
    Process a request from a user.
    
    Args:
         ctx (dict): Current process context.
         
     Returns:
         dict: Updated process context.
    """
    user_id, updated_ctx = lookup_db_user(ctx)
    logger.info("completed user processing", **updated_ctx)
    return {**updated_ctx, "another_key": 909}

def handle_request(req_id: str):
    """
    Handle a request
    
    Args:
        req_id (str): Identifier for a request.
    """
    ctx = {"request_id": req_id}
    updated_ctx = process_user_req()
    logger.info("Process summary", **updated_context)
```

Output from calling `handle_request("req-123")`

```python
{"event": "completed user processing", "request_id": "req-123", "user_id": "user-456"}
{"event": "Process summary", "request_id": "req-123", "user_id": "user-456", "another_key": 909}
```

#### Using thread-local context

Leveraging the thread-local context removes the complexity of passing state around.

```python
from structlog.contextvars import bind_contextvars, clear_contextvars
import structlog

logger = structlog.get_logger()

def lookup_db_user() -> str:
    """
    Call the database to find a user.
    
    Returns:
        str: user id.
    """
    # Set context deep in the call stack
    user_id = "user-456"
    bind_contextvars(user_id=user_id)
    return user_id

def process_user_req():
    """
    Process a request from a user.
    """:
    lookup_db_user()
	  logger.info("completed user processing")
    bind_contextvars(another_key=909)

def handle_request(req_id: str):
    """
    Handle a request
    """
    bind_contextvars(request_id=req_id)
    process_user()
    logger.info("Process summary")
```

Same output from calling `handle_request("req-123")`

```python
{"event": "completed user processing", "request_id": "req-123", "user_id": "user-456"}
{"event": "finished request", "request_id": "req-123", "user_id": "user-456"}
```

### Pattern 4: Pass context along with errors

Context is especially valuable when logging exceptions. Lots of information is needed to effectively diagnose the root cause of an exception. Fortunately exceptions are a great vehicle for passing context!

In the next post, we'll cover how to attach context to errors, log them consistently, and propagate metadata across layers.

→ *Join me in the [next post]({{< ref "logging-errors">}}) to dive into error logging!*
