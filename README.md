
# <img src="doc/images/t9_logo.png" height="25"> Tyzen9 - docker-lftp-mirror
This ia a container is excellent for mirroring the contents of a source server to a target server using an `ssh` tunnel and `lftp` to perform efficient file transfers through a secure connection. 

This container is specifically designed to PULL data from a source server, to a target server and mirror the contents of the source server.

> [!WARNING]
> This means that deleted files on the source server, will also be deleted on the target server as a result of the mirroring process.

LFTP is a sophisticated, command-line file transfer program designed for Unix and Unix-like operating systems, such as Linux. It supports a wide range of protocols, including FTP, FTPS, HTTP, HTTPS, and SFTP, making it a versatile tool for transferring files between local and remote systems. Unlike traditional FTP clients that rely on graphical interfaces, LFTP operates entirely through the command line, offering a lightweight and scriptable alternative.

Using LFTP has the following benefits:

- Parallel file transfers, which can significantly speed up operations when handling multiple files or large datasets
- Automatic resumption of interrupted transfers
- Retry mechanisms for non-fatal errors

## Supported Architectures
Simply pulling `tyzen9/lftp-mirror:latest` should retrieve the correct image for your arch. The architectures supported by this image are:

| Architecture | Available | Tag |
| :---   | :--- | :--- |
| x86-64 | ✅ | latest |
| arm64	 | ✅ | latest |

