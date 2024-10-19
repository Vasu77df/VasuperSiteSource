---
title: "Using UV to manage a python project"
date: 2024-10-18
tags: ["python"]
author: "Vasuper"
draft: false
---

## 1. Motivation

I wanted to put up a small snippet of code I did at work as a project on my Github, as it seemed like a common usage pattern that other folks can use of-the-shelf or copy from. Ain't that the spirit of open source?

Within Amazon, we have our own python project management and dependency management tooling that automates away a lot of the boilerplate setup in defining and distributing a Python project. 

They abstract away the setup and usage for tools and configurations like pyproject.toml, venv, mypy, black, pytest, etc. They essentially run these tools under the hood, but we as developers don't need to care about setting these up every time we start a new project. 

This also helps in standardizing linting rules, static type checker strictness, flake8 checks, etc, across a team, or organization.

I wanted to test out some external alternatives, that perform a similar functions. My goal, write less boilerplate configuration, and only care about the code. I also need a easy way to manage my dependencies and, run my linting and, testing tools.

---
## 2. The Tools
I was suprised, there are a lot of tools to choose from. Here's a list of some tools I came across and checked out.
- Peotry
- PDM
- Hatch
- Rye
- uv
- just the usual pip

I ended up choosing `uv`, for the following reasons.
- as the documentation seemed well structured enough that it was easy to follow.
- It has a straight forward pip interface, if I am not really inclined to learn any uv specifics.
- It covered a lot of features like
	- managing python versions
	- a script executor
	- project manager
		- init, to automate some boilerplate setup of a project.
		- add and remove dependencies.
		- build projects for distributions
		- and run projects in an isolated fashion
- It let's you install and run tools like black and ruff(the makers of uv also made ruff).
- It has some utility functions to manage caches.
- uv advertises to be 10 to 100x faster than pip, and in my experience just downloading and adding dependencies really has been a lot faster than pip.

My analysis was not purely a technical one, but was driven more on what I thought was easy to use, and partially driver by the media hype when the tool was portrayed as an pip replacement.

---

## 3. Installing uv 

The UV project has got a standalone installer linked below, it's as simple of running it to install uv.

> Its, always recommended to download and review scripts downloaded from the internet before running them.
> Maybe skip the pipe and add a -o to your curl and glance quickly if everything is dandy.

- https://docs.astral.sh/uv/getting-started/installation/

---

## 4. Setting up a project

I am working on building a CLI tool, and I want to setup my project as an application that can be packaged, and potentially just `pipx` installed or maybe `uv` installed.

To init such a project run:

```shell
uv init --app --package example
```

You will see a directory of your project setup with this file structure, after running this command.

```shell
❯ tree example
example
├── pyproject.toml
├── README.md
└── src
    └── example
        └── __init__.py

2 directories, 3 files
```

---

## 4. Setting up a virtual environment

To start a venv for your project you can run this command:

```shell
uv venv
```

Once run you'll see an output like so:

```
❯ uv venv
Using CPython 3.10.12 interpreter at: /usr/bin/python3
Creating virtual environment at: .venv
Activate with: source .venv/bin/activate
```

---

## 5. Adding dependencies to your project

Now let's add dependencies to the project.

For my project I have two deps `boto3` and `requests`

You can add dependencies with this command:

```shell
uv add boto3 requests
```

When you run this command `uv` will pull these dependencies into your project and update the `pyproject.toml` . 

`uv` aggressively caches packages across projects. If the dependencies have already been cached from another project, it will pull that into the current project and not re-download the dependencies.

```shell
❯ uv add boto3 requests
Resolved 12 packages in 364ms
   Built example @ file:///home/vasuper/personal_dev/example
Prepared 5 packages in 371ms
Installed 12 packages in 111ms
 + boto3==1.35.41
 + botocore==1.35.41
 + certifi==2024.8.30
 + charset-normalizer==3.4.0
 + example==0.1.0 (from file:///home/vasuper/personal_dev/example)
 + idna==3.10
 + jmespath==1.0.1
 + python-dateutil==2.9.0.post0
 + requests==2.32.3
 + s3transfer==0.10.3
 + six==1.16.0
 + urllib3==2.2.3
 +
❯ cat pyproject.toml
[project]
name = "example"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
authors = [
    { name = "Vasudevan Perumal", email = "vasu3797@gmail.com" }
]
requires-python = ">=3.10"
dependencies = [
    "boto3>=1.35.41",
    "requests>=2.32.3",
]

[project.scripts]
example = "example:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

---

## 6. Running your app or any command

`uv` has an run command one can use to run a project or just any adhoc script. UV manages the dependencies for you without manually managing environments.

```
uv run example
```

```
❯ uv run example
Hello from example!
```

You can also run any command with `uv run` within your project context. For example you can run the pytest against your project with:

```
uv run pytest
```

---

## 7. Freezing an environment

`uv` creates a `uv.lock file that acts as an cross platform lock file, which contains the exact resolved versions that are installed in the project environment. This can be committed source control to provide consistent and reproducible installation on different machines.

You can also lock the dependencies declared in `pyproject.toml` into a `requirements.txt` with this command:

```
uv pip compile pyproject.toml -o requirements.txt
```

You can then use this requirements in your environment
```
uv pip sync requirements.txt
```

---

## 8. Build the project for distribution

You can build your project using `uv` by running 

```shell
uv build
```

This will build the project into a distributable wheel or tarball.

```shell
uv build
Building source distribution...
Building wheel from source distribution...
Successfully built dist/example-0.1.0.tar.gz and dist/example-0.1.0-py3-none-any.whl
```
---

## 9. Publishing the project to a PyPI registry

`uv` also supports publishing the built project to a pypi registry.

```shell
uv publish
```

You can setup a PyPI token with `--token` or `UV_PUBLISH_TOKEN`, or set a username with `--username` or `UV_PUBLISH_USERNAME` and password with `--password` or `UV_PUBLISH_PASSWORD`.

---

## 10. The End

`uv` is a really fast python management tool, and it's user experience is pretty good. I would recommend it for folks looking for a tool to fit a similar need. In the next post I'll share what I built and managed with `uv`.

---