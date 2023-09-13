# Customizing Logging in FastAPI Server with Uvicorn and Gunicorn

When running a FastAPI server with Gunicorn, you may find the need for more control over logging to tailor it to your specific requirements. This guide will walk you through the process of setting up a FastAPI server with custom logging using Uvicorn and Gunicorn, allowing you to achieve fine-grained control over your server's logging behavior.

## Logging Configuration

### Step 1: Import Required Modules

To begin, import the necessary modules and configure the logging settings. Here's a sample configuration that includes a timestamp and status code in the log messages:


```python
from uvicorn.workers import UvicornWorker
import logging
from typing import Any, Dict

# Logging Configuration
LOGGING_CONFIG: Dict[str, Any] = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "default": {
            "()": "uvicorn.logging.DefaultFormatter",
            "fmt": "%(levelprefix)s %(asctime)s - %(message)s",
            "datefmt": "%Y-%m-%d %H:%M:%S",
            "use_colors": None,
        },
        "access": {
            "()": "uvicorn.logging.AccessFormatter",
            "fmt": '%(levelprefix)s %(asctime)s %(client_addr)s - "%(request_line)s" %(status_code)s',
            "datefmt": "%Y-%m-%d %H:%M:%S",
        },
    },
    "handlers": {
        "default": {
            "formatter": "default",
            "class": "logging.StreamHandler",
            "stream": "ext://sys.stderr",
        },
        "access": {
            "formatter": "access",
            "class": "logging.StreamHandler",
            "stream": "ext://sys.stdout",
        },
    },
    "loggers": {
        "uvicorn": {"handlers": ["default"], "level": "INFO", "propagate": False},
        "uvicorn.error": {"level": "INFO"},
        "uvicorn.access": {"handlers": ["access"], "level": "INFO", "propagate": False},
    },
}
```

In this configuration, we've updated the fmt property of the "default" and "access" formatters to include the status code and a timestamp.


### Step 2: Create a Custom Uvicorn Worker

To further customize logging, you can create a custom Uvicorn worker class. This class will override some of Uvicorn's default behavior to use the custom loggers you defined earlier. Here's an example:

```python
# Inside a separate file, e.g., gunicorn_worker.py

class CustomUvicornWorker(UvicornWorker):
    def __init__(self, *args: Any, **kwargs: Any) -> None:
        super(CustomUvicornWorker, self).__init__(*args, **kwargs)

        # Override error logger
        error_logger = logging.getLogger("uvicorn.error")
        error_logger.handlers = self.log.error_log.handlers
        error_logger.setLevel(self.log.error_log.level)
        error_logger.propagate = False

        # Override access logger
        access_logger = logging.getLogger("uvicorn.access")
        access_logger.handlers = self.log.access_log.handlers
        access_logger.setLevel(self.log.access_log.level)
        access_logger.propagate = False

        # Additional configuration for Uvicorn worker
        config_kwargs: dict = {
            "app": None,
            "log_config": LOGGING_CONFIG,
            "timeout_keep_alive": self.cfg.keepalive,
            "timeout_notify": self.timeout,
            "callback_notify": self.callback_notify,
            "limit_max_requests": self.max_requests,
            "forwarded_allow_ips": self.cfg.forwarded_allow_ips,
        }

        # Handle SSL if configured
        if self.cfg.is_ssl:
            ssl_kwargs = {
                "ssl_keyfile": self.cfg.ssl_options.get("keyfile"),
                "ssl_certfile": self.cfg.ssl_options.get("certfile"),
                "ssl_keyfile_password": self.cfg.ssl_options.get("password"),
                "ssl_version": self.cfg.ssl_options.get("ssl_version"),
                "ssl_cert_reqs": self.cfg.ssl_options.get("cert_reqs"),
                "ssl_ca_certs": self.cfg.ssl_options.get("ca_certs"),
                "ssl_ciphers": self.cfg.ssl_options.get("ciphers"),
                
            }
            config_kwargs.update(ssl_kwargs)

        # Handle other configuration options
        if self.cfg.settings["backlog"].value:
            config_kwargs["backlog"] = self.cfg.settings["backlog"].value

        config_kwargs.update(self.CONFIG_KWARGS)

        self.config = Config(**config_kwargs)
```

This custom worker class ensures that Uvicorn uses the custom loggers and any additional configuration you specify.

## Running the FastAPI Server

### Step 3: Run FastAPI Server with Custom Server

Finally, you can run your FastAPI server with Gunicorn using the custom Uvicorn worker and the specified logging configuration. Here's the command:

```bash
gunicorn -w 1 -k your_module.CustomUvicornWorker main:app --bind 0.0.0.0:8001 --reload
```

Replace `your_module` with the actual Python module where you've defined the `CustomUvicornWorker`, and `main:app` with the appropriate FastAPI app entry point.

With this setup, you have full control over logging in your FastAPI server, making it easier to adapt your server's logging behavior to your specific needs and requirements.