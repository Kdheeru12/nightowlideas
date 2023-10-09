

---

# Instrumenting SQL Queries in Django: A Deep Dive

This guide explores how to instrument SQL queries within Django, using the built-in ORM capabilities to facilitate query logging, debugging, performance monitoring, and more.

## Table of Contents
1. [Understanding BaseDatabaseWrapper](#understanding-basedatabasewrapper)
2. [How to Monitor SQL Queries at Runtime?](#how-to-monitor-sql-queries-at-runtime)
3. [Execute Wrappers: Control Layer of SQL Execution](#execute-wrappers)
4. [Enter CursorWrapper: The Execution Orchestrator](#enter-cursorwrapper)
5. [Logging Configuration](#logging-configuration)
6. [Examples](#examples)

## Understanding BaseDatabaseWrapper

Django's `BaseDatabaseWrapper` class represents an individual database connection, serving as an interface between your application and the underlying database engine. This class has various methods and attributes to facilitate SQL execution and manage the connection.

**Location**: `db.backends.base`

## How to Monitor SQL Queries at Runtime?

To monitor and manipulate SQL queries during execution, the `execute_wrappers` attribute is used within the `BaseDatabaseWrapper`. This attribute is a stack of functions that wrap calls to `execute()` and `executemany()` in `CursorWrapper`.

### Execute Wrappers: Control Layer of SQL Execution

`execute_wrappers` functions act as middleware, intervening in SQL query execution. They accept five parameters: `execute`, `sql`, `params`, `many`, and `context`, allowing inspection or modification of the SQL statement and parameters.

Developers can utilize the `execute_wrapper` function to append custom wrappers for advanced capabilities such as logging and performance monitoring.

Execute wrappers are a subpart of the backend[mysql, psogres] connection class `class DatabaseWrapper(BaseDatabaseWrapper):` from where one can create CursorWrapper. 

### Enter CursorWrapper: The Execution Orchestrator

The `CursorWrapper` class manages query execution, invoking functions in `execute_wrappers` sequentially, with the required parameters. This class ensures adherence to specific protocols and logic.

## Logging Configuration

In your local environment, add the following to your logger configs in `settings.py`:

```python
"django.db": {
    "handlers": ["console"],
    "level": "DEBUG",
}
```

### Logging when `debug` is equal to `False`

Utilize the following code to log SQL queries:

```python
from django.db import connection, connections
import logging

def logsql(execute, sql, params, many, context):
    start = time.monotonic()
    try:
        return execute(sql, params, many, context)
    finally:
        duration = time.monotonic() - start
        logging.debug(
            "(%.3f) %s; args=%s",
            duration,
            sql,
            params,
            extra={"duration": duration, "sql": sql, "params": params},
        )
```

## Examples

### Instrumenting a particular View

```python
def my_view(request):
    context = {...}
    template_name = ...
    with connection.execute_wrapper(logsql):
        pass
```

### Instrumenting every query in the Django application using middleware

```python
class DatabaseInstrumentationMiddleware:

    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        with connection.execute_wrapper(DatabaseLogger()):
            return self.get_response(request)

class DatabaseLogger:
    def __call__(self, execute, sql, params, many, context):
        start = time.monotonic()
        try:
            return execute(sql, params, many, context)
        finally:
            duration = time.monotonic() - start
            logging.debug(
                "(%.3f) %s; args=%s",
                duration,
                sql,
                params,
                extra={"duration": duration, "sql": sql, "params": params},
            )
```

### Note

For multi-DB setups or read replicas, use `connections["name"]` to instrument a particular database connection. By default, only the default database is instrumented.

---
For more Ref : https://docs.djangoproject.com/en/4.2/topics/db/instrumentation/
