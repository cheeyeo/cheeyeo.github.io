---
layout: post
show_meta: true
title: Centralized logging for flask application
header: Centralized logging for flask application
date: 2025-07-10 00:00:00
summary: How to use centralize logging in a flask application
categories: flask python logging
author: Chee Yeo
---

[Flask logging]: https://flask.palletsprojects.com/en/stable/logging/#injecting-request-information
[dash0 guide to logging in python]: https://www.dash0.com/guides/logging-in-python

In all my flask projects to date, logging is an after thought, with poorly structured log statements scattered across different modules or only relying on the default framework logger. This might work in dev mode but when deployed, its almost impossible to monitor and observe any issues with the running application.

We can apply a technique known as **structure query logging**. Structured Query Logging refers to a technique of recording logs in a wel-defined, machine-readable format such as JSON. Storing logs in free form text makes it difficult to search for and analyze for production issues. Usinng structured query logging, we can perform more advanced search queries on logs to triage operational issues. AWS Cloudwatch Logs support structured query logging. We can publish our logs to cloudwatch in JSON format.

The example below is based on a Flask web application running on ECS Fargate mode but the same concept applies to any python application.

Firstly, we need to centralize logging. Based on this excellent guide [dash0 guide to logging in python], I was able to refactor the application to load its logging configuration into external config files before the actual Flask application is instantiated. Flask uses the logging module and we need to set this up before the application object is initialized.  This allows us to load different config files based on environment type. For example, in dev mode, we only need to output logs to stdout while running on AWS, we can output the logs in JSON format.


Below is an example of the logging config for deployment to AWS:
{% highlight yaml %}
version: 1
disable_existing_loggers: False

filters:
  add_context:
    (): board.log_context.ContextFilter

formatters:
  stream_formatter:
    format: "%(asctime)s %(levelname)s: %(message)s [in %(pathname)s:%(lineno)d]"
  json_formatter:
    (): pythonjsonlogger.jsonlogger.JsonFormatter
    format: "%(asctime)s %(name)s %(levelname)s %(message)s"
    rename_fields:
      levelname: level
      asctime: time

handlers:
  console:
    class: logging.StreamHandler
    formatter: json_formatter
    level: INFO
    stream: ext://sys.stdout
    filters: [add_context]

loggers:
  board:
    handlers: [console]
    level: INFO
    propagate: False

root:
  handlers: [console]
  level: WARNING
{% endhighlight %}

`python-json-logger` library provides a JSON formatter that will convert your logs into a JSON structure. We can install the library and apply the formatter to the `logging.StreamHandler` as the application runs in a docker container and we need to stream the logs to stdout for Cloudwatch.

We also define a logging filter `ContextFilter` class. It subclasses `logging.Filter` and only has 1 overwritten method which filters each log record as it arrives. Here, we are using it to attach additional fields to each record such as `hostname` and `process_id` as shown above. The context filter class is defined below:

{% highlight python %}
_log_context = contextvars.ContextVar("log_context", default={})


class ContextFilter(logging.Filter):
    def __init__(self, name=""):
        super().__init__(name)
        self.hostname = socket.gethostname()
        self.process_id = os.getpid()

    def filter(self, record):
        record.hostname = self.hostname
        record.process_id = self.process_id

        context = _log_context.get()
        for key, value in context.items():
            setattr(record, key, value)
        
        return True


@contextmanager
def add_to_log_context(**kwargs):
    current_context = _log_context.get()
    new_context = {**current_context, **kwargs}

    # set the new context and get the token
    token = _log_context.set(new_context)

    try:
        yield
    finally:
        _log_context.reset(token)
{% endhighlight %}

Note that we utilise a `_log_context` context var object which is set via `add_to_log_context` function. This function allows for more fine-grained logging via code blocks attached to say, specific routes, which will be explained later.

The second stage to apply centralized logging is some form of middleware which would intercept the request and before the response is returned. I used Flask `before_request` and `after_this_request` decorators to achieve this. Below is my implementation:

{% highlight python %}
@app.before_request
    def log_before_request():
        start_time = time.monotonic()
        client_ip = request.headers.get("X-Forwarded-For") or request.remote_addr
        app.logger.info(
            f"incoming {request.method} from {request.path}",
            extra={
                "method": request.method,
                "url": request.url,
                "path": request.path,
                "client_ip": client_ip,
                "user_agent": request.headers.get("user-agent")
            }
        )

        @after_this_request
        def log_after_request(response):
            duration = (time.monotonic() - start_time) * 100
            log_level = logging.INFO
            if response.status_code >= 500:
                log_level = logging.ERROR
            elif response.status_code >= 400:
                log_level = logging.WARNING
            
            app.logger.log(
                log_level,
                f"{request.method} to {request.path} completed with status {response.status_code}",
                extra={
                    "method": request.method,
                    "url": request.url,
                    "path": request.path,
                    "status_code": response.status_code,
                    "latency": duration
                }
            )

            return response

{% endhighlight %}

In `log_before_request`, we defined a start timer and try to capture other request attributes such as ip addr, path and method. `after_this_request` decorator runs immediately after the request is processed. Here, we log the request method, path and status code. The above creates 2 entries per event, before and after each request.

Following is an example of a log entry when accessing the login page:

{% highlight json %}
{"
  time": "2025-07-18 15:36:36,965", 
  "name": "board", 
  "level": "INFO", 
  "message": "incoming GET from /login", 
  "method": "GET", 
  "path": "/login", 
  "client_ip": "172.18.0.1", 
  "user_agent": "Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0", "hostname": "8a79d4222d78", 
  "process_id": 60, 
  "url": "http://localhost:8000/login?next=/", 
  "remote_addr": "172.18.0.1"
}

{
  "time": "2025-07-18 15:36:37,057",
  "name": "board",
  "level": "INFO",
  "message": "GET to /login completed with status 200",
  "status_code": 200,
  "duration_ms": 9.182815499934804,
  "hostname": "8a79d4222d78",
  "process_id": 60,
  "url": "http://localhost:8000/login?next=/", 
  "remote_addr": "172.18.0.1"
}
{% endhighlight %}

The `add_to_log_context` function allows the addition of specific messages to the log stream. For example, we can add an additional log statement if a user visits the /login page:

{% highlight python %}
@bp.route("/login", methods=['GET', 'POST'])
def login():
  with add_to_log_context(user_email=form.email.data):
    current_app.logger.info(f"User {current_user.email} has logged in.")
  
  ...

{% endhighlight %}

This would result in the following log entries:

{% highlight json %}
{"time": "2025-07-19 16:06:18,940", "name": "board", "level": "INFO", "message": "incoming POST from /login", "method": "POST", "url": "http://localhost:8000/login?next=/", "path": "/login", "client_ip": "172.18.0.1", "user_agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36", "hostname": "3832968803fa", "process_id": 13}

{"time": "2025-07-19 16:06:19,503", "name": "board", "level": "INFO", "message": "User test@example.com has logged in.", "hostname": "3832968803fa", "process_id": 13, "user_email": "test@example.com"}

{"time": "2025-07-19 16:06:19,503", "name": "board", "level": "INFO", "message": "POST to /login completed with status 302", "method": "POST", "url": "http://localhost:8000/login?next=/", "path": "/login", "status_code": 302, "latency": 56.536332900031994, "hostname": "3832968803fa", "process_id": 13}
{% endhighlight %}

Notice how an additional second log record is created with a custom message of a logged in user. This could help improve observability in certain parts of the application.

In summary, using centralized logging in a web application can help to simplify the use of the python logging module. Structured query logging further provides better observability in human-readable logs that provide more context and are searchable.
