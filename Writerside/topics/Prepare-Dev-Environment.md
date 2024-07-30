# Prepare Dev Environment

## On python versions {id="python_version"}

Tracking multiple versions of python is not the same as tracking multiple virtual environments.
With a little bit of work, switching from one python version to another can be greatly simplified.

Using
[pyenv](https://github.com/pyenv/pyenv)
greatly simplifies python version selection:

   ```Bash
   pyenv install 3.10
   pyenv global 3.10
   ```
There is no need to specify the python micro version number. 

It is preferable to keep the global python environment void of any pip installs, other
than having the most up-to-date latest version of `pip` itself.

Project-specific and special packages can then be installed 
in a separate python virtual environment. 

## On python virtual environments {id="virtual_env"}

There is more than one way of preparing clean and nimble virtual environment. We describe
two, with our preference being the last one. 

### `venv`
The standard way of creating lightweight virtual environments is to use the 
[venv](https://docs.python.org/3/library/venv.html) module from the standard library.

### `pyenv-virtualenv`

Once you have decided to use `pyenv` as a python manager, it is worth considering 
[pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) for virtual environment
management. 

Assuming we have already defined (exported) a shell global variable named `${PROJECT_ID}`, 
creating a virtual environment boils down to issuing:

```bash
pyenv virtualenv ${PROJECT_ID}
pyenv activate ${PROJECT_ID}
```

At this point your prompt should change indicating that you are working in the virtual environment.
The only pip packages that are available should be `pip` and `setuptools`. To stay on the 
safe side, it is worth updating `pip` with

```Bash
pip install -U pip
pip install -U setuptools
```

In this virtual environment, there should be the latests 
version of only two packages: `pip` and `setuptools`.


## Bonus: Jupyter custom kernel {id="custom_kernel"}

To prepare a clean dev env with a custom jupyter kernel, use the following:

> **Important**
>
> Make sure you have activated the virtual environment and that 
> you are issuing the following commands from it. You can then also start
> `jupyter lab` from the same virtual environment.
>
{style="note"}

```bash
pip install -U ipykernel
ipython kernel install --user --name="${PROJECT_ID}"
pip install jupyterlab
```

At this point, you should be ready to use your suitable/favorite
python version in a clean virtual environment without any pip-installed
packages. In addition, you can test you work using this same environment
from a jupyter notebook.

## TL;DR

Here are the outputs

### Set Up PROJECT_ID

The Project ID is the first linchpin of this project. 
```Bash
export PROJECT_ID=ray-whisper
```

### Install ipykernel

That is the first install in the clean environment.

```bash
$ pip install -U ipykernel
Collecting ipykernel
  Downloading ipykernel-6.29.5-py3-none-any.whl.metadata (6.3 kB)
Collecting appnope (from ipykernel)
  Using cached appnope-0.1.4-py2.py3-none-any.whl.metadata (908 bytes)
Collecting comm>=0.1.1 (from ipykernel)
  Using cached comm-0.2.2-py3-none-any.whl.metadata (3.7 kB)
Collecting debugpy>=1.6.5 (from ipykernel)
  Using cached debugpy-1.8.2-cp310-cp310-macosx_11_0_x86_64.whl.metadata (1.1 kB)
Collecting ipython>=7.23.1 (from ipykernel)
  Downloading ipython-8.26.0-py3-none-any.whl.metadata (5.0 kB)
Collecting jupyter-client>=6.1.12 (from ipykernel)
  Using cached jupyter_client-8.6.2-py3-none-any.whl.metadata (8.3 kB)
Collecting jupyter-core!=5.0.*,>=4.12 (from ipykernel)
  Using cached jupyter_core-5.7.2-py3-none-any.whl.metadata (3.4 kB)
Collecting matplotlib-inline>=0.1 (from ipykernel)
  Using cached matplotlib_inline-0.1.7-py3-none-any.whl.metadata (3.9 kB)
Collecting nest-asyncio (from ipykernel)
  Using cached nest_asyncio-1.6.0-py3-none-any.whl.metadata (2.8 kB)
Collecting packaging (from ipykernel)
  Using cached packaging-24.1-py3-none-any.whl.metadata (3.2 kB)
Collecting psutil (from ipykernel)
  Using cached psutil-6.0.0-cp36-abi3-macosx_10_9_x86_64.whl.metadata (21 kB)
Collecting pyzmq>=24 (from ipykernel)
  Using cached pyzmq-26.0.3-cp310-cp310-macosx_10_15_universal2.whl.metadata (6.1 kB)
Collecting tornado>=6.1 (from ipykernel)
  Using cached tornado-6.4.1-cp38-abi3-macosx_10_9_x86_64.whl.metadata (2.5 kB)
Collecting traitlets>=5.4.0 (from ipykernel)
  Using cached traitlets-5.14.3-py3-none-any.whl.metadata (10 kB)
Collecting decorator (from ipython>=7.23.1->ipykernel)
  Using cached decorator-5.1.1-py3-none-any.whl.metadata (4.0 kB)
Collecting jedi>=0.16 (from ipython>=7.23.1->ipykernel)
  Using cached jedi-0.19.1-py2.py3-none-any.whl.metadata (22 kB)
Collecting prompt-toolkit<3.1.0,>=3.0.41 (from ipython>=7.23.1->ipykernel)
  Using cached prompt_toolkit-3.0.47-py3-none-any.whl.metadata (6.4 kB)
Collecting pygments>=2.4.0 (from ipython>=7.23.1->ipykernel)
  Using cached pygments-2.18.0-py3-none-any.whl.metadata (2.5 kB)
Collecting stack-data (from ipython>=7.23.1->ipykernel)
  Using cached stack_data-0.6.3-py3-none-any.whl.metadata (18 kB)
Collecting exceptiongroup (from ipython>=7.23.1->ipykernel)
  Using cached exceptiongroup-1.2.1-py3-none-any.whl.metadata (6.6 kB)
Collecting typing-extensions>=4.6 (from ipython>=7.23.1->ipykernel)
  Using cached typing_extensions-4.12.2-py3-none-any.whl.metadata (3.0 kB)
Collecting pexpect>4.3 (from ipython>=7.23.1->ipykernel)
  Using cached pexpect-4.9.0-py2.py3-none-any.whl.metadata (2.5 kB)
Collecting python-dateutil>=2.8.2 (from jupyter-client>=6.1.12->ipykernel)
  Using cached python_dateutil-2.9.0.post0-py2.py3-none-any.whl.metadata (8.4 kB)
Collecting platformdirs>=2.5 (from jupyter-core!=5.0.*,>=4.12->ipykernel)
  Using cached platformdirs-4.2.2-py3-none-any.whl.metadata (11 kB)
Collecting parso<0.9.0,>=0.8.3 (from jedi>=0.16->ipython>=7.23.1->ipykernel)
  Using cached parso-0.8.4-py2.py3-none-any.whl.metadata (7.7 kB)
Collecting ptyprocess>=0.5 (from pexpect>4.3->ipython>=7.23.1->ipykernel)
  Using cached ptyprocess-0.7.0-py2.py3-none-any.whl.metadata (1.3 kB)
Collecting wcwidth (from prompt-toolkit<3.1.0,>=3.0.41->ipython>=7.23.1->ipykernel)
  Using cached wcwidth-0.2.13-py2.py3-none-any.whl.metadata (14 kB)
Collecting six>=1.5 (from python-dateutil>=2.8.2->jupyter-client>=6.1.12->ipykernel)
  Using cached six-1.16.0-py2.py3-none-any.whl.metadata (1.8 kB)
Collecting executing>=1.2.0 (from stack-data->ipython>=7.23.1->ipykernel)
  Using cached executing-2.0.1-py2.py3-none-any.whl.metadata (9.0 kB)
Collecting asttokens>=2.1.0 (from stack-data->ipython>=7.23.1->ipykernel)
  Using cached asttokens-2.4.1-py2.py3-none-any.whl.metadata (5.2 kB)
Collecting pure-eval (from stack-data->ipython>=7.23.1->ipykernel)
  Using cached pure_eval-0.2.2-py3-none-any.whl.metadata (6.2 kB)
Downloading ipykernel-6.29.5-py3-none-any.whl (117 kB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 117.2/117.2 kB 1.0 MB/s eta 0:00:00
Using cached comm-0.2.2-py3-none-any.whl (7.2 kB)
Using cached debugpy-1.8.2-cp310-cp310-macosx_11_0_x86_64.whl (1.7 MB)
Downloading ipython-8.26.0-py3-none-any.whl (817 kB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 817.9/817.9 kB 2.4 MB/s eta 0:00:00
Using cached jupyter_client-8.6.2-py3-none-any.whl (105 kB)
Using cached jupyter_core-5.7.2-py3-none-any.whl (28 kB)
Using cached matplotlib_inline-0.1.7-py3-none-any.whl (9.9 kB)
Using cached pyzmq-26.0.3-cp310-cp310-macosx_10_15_universal2.whl (1.4 MB)
Using cached tornado-6.4.1-cp38-abi3-macosx_10_9_x86_64.whl (433 kB)
Using cached traitlets-5.14.3-py3-none-any.whl (85 kB)
Using cached appnope-0.1.4-py2.py3-none-any.whl (4.3 kB)
Using cached nest_asyncio-1.6.0-py3-none-any.whl (5.2 kB)
Using cached packaging-24.1-py3-none-any.whl (53 kB)
Using cached psutil-6.0.0-cp36-abi3-macosx_10_9_x86_64.whl (250 kB)
Using cached jedi-0.19.1-py2.py3-none-any.whl (1.6 MB)
Using cached pexpect-4.9.0-py2.py3-none-any.whl (63 kB)
Using cached platformdirs-4.2.2-py3-none-any.whl (18 kB)
Using cached prompt_toolkit-3.0.47-py3-none-any.whl (386 kB)
Using cached pygments-2.18.0-py3-none-any.whl (1.2 MB)
Using cached python_dateutil-2.9.0.post0-py2.py3-none-any.whl (229 kB)
Using cached typing_extensions-4.12.2-py3-none-any.whl (37 kB)
Using cached decorator-5.1.1-py3-none-any.whl (9.1 kB)
Using cached exceptiongroup-1.2.1-py3-none-any.whl (16 kB)
Using cached stack_data-0.6.3-py3-none-any.whl (24 kB)
Using cached asttokens-2.4.1-py2.py3-none-any.whl (27 kB)
Using cached executing-2.0.1-py2.py3-none-any.whl (24 kB)
Using cached parso-0.8.4-py2.py3-none-any.whl (103 kB)
Using cached ptyprocess-0.7.0-py2.py3-none-any.whl (13 kB)
Using cached six-1.16.0-py2.py3-none-any.whl (11 kB)
Using cached pure_eval-0.2.2-py3-none-any.whl (11 kB)
Using cached wcwidth-0.2.13-py2.py3-none-any.whl (34 kB)
Installing collected packages: wcwidth, pure-eval, ptyprocess, typing-extensions, traitlets, tornado, six, pyzmq, pygments, psutil, prompt-toolkit, platformdirs, pexpect, parso, packaging, nest-asyncio, executing, exceptiongroup, decorator, debugpy, appnope, python-dateutil, matplotlib-inline, jupyter-core, jedi, comm, asttokens, stack-data, jupyter-client, ipython, ipykernel
Successfully installed appnope-0.1.4 asttokens-2.4.1 comm-0.2.2 debugpy-1.8.2 decorator-5.1.1 exceptiongroup-1.2.1 executing-2.0.1 ipykernel-6.29.5 ipython-8.26.0 jedi-0.19.1 jupyter-client-8.6.2 jupyter-core-5.7.2 matplotlib-inline-0.1.7 nest-asyncio-1.6.0 packaging-24.1 parso-0.8.4 pexpect-4.9.0 platformdirs-4.2.2 prompt-toolkit-3.0.47 psutil-6.0.0 ptyprocess-0.7.0 pure-eval-0.2.2 pygments-2.18.0 python-dateutil-2.9.0.post0 pyzmq-26.0.3 six-1.16.0 stack-data-0.6.3 tornado-6.4.1 traitlets-5.14.3 typing-extensions-4.12.2 wcwidth-0.2.13
```
{collapsible="true" collapsed-title="pip install -U ipykernel"}

### Create Custom Jupyter Kernel

```bash
$ ipython kernel install --user --name="${PROJECT_ID}"
Installed kernelspec ray-whisper in ${HOME}/Library/Jupyter/kernels/ray-whisper
```

### Install jupyter lab

At this point we have the following jupyter packages that got pulled with ipykernel

```Bash
$ pip list | grep jupyter
jupyter_client    8.6.2
jupyter_core      5.7.2
```

Next we pip install jupyter, which pulls a lot of dependencies. Because I have done
it more than a few times on my machine, most of the packages are cached.

```Bash
$ pip install -U jupyter
Collecting jupyter
  Using cached jupyter-1.0.0-py2.py3-none-any.whl.metadata (995 bytes)
Collecting notebook (from jupyter)
  Using cached notebook-7.2.1-py3-none-any.whl.metadata (10 kB)
Collecting qtconsole (from jupyter)
  Using cached qtconsole-5.5.2-py3-none-any.whl.metadata (5.1 kB)
Collecting jupyter-console (from jupyter)
  Using cached jupyter_console-6.6.3-py3-none-any.whl.metadata (5.8 kB)
Collecting nbconvert (from jupyter)
  Using cached nbconvert-7.16.4-py3-none-any.whl.metadata (8.5 kB)
Requirement already satisfied: ipykernel in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from jupyter) (6.29.5)
Collecting ipywidgets (from jupyter)
  Using cached ipywidgets-8.1.3-py3-none-any.whl.metadata (2.4 kB)
Requirement already satisfied: appnope in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (0.1.4)
Requirement already satisfied: comm>=0.1.1 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (0.2.2)
Requirement already satisfied: debugpy>=1.6.5 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (1.8.2)
Requirement already satisfied: ipython>=7.23.1 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (8.26.0)
Requirement already satisfied: jupyter-client>=6.1.12 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (8.6.2)
Requirement already satisfied: jupyter-core!=5.0.*,>=4.12 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (5.7.2)
Requirement already satisfied: matplotlib-inline>=0.1 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (0.1.7)
Requirement already satisfied: nest-asyncio in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (1.6.0)
Requirement already satisfied: packaging in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (24.1)
Requirement already satisfied: psutil in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (6.0.0)
Requirement already satisfied: pyzmq>=24 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (26.0.3)
Requirement already satisfied: tornado>=6.1 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (6.4.1)
Requirement already satisfied: traitlets>=5.4.0 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipykernel->jupyter) (5.14.3)
Collecting widgetsnbextension~=4.0.11 (from ipywidgets->jupyter)
  Using cached widgetsnbextension-4.0.11-py3-none-any.whl.metadata (1.6 kB)
Collecting jupyterlab-widgets~=3.0.11 (from ipywidgets->jupyter)
  Using cached jupyterlab_widgets-3.0.11-py3-none-any.whl.metadata (4.1 kB)
Requirement already satisfied: prompt-toolkit>=3.0.30 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from jupyter-console->jupyter) (3.0.47)
Requirement already satisfied: pygments in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from jupyter-console->jupyter) (2.18.0)
Collecting beautifulsoup4 (from nbconvert->jupyter)
  Using cached beautifulsoup4-4.12.3-py3-none-any.whl.metadata (3.8 kB)
Collecting bleach!=5.0.0 (from nbconvert->jupyter)
  Using cached bleach-6.1.0-py3-none-any.whl.metadata (30 kB)
Collecting defusedxml (from nbconvert->jupyter)
  Using cached defusedxml-0.7.1-py2.py3-none-any.whl.metadata (32 kB)
Collecting jinja2>=3.0 (from nbconvert->jupyter)
  Using cached jinja2-3.1.4-py3-none-any.whl.metadata (2.6 kB)
Collecting jupyterlab-pygments (from nbconvert->jupyter)
  Using cached jupyterlab_pygments-0.3.0-py3-none-any.whl.metadata (4.4 kB)
Collecting markupsafe>=2.0 (from nbconvert->jupyter)
  Using cached MarkupSafe-2.1.5-cp310-cp310-macosx_10_9_x86_64.whl.metadata (3.0 kB)
Collecting mistune<4,>=2.0.3 (from nbconvert->jupyter)
  Using cached mistune-3.0.2-py3-none-any.whl.metadata (1.7 kB)
Collecting nbclient>=0.5.0 (from nbconvert->jupyter)
  Using cached nbclient-0.10.0-py3-none-any.whl.metadata (7.8 kB)
Collecting nbformat>=5.7 (from nbconvert->jupyter)
  Using cached nbformat-5.10.4-py3-none-any.whl.metadata (3.6 kB)
Collecting pandocfilters>=1.4.1 (from nbconvert->jupyter)
  Using cached pandocfilters-1.5.1-py2.py3-none-any.whl.metadata (9.0 kB)
Collecting tinycss2 (from nbconvert->jupyter)
  Using cached tinycss2-1.3.0-py3-none-any.whl.metadata (3.0 kB)
Collecting jupyter-server<3,>=2.4.0 (from notebook->jupyter)
  Using cached jupyter_server-2.14.1-py3-none-any.whl.metadata (8.4 kB)
Collecting jupyterlab-server<3,>=2.27.1 (from notebook->jupyter)
  Using cached jupyterlab_server-2.27.2-py3-none-any.whl.metadata (5.9 kB)
Collecting jupyterlab<4.3,>=4.2.0 (from notebook->jupyter)
  Using cached jupyterlab-4.2.3-py3-none-any.whl.metadata (16 kB)
Collecting notebook-shim<0.3,>=0.2 (from notebook->jupyter)
  Using cached notebook_shim-0.2.4-py3-none-any.whl.metadata (4.0 kB)
Collecting qtpy>=2.4.0 (from qtconsole->jupyter)
  Using cached QtPy-2.4.1-py3-none-any.whl.metadata (12 kB)
Requirement already satisfied: six>=1.9.0 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from bleach!=5.0.0->nbconvert->jupyter) (1.16.0)
Collecting webencodings (from bleach!=5.0.0->nbconvert->jupyter)
  Using cached webencodings-0.5.1-py2.py3-none-any.whl.metadata (2.1 kB)
Requirement already satisfied: decorator in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipython>=7.23.1->ipykernel->jupyter) (5.1.1)
Requirement already satisfied: jedi>=0.16 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipython>=7.23.1->ipykernel->jupyter) (0.19.1)
Requirement already satisfied: stack-data in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipython>=7.23.1->ipykernel->jupyter) (0.6.3)
Requirement already satisfied: exceptiongroup in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipython>=7.23.1->ipykernel->jupyter) (1.2.1)
Requirement already satisfied: typing-extensions>=4.6 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipython>=7.23.1->ipykernel->jupyter) (4.12.2)
Requirement already satisfied: pexpect>4.3 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from ipython>=7.23.1->ipykernel->jupyter) (4.9.0)
Requirement already satisfied: python-dateutil>=2.8.2 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from jupyter-client>=6.1.12->ipykernel->jupyter) (2.9.0.post0)
Requirement already satisfied: platformdirs>=2.5 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from jupyter-core!=5.0.*,>=4.12->ipykernel->jupyter) (4.2.2)
Collecting anyio>=3.1.0 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached anyio-4.4.0-py3-none-any.whl.metadata (4.6 kB)
Collecting argon2-cffi>=21.1 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached argon2_cffi-23.1.0-py3-none-any.whl.metadata (5.2 kB)
Collecting jupyter-events>=0.9.0 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached jupyter_events-0.10.0-py3-none-any.whl.metadata (5.9 kB)
Collecting jupyter-server-terminals>=0.4.4 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached jupyter_server_terminals-0.5.3-py3-none-any.whl.metadata (5.6 kB)
Collecting overrides>=5.0 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached overrides-7.7.0-py3-none-any.whl.metadata (5.8 kB)
Collecting prometheus-client>=0.9 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached prometheus_client-0.20.0-py3-none-any.whl.metadata (1.8 kB)
Collecting send2trash>=1.8.2 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached Send2Trash-1.8.3-py3-none-any.whl.metadata (4.0 kB)
Collecting terminado>=0.8.3 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached terminado-0.18.1-py3-none-any.whl.metadata (5.8 kB)
Collecting websocket-client>=1.7 (from jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached websocket_client-1.8.0-py3-none-any.whl.metadata (8.0 kB)
Collecting async-lru>=1.0.0 (from jupyterlab<4.3,>=4.2.0->notebook->jupyter)
  Using cached async_lru-2.0.4-py3-none-any.whl.metadata (4.5 kB)
Collecting httpx>=0.25.0 (from jupyterlab<4.3,>=4.2.0->notebook->jupyter)
  Using cached httpx-0.27.0-py3-none-any.whl.metadata (7.2 kB)
Collecting jupyter-lsp>=2.0.0 (from jupyterlab<4.3,>=4.2.0->notebook->jupyter)
  Using cached jupyter_lsp-2.2.5-py3-none-any.whl.metadata (1.8 kB)
Requirement already satisfied: setuptools>=40.1.0 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from jupyterlab<4.3,>=4.2.0->notebook->jupyter) (70.3.0)
Collecting tomli>=1.2.2 (from jupyterlab<4.3,>=4.2.0->notebook->jupyter)
  Using cached tomli-2.0.1-py3-none-any.whl.metadata (8.9 kB)
Collecting babel>=2.10 (from jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Using cached Babel-2.15.0-py3-none-any.whl.metadata (1.5 kB)
Collecting json5>=0.9.0 (from jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Using cached json5-0.9.25-py3-none-any.whl.metadata (30 kB)
Collecting jsonschema>=4.18.0 (from jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Downloading jsonschema-4.23.0-py3-none-any.whl.metadata (7.9 kB)
Collecting requests>=2.31 (from jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Using cached requests-2.32.3-py3-none-any.whl.metadata (4.6 kB)
Collecting fastjsonschema>=2.15 (from nbformat>=5.7->nbconvert->jupyter)
  Using cached fastjsonschema-2.20.0-py3-none-any.whl.metadata (2.1 kB)
Requirement already satisfied: wcwidth in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from prompt-toolkit>=3.0.30->jupyter-console->jupyter) (0.2.13)
Collecting soupsieve>1.2 (from beautifulsoup4->nbconvert->jupyter)
  Using cached soupsieve-2.5-py3-none-any.whl.metadata (4.7 kB)
Collecting idna>=2.8 (from anyio>=3.1.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached idna-3.7-py3-none-any.whl.metadata (9.9 kB)
Collecting sniffio>=1.1 (from anyio>=3.1.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached sniffio-1.3.1-py3-none-any.whl.metadata (3.9 kB)
Collecting argon2-cffi-bindings (from argon2-cffi>=21.1->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached argon2_cffi_bindings-21.2.0-cp38-abi3-macosx_10_9_universal2.whl.metadata (6.7 kB)
Collecting certifi (from httpx>=0.25.0->jupyterlab<4.3,>=4.2.0->notebook->jupyter)
  Downloading certifi-2024.7.4-py3-none-any.whl.metadata (2.2 kB)
Collecting httpcore==1.* (from httpx>=0.25.0->jupyterlab<4.3,>=4.2.0->notebook->jupyter)
  Using cached httpcore-1.0.5-py3-none-any.whl.metadata (20 kB)
Collecting h11<0.15,>=0.13 (from httpcore==1.*->httpx>=0.25.0->jupyterlab<4.3,>=4.2.0->notebook->jupyter)
  Using cached h11-0.14.0-py3-none-any.whl.metadata (8.2 kB)
Requirement already satisfied: parso<0.9.0,>=0.8.3 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from jedi>=0.16->ipython>=7.23.1->ipykernel->jupyter) (0.8.4)
Collecting attrs>=22.2.0 (from jsonschema>=4.18.0->jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Using cached attrs-23.2.0-py3-none-any.whl.metadata (9.5 kB)
Collecting jsonschema-specifications>=2023.03.6 (from jsonschema>=4.18.0->jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Using cached jsonschema_specifications-2023.12.1-py3-none-any.whl.metadata (3.0 kB)
Collecting referencing>=0.28.4 (from jsonschema>=4.18.0->jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Using cached referencing-0.35.1-py3-none-any.whl.metadata (2.8 kB)
Collecting rpds-py>=0.7.1 (from jsonschema>=4.18.0->jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Downloading rpds_py-0.19.0-cp310-cp310-macosx_10_12_x86_64.whl.metadata (4.1 kB)
Collecting python-json-logger>=2.0.4 (from jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached python_json_logger-2.0.7-py3-none-any.whl.metadata (6.5 kB)
Collecting pyyaml>=5.3 (from jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached PyYAML-6.0.1-cp310-cp310-macosx_10_9_x86_64.whl.metadata (2.1 kB)
Collecting rfc3339-validator (from jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached rfc3339_validator-0.1.4-py2.py3-none-any.whl.metadata (1.5 kB)
Collecting rfc3986-validator>=0.1.1 (from jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached rfc3986_validator-0.1.1-py2.py3-none-any.whl.metadata (1.7 kB)
Requirement already satisfied: ptyprocess>=0.5 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from pexpect>4.3->ipython>=7.23.1->ipykernel->jupyter) (0.7.0)
Collecting charset-normalizer<4,>=2 (from requests>=2.31->jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Using cached charset_normalizer-3.3.2-cp310-cp310-macosx_10_9_x86_64.whl.metadata (33 kB)
Collecting urllib3<3,>=1.21.1 (from requests>=2.31->jupyterlab-server<3,>=2.27.1->notebook->jupyter)
  Using cached urllib3-2.2.2-py3-none-any.whl.metadata (6.4 kB)
Requirement already satisfied: executing>=1.2.0 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from stack-data->ipython>=7.23.1->ipykernel->jupyter) (2.0.1)
Requirement already satisfied: asttokens>=2.1.0 in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from stack-data->ipython>=7.23.1->ipykernel->jupyter) (2.4.1)
Requirement already satisfied: pure-eval in /Users/damir/.pyenv/versions/3.10.12/envs/ray-whisper/lib/python3.10/site-packages (from stack-data->ipython>=7.23.1->ipykernel->jupyter) (0.2.2)
Collecting fqdn (from jsonschema[format-nongpl]>=4.18.0->jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached fqdn-1.5.1-py3-none-any.whl.metadata (1.4 kB)
Collecting isoduration (from jsonschema[format-nongpl]>=4.18.0->jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached isoduration-20.11.0-py3-none-any.whl.metadata (5.7 kB)
Collecting jsonpointer>1.13 (from jsonschema[format-nongpl]>=4.18.0->jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached jsonpointer-3.0.0-py2.py3-none-any.whl.metadata (2.3 kB)
Collecting uri-template (from jsonschema[format-nongpl]>=4.18.0->jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached uri_template-1.3.0-py3-none-any.whl.metadata (8.8 kB)
Collecting webcolors>=24.6.0 (from jsonschema[format-nongpl]>=4.18.0->jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached webcolors-24.6.0-py3-none-any.whl.metadata (2.6 kB)
Collecting cffi>=1.0.1 (from argon2-cffi-bindings->argon2-cffi>=21.1->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached cffi-1.16.0-cp310-cp310-macosx_10_9_x86_64.whl.metadata (1.5 kB)
Collecting pycparser (from cffi>=1.0.1->argon2-cffi-bindings->argon2-cffi>=21.1->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached pycparser-2.22-py3-none-any.whl.metadata (943 bytes)
Collecting arrow>=0.15.0 (from isoduration->jsonschema[format-nongpl]>=4.18.0->jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached arrow-1.3.0-py3-none-any.whl.metadata (7.5 kB)
Collecting types-python-dateutil>=2.8.10 (from arrow>=0.15.0->isoduration->jsonschema[format-nongpl]>=4.18.0->jupyter-events>=0.9.0->jupyter-server<3,>=2.4.0->notebook->jupyter)
  Using cached types_python_dateutil-2.9.0.20240316-py3-none-any.whl.metadata (1.8 kB)
Using cached jupyter-1.0.0-py2.py3-none-any.whl (2.7 kB)
Using cached ipywidgets-8.1.3-py3-none-any.whl (139 kB)
Using cached jupyter_console-6.6.3-py3-none-any.whl (24 kB)
Using cached nbconvert-7.16.4-py3-none-any.whl (257 kB)
Using cached notebook-7.2.1-py3-none-any.whl (5.0 MB)
Using cached qtconsole-5.5.2-py3-none-any.whl (123 kB)
Using cached bleach-6.1.0-py3-none-any.whl (162 kB)
Using cached jinja2-3.1.4-py3-none-any.whl (133 kB)
Using cached jupyter_server-2.14.1-py3-none-any.whl (383 kB)
Using cached jupyterlab-4.2.3-py3-none-any.whl (11.6 MB)
Using cached jupyterlab_server-2.27.2-py3-none-any.whl (59 kB)
Using cached jupyterlab_widgets-3.0.11-py3-none-any.whl (214 kB)
Using cached MarkupSafe-2.1.5-cp310-cp310-macosx_10_9_x86_64.whl (14 kB)
Using cached mistune-3.0.2-py3-none-any.whl (47 kB)
Using cached nbclient-0.10.0-py3-none-any.whl (25 kB)
Using cached nbformat-5.10.4-py3-none-any.whl (78 kB)
Using cached notebook_shim-0.2.4-py3-none-any.whl (13 kB)
Using cached pandocfilters-1.5.1-py2.py3-none-any.whl (8.7 kB)
Using cached QtPy-2.4.1-py3-none-any.whl (93 kB)
Using cached widgetsnbextension-4.0.11-py3-none-any.whl (2.3 MB)
Using cached beautifulsoup4-4.12.3-py3-none-any.whl (147 kB)
Using cached defusedxml-0.7.1-py2.py3-none-any.whl (25 kB)
Using cached jupyterlab_pygments-0.3.0-py3-none-any.whl (15 kB)
Using cached tinycss2-1.3.0-py3-none-any.whl (22 kB)
Using cached anyio-4.4.0-py3-none-any.whl (86 kB)
Using cached argon2_cffi-23.1.0-py3-none-any.whl (15 kB)
Using cached async_lru-2.0.4-py3-none-any.whl (6.1 kB)
Using cached Babel-2.15.0-py3-none-any.whl (9.6 MB)
Using cached fastjsonschema-2.20.0-py3-none-any.whl (23 kB)
Using cached httpx-0.27.0-py3-none-any.whl (75 kB)
Using cached httpcore-1.0.5-py3-none-any.whl (77 kB)
Using cached json5-0.9.25-py3-none-any.whl (30 kB)
Downloading jsonschema-4.23.0-py3-none-any.whl (88 kB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 88.5/88.5 kB 1.1 MB/s eta 0:00:00
Using cached jupyter_events-0.10.0-py3-none-any.whl (18 kB)
Using cached jupyter_lsp-2.2.5-py3-none-any.whl (69 kB)
Using cached jupyter_server_terminals-0.5.3-py3-none-any.whl (13 kB)
Using cached overrides-7.7.0-py3-none-any.whl (17 kB)
Using cached prometheus_client-0.20.0-py3-none-any.whl (54 kB)
Using cached requests-2.32.3-py3-none-any.whl (64 kB)
Using cached Send2Trash-1.8.3-py3-none-any.whl (18 kB)
Using cached soupsieve-2.5-py3-none-any.whl (36 kB)
Using cached terminado-0.18.1-py3-none-any.whl (14 kB)
Using cached tomli-2.0.1-py3-none-any.whl (12 kB)
Using cached webencodings-0.5.1-py2.py3-none-any.whl (11 kB)
Using cached websocket_client-1.8.0-py3-none-any.whl (58 kB)
Using cached attrs-23.2.0-py3-none-any.whl (60 kB)
Downloading certifi-2024.7.4-py3-none-any.whl (162 kB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 163.0/163.0 kB 1.1 MB/s eta 0:00:00
Using cached charset_normalizer-3.3.2-cp310-cp310-macosx_10_9_x86_64.whl (122 kB)
Using cached idna-3.7-py3-none-any.whl (66 kB)
Using cached jsonschema_specifications-2023.12.1-py3-none-any.whl (18 kB)
Using cached python_json_logger-2.0.7-py3-none-any.whl (8.1 kB)
Using cached PyYAML-6.0.1-cp310-cp310-macosx_10_9_x86_64.whl (189 kB)
Using cached referencing-0.35.1-py3-none-any.whl (26 kB)
Using cached rfc3986_validator-0.1.1-py2.py3-none-any.whl (4.2 kB)
Downloading rpds_py-0.19.0-cp310-cp310-macosx_10_12_x86_64.whl (317 kB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 317.8/317.8 kB 1.5 MB/s eta 0:00:00
Using cached sniffio-1.3.1-py3-none-any.whl (10 kB)
Using cached urllib3-2.2.2-py3-none-any.whl (121 kB)
Using cached argon2_cffi_bindings-21.2.0-cp38-abi3-macosx_10_9_universal2.whl (53 kB)
Using cached rfc3339_validator-0.1.4-py2.py3-none-any.whl (3.5 kB)
Using cached cffi-1.16.0-cp310-cp310-macosx_10_9_x86_64.whl (182 kB)
Using cached h11-0.14.0-py3-none-any.whl (58 kB)
Using cached jsonpointer-3.0.0-py2.py3-none-any.whl (7.6 kB)
Using cached webcolors-24.6.0-py3-none-any.whl (14 kB)
Using cached fqdn-1.5.1-py3-none-any.whl (9.1 kB)
Using cached isoduration-20.11.0-py3-none-any.whl (11 kB)
Using cached uri_template-1.3.0-py3-none-any.whl (11 kB)
Using cached arrow-1.3.0-py3-none-any.whl (66 kB)
Using cached pycparser-2.22-py3-none-any.whl (117 kB)
Using cached types_python_dateutil-2.9.0.20240316-py3-none-any.whl (9.7 kB)
Installing collected packages: webencodings, fastjsonschema, widgetsnbextension, websocket-client, webcolors, urllib3, uri-template, types-python-dateutil, tomli, tinycss2, terminado, soupsieve, sniffio, send2trash, rpds-py, rfc3986-validator, rfc3339-validator, qtpy, pyyaml, python-json-logger, pycparser, prometheus-client, pandocfilters, overrides, mistune, markupsafe, jupyterlab-widgets, jupyterlab-pygments, jsonpointer, json5, idna, h11, fqdn, defusedxml, charset-normalizer, certifi, bleach, babel, attrs, async-lru, requests, referencing, jupyter-server-terminals, jinja2, httpcore, cffi, beautifulsoup4, arrow, anyio, jsonschema-specifications, isoduration, httpx, argon2-cffi-bindings, jsonschema, ipywidgets, argon2-cffi, qtconsole, nbformat, jupyter-console, nbclient, jupyter-events, nbconvert, jupyter-server, notebook-shim, jupyterlab-server, jupyter-lsp, jupyterlab, notebook, jupyter
Successfully installed anyio-4.4.0 argon2-cffi-23.1.0 argon2-cffi-bindings-21.2.0 arrow-1.3.0 async-lru-2.0.4 attrs-23.2.0 babel-2.15.0 beautifulsoup4-4.12.3 bleach-6.1.0 certifi-2024.7.4 cffi-1.16.0 charset-normalizer-3.3.2 defusedxml-0.7.1 fastjsonschema-2.20.0 fqdn-1.5.1 h11-0.14.0 httpcore-1.0.5 httpx-0.27.0 idna-3.7 ipywidgets-8.1.3 isoduration-20.11.0 jinja2-3.1.4 json5-0.9.25 jsonpointer-3.0.0 jsonschema-4.23.0 jsonschema-specifications-2023.12.1 jupyter-1.0.0 jupyter-console-6.6.3 jupyter-events-0.10.0 jupyter-lsp-2.2.5 jupyter-server-2.14.1 jupyter-server-terminals-0.5.3 jupyterlab-4.2.3 jupyterlab-pygments-0.3.0 jupyterlab-server-2.27.2 jupyterlab-widgets-3.0.11 markupsafe-2.1.5 mistune-3.0.2 nbclient-0.10.0 nbconvert-7.16.4 nbformat-5.10.4 notebook-7.2.1 notebook-shim-0.2.4 overrides-7.7.0 pandocfilters-1.5.1 prometheus-client-0.20.0 pycparser-2.22 python-json-logger-2.0.7 pyyaml-6.0.1 qtconsole-5.5.2 qtpy-2.4.1 referencing-0.35.1 requests-2.32.3 rfc3339-validator-0.1.4 rfc3986-validator-0.1.1 rpds-py-0.19.0 send2trash-1.8.3 sniffio-1.3.1 soupsieve-2.5 terminado-0.18.1 tinycss2-1.3.0 tomli-2.0.1 types-python-dateutil-2.9.0.20240316 uri-template-1.3.0 urllib3-2.2.2 webcolors-24.6.0 webencodings-0.5.1 websocket-client-1.8.0 widgetsnbextension-4.0.11
```
{collapsible="true" collapsed-title="pip install -U jupyter"}

