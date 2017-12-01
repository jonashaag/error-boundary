# Python Error Boundaries

A Python library for inserting error boundaries in your code.

[![Build Status](https://travis-ci.org/cashlink/error-boundary.svg?branch=travis)](https://travis-ci.org/cashlink/error-boundary)

## What are Error Boundaries?

An error boundary prevents exceptions in a code block from crashing your program.

This is very useful of some parts of your code are "optional", but likely to fail. With error boundaries, you can prevent your code from crashing if an optional operation fails:

```python
important_stuff()
with ErrorBoundary():
    not_so_important_stuff()
```

If `not_so_important_stuff` raises an exception, the exception is logged and the program continues execution after the `with` block.

Examples of stuff that is "optional" for your program logic and likely to fail:

- Sending e-mail
- Calling external APIs
- Queueing backgrounds jobs
- Reporting performance metrics

## How to use this library

We'll use Django request examples here, but the library works with any Python code.

```python
from error_boundary import ErrorBoundary

def some_view(request):
    some_model = ...
    some_model.save()

    # Return HTTP 200 even if sending the mail fails.
    # This logs + suppresses all exceptions.
    with ErrorBoundary():
        django.core.mail.send_mail(...)

    return HttpResponse(...)
```

### Activating the boundary only in production

This is the recommended way to use error boundaries. In development you'll want to never "miss" an error. In production, failures in optional code should not make your program fail.

```python
from error_boundary import ProductionErrorBoundary

in_production = determine_if_were_in_production()
with ProductionErrorBoundary(in_production):
	django.core.mail.send_mail(...)
```

You can also re-use an error boundary object:

```python
in_production = determine_if_were_in_production()
production_error_boundary = ProductionErrorBoundary(in_production)
with production_error_boundary:
    ...
...
with production_error_boundary:
    ...
```

### Logging 

Exceptions are logged to the console using a Python `logging.Logger` based default logger (see [reference documentation](http://python-error-boundaries.readthedocs.io/en/latest/error_boundary.html) for details).

You can customize the default logger or provide your own logging altogether. This package ships with the following error boundary loggers:

- A Python `logging.Logger` based logger
- A Sentry/Raven logger

Error boundary loggers can be specfied with the `loggers` argument:

```python
from error_boundary import default_logging_logger, django_raven_logger

with ErrorBoundary(loggers=[default_logging_logger, django_raven_logger]):
    ...
```

Loggers have a very simple interface:

```python
def my_custom_logger(exc_info):
    ...
    
with ErrorBoundary(loggers=[my_custom_logger]):
    ...
```

See the documentation for the Python built-in `sys.exc_info()` function for details on the `exc_info` argument.

### Catching only some exceptions

You can pass the `dont_catch` option to never catch and suppress some exception types:

```python
with ErrorBoundary(dont_catch=[MyException]):
    raise MyException
# MyException is NOT suppressed but raised as if not using an error boundary in the first place.
```

### Customization

You can customize many aspects of error boundaries by overwriting hook methods provided by this package. See the [reference documentation](http://python-error-boundaries.readthedocs.io/en/latest/error_boundary.html) for details. In doubt, read the source, it's pretty simple.

| Method                                   | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| `should_propagate_exception(self, exc_info)` | Customize whether an exception should be propagated or suppressed. By default logs all exceptions whose type is not in `dont_catch`. Return value: bool |
| `should_log_exception(self, exc_info)`   | Customize whether an exception should be logged. By default logs all exceptions. Return value: bool |
| `log_exception(self, exc_info)`          | Log an exception. By default logs to all loggers from `get_loggers_for_exception`. |
| `get_loggers_for_exception(self, exc_info)` | Get all loggers to log an exception to.  |
| `on_no_exception(self)`                  | Called when no exception happened in the code block. |
| `on_propagate_exception(self, exc_info)` | Called when an exception is going to be propagated. |
| `on_suppress_exception(self, exc_info)`  | Called when an exception is going to be suppressed. |

### Django integration

There's a shorthand production error boundary for Django projects:

```python
from error_boundary import DjangoSettingErrorBoundary

with DjangoSettingErrorBoundary(...):
    # Suppress exceptions if DEBUG = False, otherwise propagate.

with DjangoSettingErrorBoundary('PRODUCTION', ...):
    # Suppress exceptions if PRODUCTION = True, otherwise propagate.

with DjangoSettingErrorBoundary('not MY_DEBUG', ...):
    # Suppress exceptions if MY_DEBUG = False, otherwise propagate.
```

### Reference documentation

[Reference documentation](http://python-error-boundaries.readthedocs.io/en/latest/error_boundary.html)