Specific version tags are available on [Docker Hub](https://hub.docker.com/repository/docker/tyzen9/lftp-mirror/tags).

# Deployment
The recommended means of deploying this container is through Docker compose.  The `compose.yml` file provides an excellent example of this. Below you will see an example of this:

```yaml
services:
  lftp-mirror:
    image: tyzen9/lftp-mirror:latest
    container_name: lftp-mirror
    environment:
      - TZ_ID=${TZ_ID}
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - SOURCE_HOSTNAME=${SOURCE_HOSTNAME}
      - SSH_USERNAME=${SSH_USERNAME}
      - SSH_PASSWORD=${SSH_PASSWORD}
      - SOURCE_DIR=${SOURCE_DIR}
      - SOURCE_EXCLUDES=${SOURCE_EXCLUDES}
    ports:
      - "${WEB_PORT:-8080}:8080"
    volumes:
      - ${LOCAL_TARGET_DIR}:/downloads
```

## Required Configuration Options
These environment variables MUST be defined:
| Variable | Type | Example | Definition |
| :---   | :--- | :--- | :--- |
| SOURCE_HOSTNAME | string | host.domain.com | The name of the server to connect with |
| SSH_USERNAME | string | \<username\> | The ssh username used to connect to the source server |
| SSH_PASSWORD | string | \<password\> | The ssh password used to  connect to the source server |
| SOURCE_DIR | string | \downloads | The path to the directory to mirror on the source server |

## Optional Configuration Options
These environment variables are optional, and could be used to adjust functionality:
| Variable | Type | Default | Definition |
| :---   | :--- | :--- | :--- |
| TZ_ID | string | UTC | Timezone to be running this container as [TZ IDs](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List) |
| PUID | number | 1000 | The linux user ID to run this process as |
| PGID | number | 1000 | The linux group ID to run this process as |
| UPDATE_INTERVAL | string | 300 | The number of seconds between LFTP initiated requests |
| SOURCE_EXCLUDES | string | "\<empty\>" | A comma separated lists of sources to exclude, [see here](https://www.cyberciti.biz/faq/lftp-command-mirror-x-exclude-files-sub-directory-syntax/) for details. (Example: temp/,freeleech/) |
| SSH_PORT | string | 22 | The ssh port used to  connect to the source server |
| LOG_LEVEL | string | INFO | Valid values: DEBUG, INFO, WARNING, ERROR, CRITICAL  |
| STALL_TIMEOUT | number | 900 | The number of seconds without lftp progress (no output and no growth of any in-flight file) before a sync is considered stalled and killed. Also used by the container's `HEALTHCHECK` to decide how stale the heartbeat is allowed to get before reporting unhealthy |
| WEB_PORT | number | 8080 | Port for the [status API](#status-api) (`/status`, `/events`) |
| HISTORY_FILE | string | /var/log/lftp-mirror/history.jsonl | Path to the JSON-lines start/download/remove history log read by `/events` |
| DISABLE_WEB_API | boolean | false | Set to `true` to disable the [status API](#status-api) entirely, rather than just leaving `WEB_PORT` unpublished |

## Healthcheck
The image defines a Docker `HEALTHCHECK` that runs `app/healthcheck.py` every 60 seconds. `main.py` touches a heartbeat file (`/tmp/heartbeat`) at the start of each sync cycle and whenever there is `lftp` progress (a line of output, or growth of an in-flight `.lftp` temp file); a watchdog thread kills the `lftp` process if there's no progress for `STALL_TIMEOUT` seconds, so a dead SFTP connection can't hang forever, while a single large file that's copying quietly isn't mistaken for a stall. The healthcheck reports unhealthy if the heartbeat file is missing or older than `UPDATE_INTERVAL + STALL_TIMEOUT` (plus a small grace period), which lets orchestrators (e.g. Docker Compose `restart: unless-stopped`, Kubernetes liveness probes) detect and restart a hung container.

## Status API
The container runs a small built-in HTTP server (no extra dependencies) for integrating sync status and live copy activity into external tools like Home Assistant, without needing to parse logs.

| Endpoint | Description |
| :--- | :--- |
| `GET /status` | Current cycle summary: last/next sync times, download/removed counts, success state, any error, and the files currently being copied (`active`) |
| `GET /events?limit=50` | Most recent start/download/remove events (newest first, max 1000), sourced from the rotating `HISTORY_FILE` JSON-lines log |

Example `/status` response:
```json
{
  "cycle": 42,
  "last_sync_start": 1754345600.1,
  "last_sync_end": 1754345612.4,
  "last_sync_success": true,
  "last_error": null,
  "downloaded": 3,
  "removed": 0,
  "next_sync": "2026-08-05T10:05:00",
  "active": [
    {"path": "movies/example.mkv", "subfolder": "movies", "started": "2026-08-05T10:00:03-04:00", "bytes": 734003200},
    {"path": "tv/show/s01e01.mkv", "subfolder": "tv", "started": "2026-08-05T10:00:04-04:00", "bytes": 52428800}
  ],
  "totals": {"start": 131, "download": 128, "remove": 4}
}
```

### Live copy activity
lftp writes each file under a temporary `<name>.lftp` name and renames it once the copy is complete (`xfer:use-temp-file` in `lftp.conf`). The container checks every in-flight file every 5 seconds:

- `active` lists the files being copied right now. `bytes` is how much of the file has landed locally so far. The total size isn't known, so there's no percentage. With `--use-pget-n=5` each file arrives in 5 chunks, so `bytes` grows unevenly.
- A file leaves `active` and gets a `download` event as soon as it's renamed into place, not when the whole sync cycle ends.
- `downloaded` is the number of files completed in the last cycle, including files that finished before a cycle failed.

### Events
| Event | Meaning |
| :--- | :--- |
| `start` | lftp began copying the file |
| `download` | The file is complete and in its final location |
| `remove` | The file or directory was deleted locally because it no longer exists on the source (`--delete`) |

If a cycle fails or is killed by the stall watchdog, files still in flight get no `download` event and are dropped from `active`. The next cycle resumes them (`--continue`) and records them as `start` again, so `totals.start` can be higher than `totals.download`.

Event timestamps are rendered in the zone configured by `TZ_ID` (falling back to UTC if unset or not a recognized [IANA zone name](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List)). History is recorded even when `DISABLE_WEB_API` is set.

Example `/events` response:
```json
{
  "events": [
    {"time": "2026-08-05T10:04:41-04:00", "event": "download", "subfolder": "movies", "path": "movies/example.mkv"},
    {"time": "2026-08-05T10:00:05-04:00", "event": "remove", "subfolder": "temp", "path": "temp/old.txt"},
    {"time": "2026-08-05T10:00:03-04:00", "event": "start", "subfolder": "movies", "path": "movies/example.mkv"}
  ]
}
```

### Home Assistant integration
Add a [RESTful sensor](https://www.home-assistant.io/integrations/rest/) to `configuration.yaml`, pointed at the container's published `WEB_PORT`:

```yaml
sensor:
  - platform: rest
    name: LFTP Mirror
    resource: http://<container-host>:8080/status
    value_template: "{{ 'ok' if value_json.last_sync_success else 'error' }}"
    json_attributes:
      - downloaded
      - removed
      - next_sync
      - last_error
      - totals
      - active
    scan_interval: 60
```

This exposes the last sync's outcome as the sensor state, with download/removed counts, the files currently being copied, and the next scheduled sync as attributes for automations or dashboard cards.

# Development
This project is designed to be developed with VS code and the [Dev Containers](https://marketplace.visualstudio.com/items/?itemName=ms-vscode-remote.remote-containers) extension. When testing LFTP transfers, the development environment is set to mirror a provided source server with the local `downloads` file.

> [!IMPORTANT]
> In development, a `.env` file is expected. You can copy `sample.env` to make a `.env` file for testing.

## Development Environment Requirements
- Docker Engine 
- Docker Desktop (optional)
- Make - used to build and publish images

## VS Code
The following extensions are recommended to be installed in VS Code:

- [Dev Containers](https://marketplace.visualstudio.com/items/?itemName=ms-vscode-remote.remote-containers)
- [Docker](https://marketplace.visualstudio.com/items/?itemName=ms-azuretools.vscode-docker)
- [Python](https://marketplace.visualstudio.com/items/?itemName=ms-python.python)

### Open the project in a Docker Dev Container for development using VS Code
1. Clone this repository and open the project folder in VS Code.
2. Install the recommended extensions listed above.
3. Ensure Docker Desktop (or another Docker service) is running on your system.
4. Copy `sample.env` to `.env` and fill in your values before opening the container:
    ```
    cp sample.env .env
    ```
5. Open the Command Palette (`Ctrl+Shift+P` or `Cmd+Shift+P`) and select:
    - `Dev Containers: Reopen in Container` — if the project folder is already open in VS Code
    - `Dev Containers: Open Folder in Container` — if opening for the first time
    - VS Code will build the container based on `.devcontainer/devcontainer.json`. The first build may take a few minutes; subsequent openings will be faster.
6. Once connected, you can use all of VS Code's features (e.g., IntelliSense, debugging) as if working locally.

> [!IMPORTANT]
> There is no hot reload configured. Each time you update `main.py`, run `Dev Containers: Rebuild Container` from the Command Palette for changes to take effect.

To start the project inside the dev container, run this in the VS Code terminal:

```
python3 /usr/src/tyzen9/main.py
```


## Build & Publish
Update the `Makefile` to contain the appropriate Docker Hub username, application name and version number

```
DOCKER_USERNAME ?= username
APPLICATION_NAME ?= application-name
VERSION ?= 1.0.0
```

To build images of this container, use this command in the root directory of the project:

```
make build
```

To publish the built images to to Docker Hub use this command in the root directory of the project:

To build use this command:
```
make push
```

# References
[Setting up a dockerized Python environment the elegant way](https://towardsdatascience.com/setting-a-dockerized-python-environment-the-elegant-way-f716ef85571d/)
