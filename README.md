# Guide for LangServe deployment

## Deactivate virtual environment
It is strongly recommended using Docker to run this. By now, LangServe does not go with a virtual environment well.
```bash
conda deactivate
```

## (Optional) Installation
Outside of any virtual environment, install the LangChain CLI if you haven't yet.

```bash
pip install -U langchain-cli
```

**Note**: By installing langchain-cli, you get to be connected to https://github.com/langchain-ai/langchain/tree/master/templates. 

It is highly likely that you see some dependency confilcts during installing it. But for most cases, it does not matter. 

## Creating LangServe app
```bash
langchain app new {$app_name} && cd {$app_name}
```

## Populating 'packages' folder 
**Note 1**: In 'packages' folder, actual source codes - LangChain Runnables - live.

**Note 2**: If you pull down source codes as reference from https://github.com/langchain-ai/langchain/tree/master/templates, 
```bash
langchain app add {$template_name}  # For instance, langchain app add pirate-speak
```

**Note 3**: If you pull down source codes as reference from any custom git repo,
```bash
langchain app add --repo {$git_user_name/$repo} --branch main  
```

## Modifying app/server.py
'app/server.py' is the file responsible for running LangServe. To it, you need to add lines for

1. importing the runnable to be invoked
```python
# Assumung that we will invoke 'rag_chain' from 'packages/pirate_speak/chain.py'
import os, sys
sys.path.append(os.path.join(os.path.dirname(__file__), "../packages")   )
from pirate_speak.chain import rag_chain
```
2. modigying 'add_routing' function
```python
add_routes(app, rag_chain, path="/pirate-speak")
# By doing this, you can access LangServe app via 'http://localhost:8080/pirate-speak/playground'
```

## Modifying pyproject.toml(s)
In case that there is/are pyproject.toml(s) for packages in 'packages' folder, you need to modify them as well as the top-level pyproject.toml to make sure there will be no dependency conflicts.

Sample of *root pyproject.toml* is as follows:
```shell
# Root Project Definition
# 'app/' directory is included as a package
[tool.poetry]
name = "on_prem_rag_docker"
version = "0.1.0"
description = ""
authors = ["Jet Taekyo Lee <jetleee.1888@gmail.com>"]
readme = "README.md"
packages = [
    { include = "app" },
]

# Shared dependences all across packages
# After pydantic are sub packages. 'develop=true' means changes to this package will be reflected without needing to reinstall.
[tool.poetry.dependencies]
python = "^3.11"
uvicorn = "^0.23.2"
langserve = {extras = ["server"], version = "^0.3.0"}
pydantic = "^2.9.2"
opensource-rag = {path = "package/opensource_rag", develop = true}


[tool.poetry.group.dev.dependencies]
langchain-cli = "^0.0.31"

# poetry-core is used as the build backend, which is typical for Poetry-managed projects.
[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

When encountering poetry error, consider swaping out build-system in pyproject.toml file(s)
```python
[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.build_meta"
``` 

## Building Docker image
```shell
docker build . -t my-langserve-app
```

Sample of Dockerfile is as follows:
```shell
FROM python:3.11-slim

RUN pip install poetry==1.8.4

RUN poetry config virtualenvs.create false

WORKDIR /code

COPY ./pyproject.toml ./README.md ./poetry.lock* ./

COPY ./package[s] ./packages

RUN poetry install  --no-interaction --no-ansi --no-root

COPY ./app ./app

RUN poetry install --no-interaction --no-ansi

EXPOSE 8080

CMD ["uvicorn", "app.server:app", "--host", "0.0.0.0", "--port", "8080"]
```

Sample of .dockerignore is as follows:
```shell
*env.list*
*.env*
*__pycache__*
```

Sample of env.list is as follows (No space and no quotation):
```shell
OPENAI_API_KEY={$OPENAI_API_KEY}
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY={$LANGCHAIN_API_KEY}
LANGCHAIN_PROJECT={$LANGCHAIN_PROJECT}
```

### Running the Image Locally

We expose port 8080 with the `-p 8080:8080` option.

```shell
docker run -d --rm --name {$container_name} --env-file env.list -p 8080:8080 my-langserve-app
```

we can view logs in real time by
```shell
docker logs -f {$container_name}
```
