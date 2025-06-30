---
layout: post
show_meta: true
title: Using Docker with Flask with autoreload and sync
header: Using Docker with Flask with autoreload and sync
date: 2025-06-12 00:00:00
summary: How to use Docker with Flask for local and deployment
categories: flask python docker
author: Chee Yeo
---

To be able to deploy a Flask web application, we need to be able to bundle the application code and its dependencies into a reproducible and portable format. In a previous post, I showed how one could use ElasticBeanstalk to deploy it from within the application directory itself. By specifying the environment type manually in the CLI, Elastibeanstalk will create the resources automatically to host the application.

However, this is not how applications are developed in an agile manner. We need to be able to bundle the application and its dependencies together in a reproducible and deterministic format for portability so it can be deployed either via CI/CD pipelines or by the developer. 

Docker containers are widely used format for this reason. In addition, it is also supported by Elasticbeanstalk. Below specifies an approach I use to develop the Flask application locally using Docker while keeping the same features in standalone development mode such as hot code reload. 

Docker compose is used to link the database and application containers together in development.


### Auto reload with gunicorn and docker-compose

[gunicorn configuration documentation]: https://docs.gunicorn.org/en/latest/configure.html

[reload configuration]: https://docs.gunicorn.org/en/latest/settings.html#reload

[reload_extra_files configuration]: [reload configuration]

[docker compose develop specification]: https://docs.docker.com/reference/compose-file/develop/


According to the [gunicorn configuration documentation], gunicorn looks for a configuration file which defaults to `gunicorn.conf.py`. gunicorn supports the [reload configuration] which restarts the workers when code changes are detected. This is similar to using the `debug` flag when running the application in development mode using `flask` CLI. 

Note that reload only works for application code. It won't detect changes in the template for example. To handle that, we need to use the [reload_extra_files configuration]. This extends the reload option to also watch and reload on additional files such as template changes.

The `gunicorn.conf.py` is defined in the application root directory and contains the following:

{% highlight python %}
from glob import glob
import os
import multiprocessing


bind = ':5000'
workers = multiprocessing.cpu_count()

if os.getenv('AUTO_RELOAD'):
    reload = True
    reload_extra_files = glob('board/templates/**/*.html', recursive=True) + glob('board/static/*.css', recursive=True)
{% endhighlight %}

We define each parameter by name and its value, similar to how we set the parameters via the gunicorn CLI. We set an environment variable of `AUTO_RELOAD` during local development. If true, this enables auto reload. During deployment to production, this will not be set and the application will not have reload option enabled.

The docker-compose.yml file will also need to be updated to include the `develop` block. According to [docker compose develop specification], the `develop` block helps with local development by updating the service container image specified without having to stop and restart compose. If we don't enable it, compose will keep the service container running with the old image and the changes will not be in sync.

We specify the `watch` command to watch for changes to local directories or files and perform a corresponding action to update the service container image. The valid actions are:

* rebuild
  Rebuilds the service image and recreates service with updated image.

* restart
  Restarts the service container

* sync
  Keeps existing service container running but copies the source changes to the target path within the container.

* sync+restart
  Synchronizes the file as above but also restarts the container

* sync+exec
  Synchronizes the source files within container and then executes a command inside the container.

Below is the `develop` block I use within my compose.yml file:

{% highlight yaml %}
webapp:
    build:
      dockerfile: Dockerfile
      context: .
    container_name: webapp
    depends_on:
      db:
        condition: service_healthy
        restart: true
    env_file: .env
    environment:
      RDS_HOSTNAME: db # overwrite here to link to db above ^
      AUTO_RELOAD: True # set to true to enable gunicorn reload; see gunicorn.conf.py 
      # overwrite other env var here
      # POSTS_PER_PAGE: 2
    restart: always
    develop:
      watch:
        - action: sync
          path: ./board
          target: /app/board
          ignore:
            - __pycache__
        - action: rebuild
          path: ./requirements.txt
        - action: rebuild
          path: ./migrations
    ports:
      - 8000:5000
{% endhighlight %}

Each `watch` statement is a rule to perform when the change it specifies is detected. For the main application code, we set watch on the application code subdirectory. Any changes will be synced to the `/app/board` code directory in the service container. Since gunicorn is already set to reload, this will show the changes made. We pass `__pycache__` to be ignored. This is a list of files or directories to ignore changes to.

Next, we pass `requirements.txt` and the `migrations` directory to watch to rebuild the service container image if any changes are detected. Both changes require a full rebuild of the image i.e. updates to dependencies require running pip install and updates to migrations require running `flask db migrate & flask db upgrade`. 

To simulate reload of the application code, I made a change to one of the model files:

![Application sync](/assets/img/flask/autoreload/reload1.png)

To simulate rebuild following dependency changes, I added a new package using pip in a separate terminal and updated the `requirements.txt` file using `pip freeze > requirements.txt`. This caused a rebuild of the service container as shown below:

![Rebuild](/assets/img/flask/autoreload/reload2.png)

To simulate a new migration being added, I added a new database field to one of the models and ran `flask db:migrate` in a separate terminal to generate a new migration file. This caused a rebuild of the service container as shown below:

![Rebuild](/assets/img/flask/autoreload/reload3.png)

By using this approach, I was able to develop locally using docker with auto-reload of the Flask application. 

Further updates that could be used to improve the process include:

* Using multi-stage build for the python image
* Using inotify in the container image for gunicorn reload.
