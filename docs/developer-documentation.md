# Developer documentation

## Understanding the components that make up Anthias

Here is a high-level overview of the different components that make Anthias:

![Anthias Diagram Overview](/docs/d2/anthias-diagram-overview.svg)

These components and their dependencies are mostly installed and handled with Ansible and Docker.

* The **NGINX** component (`anthias-nginx`) forwards requests to the backend and serves static files. It also acts as a reverse proxy.
* The **viewer** (`anthias-viewer`) is what drives the screen (e.g., shows web page, image or video).
* The **web app** component (`anthias-server`) &mdash; which consists of the front-end and back-end code &ndash; is what the user interacts with via browser.
* The **Celery** (`anthias-celery`) component is for aynschronouslt queueing and executing tasks outside the HTTP request-response cycle (e.g., doing assets cleanup).
* The **WebSocket** (`anthias-websocket`) component is used for forwarding requests from NGINX to the backend.
* **Redis** (`redis`) is used as a database, cache and message broker.
* The **database** component uses **SQLite** for storing the assets information.

## Dockerized development environment

To simplify development of the server module of Anthias, we've created a Docker container. This is intended to run on your local machine with the Anthias repository mounted as a volume.

> [!IMPORTANT]
> Anthias is using Docker's [buildx](https://docs.docker.com/engine/reference/commandline/buildx/) for the image builds. This is used both for cross compilation as well as for local caching. You might need to run `docker buildx create --use` first.

Assuming you're in the source code repository, simply run:

```bash
$ ./bin/start_development_server.sh

# The console output was truncated for brevity.
# ...

[+] Running 6/6
 ✔ Network anthias_default                Created                            0.1s
 ✔ Container anthias-redis-1              Started                            0.2s
 ✔ Container anthias-anthias-server-1     Started                            0.2s
 ✔ Container anthias-anthias-celery-1     Started                            0.3s
 ✔ Container anthias-anthias-websocket-1  Started                            0.4s
 ✔ Container anthias-anthias-nginx-1      Started                            0.5s
```

Running the command above will start the development server and you should be able to
access the web interface at `http://localhost:8000`.

To stop the development server, run the following:

```bash
docker compose -f docker-compose.dev.yml down
```

## Building containers locally

### Dependencies

#### `buildx`

> [!IMPORTANT]
> Make sure that you have `buildx` installed and that you have run
> `docker buildx create --use` before you run the image build script.

#### Poetry

