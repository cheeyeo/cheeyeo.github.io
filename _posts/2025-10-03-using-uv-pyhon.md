---
layout: post
show_meta: true
title: Using uv in python projects
header: Using uv in python projects
date: 2025-10-03 00:00:00
summary: A short introduction to using uv in python projects
categories: python uv
author: Chee Yeo
---

[UV Python package manager]: https://docs.astral.sh/uv/
[UV Migration Guide]: https://docs.astral.sh/uv/guides/migration/pip-to-project/#requirements-files


In most of my python projects, I default to using the built-in python venv module to create a virtualenv and installs the required dependencies from a `requirements.txt` file using `pip install -r requirements.txt`.

Using [UV Python package manager], we can streamline the process to using a single binary to install the dependencies as well as manage the required python versions.

To create a new project using `uv`, we run `uv init <project name>` to create a minimal project layout which has the following structure:

{% highlight shell %}
├── .python-version
├── main.py
├── pyproject.toml
└── README.md
{% endhighlight %}

`uv` automatically creates the `.python-version` file through auto-discovery of the python versions available on the localhost. To disable the creation of `main.py` we can use `uv init --bare` to create a minimal project with only a `pyproject.toml` file.

To setup a venv, we can run `uv venv` which creates a virtualenv inside a `.venv` directory within the application foler. It is recommended to also specify the python version while creating the venv:

{% highlight shell %}
  uv venv --python 3.13.8
{% endhighlight %}

`uv` will try to auto discover the python version on the local system and download it if its not available. The uv downloaded python versions can be located within `<HOME DIR>/.local/share/uv/python/<python version>`

`uv` provides 2 different approaches to managing dependencies: using the project API via `uv add` or using the lower level api via `uv pip install`. 

To use the project level API to add a dependency, we can use:

{% highlight shell %}
  source .venv/bin/activate

  uv add requests
{% endhighlight %}

The above activated the venv and installed the `requests` library. It also created a `uv.lock` file which specifies the version of each of the packages installed and its dependencies. The `pyproject.toml` dependencies list will also be updated with the dependency installed. 

To remove the dependency, we can use:

{% highlight shell %}
  uv remove requests
{% endhighlight %}

This will update both the `pyproject.toml` and `uv.lock` files.

To use the lower level API with pip, we can run:
{% highlight shell %}
  uv pip install requests
{% endhighlight %}

However, this will not update the `pyproject.toml` and `uv.lock` files.

As per the documentation, the recommended workflow is to choose one of the two approaches. The recommendation from the UV team is to use the project level API on new projects and use the lower level APIs for existing projects with established workflows.


#### To recreate an env from existing uv project:

* Run `uv venv --python <version>` to create a venv
* Run `source .venv/bin/activate`
* Run `uv sync --locked` to install the exact dependencies versions in the `uv.lock` file

The `uv sync --locked` command updates the venv to match the lock file exactly, supporting reproducible builds especially in CI/CD pipelines. If the lock file is out of date, `uv` will resolve the dependencies and update `uv.lock`


#### To migrate an existing project from using pip to uv:

* Run `uv init` to create a `pyproject.toml` file in project directory.
* Run `uv venv --python <version>` to create a venv.
* Run `uv add -r requirements.txt` to import the requirements.txt file if it exists. This will install all the dependencies and update the uv.lock and pyproject.toml files to match the environment.

The [UV Migration Guide] has more detailed instructions on how to perform the migration including how to manage both development and production dependencies.

Having a common dependency management system allows for reproducible builds and faster development cycles without the issues of debugging failures in builds. Using `uv` reminds me of using `bundler` in ruby projects. Since `uv` also manages python versions, it also helps to resolve the issue of building multiple python versions manually in order to create a project environment.
