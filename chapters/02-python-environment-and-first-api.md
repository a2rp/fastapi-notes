# 02. Python environment and the first API

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: FastAPI and the ASGI request lifecycle](./01-fastapi-and-asgi.md) | [Notes index](../README.md) | [Next: Path operation routing](./03-path-operation-routing.md) |

## Check Python and keep dependencies isolated

Use a supported Python version that is compatible with the FastAPI and Pydantic releases in the project. Check which interpreter your terminal will use:

~~~sh
python --version
python -m pip --version
~~~

On Windows, the Python launcher can select an installed Python version:

~~~powershell
py -3 --version
~~~

Create a virtual environment in the project directory. It keeps this project's packages separate from other Python projects:

~~~sh
python -m venv .venv
~~~

On Windows, use the Python launcher if `python` does not point to the interpreter you intend to use:

~~~powershell
py -3 -m venv .venv
~~~

Activate the environment in PowerShell:

~~~powershell
.\.venv\Scripts\Activate.ps1
~~~

On macOS or Linux, activate it with:

~~~sh
source .venv/bin/activate
~~~

The shell prompt usually shows `(.venv)` after activation. Activation is convenient, but not required. You can run the environment's Python directly, for example `.\.venv\Scripts\python.exe` on Windows or `.venv/bin/python` on macOS and Linux.

## Install FastAPI and its standard dependencies

FastAPI's standard extra includes Uvicorn, an ASGI server, and the `fastapi` command-line tool:

~~~sh
python -m pip install --upgrade pip
python -m pip install "fastapi[standard]"
~~~

Use `python -m pip` so the package installer belongs to the same Python interpreter that will run the app. Keep the quotes around `fastapi[standard]`, especially in shells that treat square brackets specially.

A small project can record its direct dependency in `requirements.txt`:

~~~text
fastapi[standard]
~~~

Install from that file with:

~~~sh
python -m pip install -r requirements.txt
~~~

For a deployed application, use a lockfile or pinned dependency versions so another environment can install the versions that were tested. Dependency management is covered again in the configuration and deployment chapters.

## Create a small project

For a first app, keep the file in the project root:

~~~text
reading-list/
|-- .venv/
|-- main.py
`-- requirements.txt
~~~

Add the app from chapter 1 to `main.py`:

~~~python
from fastapi import FastAPI

app = FastAPI(title="Reading List API")


@app.get("/")
async def read_home():
    return {"message": "Hello, FastAPI!"}
~~~

The virtual environment is local to this machine. It should not be committed. The repository `.gitignore` already excludes `.venv`, Python cache files, and `.env` values.

## Start the development server

Run Uvicorn from the directory containing `main.py`:

~~~sh
python -m uvicorn main:app --reload
~~~

The import string `main:app` has two parts:

- `main` is the Python module from `main.py`.
- `app` is the FastAPI object created inside that module.

`--reload` watches Python files and restarts Uvicorn after changes. It is intended for development and should not be used as the production process.

Open `http://127.0.0.1:8000/docs` to see the generated API documentation. Stop the development process with `Ctrl+C`.

## Organize the app into a package

As the app grows, move the entry module into a package:

~~~text
reading-list/
|-- app/
|   |-- __init__.py
|   `-- main.py
|-- .venv/
`-- requirements.txt
~~~

The `__init__.py` file makes the package boundary clear. Start the app from the project root with:

~~~sh
python -m uvicorn app.main:app --reload
~~~

Here, `app.main:app` means: import the `app.main` module, then load the variable named `app`.

Keep the Python import path consistent with the directory where you run the command. If Uvicorn cannot import the module, check the current directory, the package names, and the object name after the colon.

## Use the FastAPI command-line tool

Installing `fastapi[standard]` also provides the FastAPI command. It can detect the app in a file and start a development server:

~~~sh
fastapi dev main.py
~~~

This is a convenient alternative for local development. Uvicorn remains the ASGI server running the app. Use one start command in the project's instructions so everyone knows which entry point to run.

## Verify the environment

Confirm that the packages are installed in the active environment:

~~~sh
python -m pip show fastapi uvicorn
~~~

If you see an import error, check that the environment is active and that installation used its Python interpreter. Installing packages globally can make one terminal work while another terminal, editor, test process, or deployment cannot find the same packages.

## Common setup problems

- **The command uses another Python installation.** Compare `python --version` with `python -m pip --version`, then activate the project environment.
- **PowerShell blocks the activation script.** Use the environment's Python executable directly instead of changing system execution settings.
- **Uvicorn cannot import `main:app`.** Run the command from the project root and check both names in the import string.
- **The port is already in use.** Stop the other server or choose a different port with `--port`.
- **Another device cannot reach the server.** Uvicorn binds to `127.0.0.1` by default. Binding to `0.0.0.0` allows network access and should only be done when that exposure is intended.

## Main references

- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [FastAPI manually running an ASGI server](https://fastapi.tiangolo.com/deployment/manually/)
- [Python virtual environments](https://docs.python.org/3/library/venv.html)
- [Python pip guide](https://pip.pypa.io/en/stable/user_guide/)
- [Uvicorn settings](https://www.uvicorn.org/settings/)