We have switched from using Bash to [Poetry](https://python-poetry.org/) for building the Docker images.
Make sure that you have Python 3.11 and Poetry installed on your machine.

> [!TIP]
> You can install Poetry by running the following script:
> ```bash
> ./bin/install_poetry.sh
> ```
>
> After running the script, you can add the following to your `~/.bashrc` or similar file:
>
> ```bash
> # Add `pyenv` to the load path.
> export PYENV_ROOT="$HOME/.pyenv"
> [[ -d $PYENV_ROOT/bin ]] && \
>     export PATH="$PYENV_ROOT/bin:$PATH"
> eval "$(pyenv init -)"
>
> # Add `poetry to the load path.
> export PATH="$HOME/.local/bin:$PATH"
>
> poetry install --only=docker-image-builder
> ```
>
> You can either restart your terminal or run `source ~/.bashrc` to start using Poetry.

### Building only specific services

Say that you would only like to build the `anthias-server` and `anthias-viewer`
services. Just run the following:

```bash
$ poetry run python tools/image_builder \
    --service anthias-server \
    --service anthias-viewer
```

### Generating only Dockerfiles

If you'd like to just generate the Dockerfiles from the templates provided
inside the `docker/` directory, run the following:

```bash
$ poetry run python tools/image_builder \
  --dockerfiles-only
```

### Disabling cache mounts

If you'd like to disable cache mounts in the generated Dockerfiles, run the
following:

```bash
$ poetry run python tools/image_builder \
  --disable-cache-mounts
```

## Testing
### Running the unit tests

Build and start the containers.

```bash
$ poetry run python tools/image_builder \
  --dockerfiles-only \
  --disable-cache-mounts \
  --service celery \
  --service redis \
  --service test
$ docker compose \
    -f docker-compose.test.yml up -d --build
```

Run the unit tests.

```bash
$ docker compose \
    -f docker-compose.test.yml \
    exec anthias-test bash ./bin/prepare_test_environment.sh -s

# Integration and non-integration tests should be run separately as the
# former doesn't run as expected when run together with the latter.

$ docker compose \
    -f docker-compose.test.yml \
    exec anthias-test ./manage.py test --exclude-tag=integration

$ docker compose \
    -f docker-compose.test.yml \
    exec anthias-test ./manage.py test --tag=integration
```

### The QA checklist

We've also provided a [checklist](/docs/qa-checklist.md) that can serve as a guide for testing Anthias manually.

## Generating CSS and JS files

To get started, you need to start the development server first. See this [section](#dockerized-development-environment)
for details.

### Installing Node.js dependencies

Run the following command from the project root directory.

```bash
$ docker compose -f docker-compose.dev.yml exec anthias-server \
    npm install
```

### Transpiling CSS from SASS

Open a new terminal session and run the following command:

```bash
$ docker compose -f docker-compose.dev.yml exec anthias-server \
    npm run sass-dev
```

### Transpiling JS from CoffeeScript

Open a new terminal session and run the following command:

```bash
$ docker compose -f docker-compose.dev.yml exec anthias-server \
    npm run coffee-dev
```

### Closing the transpiler

Just press `Ctrl-C` to close the SASS and CoffeeScript transpilers.

## Linting Python code locally

The project uses `flake8` for linting the Python codebase. While the linter is being run on the CI/CD pipeline,
you can also run it locally. There are several ways to do this.

### Run the linter using `act`

[`act`](https://nektosact.com/) lets you run GitHub Actions locally. This is useful for testing the CI/CD pipeline locally.
Installation instructions can be found [here](https://nektosact.com/installation/index.html).

After installing and setting up `act`, run the following command:

```bash
$ act -W .github/workflows/python-lint.yaml
```

The command above will run the linter on the all the Python files in the repository. If you want to run the linter
on a specific file, you can try the commands in the next section.

### Running the linter using Poetry

You have to install Poetry first. You can find the installation instructions
[here](https://python-poetry.org/docs/#installing-with-the-official-installer).

After installing Poetry, run the following commands:

```bash
# Install the dependencies
$ poetry install --only=dev-host
$ poetry run flake8 $(git ls-files '*.py')
```

To run the linter on a specific file, run the following command:

```bash
$ poetry run flake8 path/to/file.py
```


## Managing releases
### Creating a new release

Check what the latest release is:

```bash
$ git pull
$ git tag

# Running the `git tag` command should output something like this:
# 0.16
# ...
# v0.18.6
```

Create a new release:

```bash
$ git tag -a v0.18.7 -m "Test new automated disk images"
```

Push release:
```bash
$ git push origin v0.18.7
```

### Delete a broken release

```bash
$ git tag -d v0.18.5                         [±master ✓]
Deleted tag 'v0.18.5' (was 9b86c39)

$ git push --delete origin v0.18.5           [±master ✓]
```

## Directories and files explained

In this section, we'll explain the different directories and files that are
present in a Raspberry Pi with Anthias installed.

### `home/${USER}/screenly/`

* All of the files and folders from the Github repo should be cloned into this directory.

### `/home/${USER}/.screenly/`

* `default_assets.yml` &mdash; configuration file which contains the default assets that get added to the assets list if enabled
* `initialized` &mdash; tells whether access point service (for Wi-Fi connectivity) runs or not
* `screenly.conf` &mdash; configuration file for web interface settings
* `screenly.db` &ndash; database file containing current assets information.


### `/etc/systemd/system/`

* `wifi-connect.service` &mdash; starts the Balena `wifi-connect` program to dynamically set the Wi-Fi config on the device via the captive portal
* `anthias-host-agent.service` &mdash; starts the Python script `host_agent.py`, which subscribes from the Redis component and performs a system call to shutdown or reboot the device when the message is received.

### `/etc/sudoers.d/screenly_overrides`

* `sudoers` configuration file that allows pi user to execute certain `sudo` commands without being a superuser (i.e., `root`)

### `/usr/share/plymouth/themes/anthias`

* `anthias.plymouth` &mdash; Plymouth config file (sets module name, `ImageDir` and `ScriptFile` dir)
* `anthias.script` &ndash; plymouth script file that loads and scales the splash screen image during the boot process
* `splashscreen.png` &mdash; the spash screen image that is displayed during the boot process

## Debugging the Anthias WebView

```
export QT_LOGGING_DEBUG=1
export QT_LOGGING_RULES="*.debug=true"
export QT_QPA_EGLFS_DEBUG=1
```

The Anthias WebView is a custom-built web browser based on the [Qt](https://www.qt.io/) toolkit framework.
The browser is assembled with a Dockerfile and built by a `webview/build_qt#.sh` script.

For further info on these files and more, visit the following link: [https://github.com/Screenly/Anthias/tree/master/webview](https://github.com/Screenly/Anthias/tree/master/webview)
