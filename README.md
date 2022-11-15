<p align="center">
  <img src="https://hsto.org/webt/mn/fz/q-/mnfzq-lgdnbmv-3xv-1qm6gn82e.png" alt="Logo" width=128" />
</p>

# WebHook Tester

[![Release version][badge_release_version]][link_releases]
![Project language][badge_language]
[![Build Status][badge_build]][link_build]
[![Release Status][badge_release]][link_build]
[![Coverage][badge_coverage]][link_coverage]
[![Image size][badge_size_latest]][link_docker_hub]
[![License][badge_license]][link_license]

This application allows you to test and debug Webhooks and HTTP requests using unique (random) URLs. You can customize the response code, `content-type` HTTP header, response content and set some delay for the HTTP responses. The main idea is viewed [here](https://github.com/fredsted/webhook.site).

<p align="center">
  <img src="/assets/app-animated.png?raw=true" alt="screenshot" width="925" />
</p>

This application is written in GoLang and works very fast. It comes with a tiny UI (written in `Vue.js`), which is built in the binary file, so you don't need any additional assets for the application using. Websockets are also used for incoming webhook notifications in the UI - you don't need any 3rd party solutions (like `pusher.com`) for this!

### :fire: Features list

- Liveness/readiness probes (routes `/live` and `/ready` respectively)
- Can be started without any 3rd party dependencies
- Metrics in prometheus format (route `/metrics`)
- Built-in tiny and fast UI, based on `vue.js`
- Multi-arch docker image, based on `scratch`
- Unprivileged user in docker image is used
- Well-tested and documented source code
- Built-in CLI health check sub-command
- Recorded request binary view using UI
- JSON/human-readable logging formats
- Customizable webhook responses
- Built-in Websockets support
- Low memory/cpu usage
- Free and open-source
- Ready to scale

### Storage

At the moment 2 types of data storage are supported - **memory** and **redis server** (flag `--storage-driver`).

The **memory** driver is useful for fast local debugging when recorded requests will not be needed after the app stops. The **Redis driver**, on the contrary, stores all the data on the redis server, and the data will not be lost after the app restarts. When running multiple app instances (behind the load balancer), it is also necessary to use the redis driver.

### Pub/sub

Publishing/subscribing are used to send notifications using WebSockets, and it also supports 2 types of driver - **memory** and **redis server** (flag `--pubsub-driver`).

For multiple app instances redis driver must be used.

## Installing

Download the latest binary file for your os/arch from the [releases page][link_releases] or use our [docker image][link_ghcr] ([hub.docker.com][link_docker_hub]).

## Usage

This application supports the following sub-commands:

| Sub-command   | Description                                                        |
|---------------|--------------------------------------------------------------------|
| `serve`       | Start HTTP server                                                  |
| `healthcheck` | Health checker for the HTTP server (use case - docker healthcheck) |

And global flags:

| Flag              | Description                 |
|-------------------|-----------------------------|
| `--version`, `-v` | Display application version |
| `--verbose`       | Verbose output              |
| `--debug`         | Debug output                |
| `--log-json`      | Logs in JSON format         |

### HTTP server starting

`serve` sub-command allows to use next flags:

| Flag                      | Description                                                                                        | Default value              | Environment variable |
|---------------------------|----------------------------------------------------------------------------------------------------|----------------------------|----------------------|
| `--listen`, `-l`          | IP address to listen on                                                                            | `0.0.0.0` (all interfaces) | `LISTEN_ADDR`        |
| `--port`, `-p`            | TCP port number                                                                                    | `8080`                     | `LISTEN_PORT`        |
| `--create-session`        | Crete a session on server startup with this UUID (example: `00000000-0000-0000-0000-000000000000`) |                            | `CREATE_SESSION`     |
| `--storage-driver`        | Storage engine (`memory` or `redis`)                                                               | `memory`                   | `STORAGE_DRIVER`     |
| `--pubsub-driver`         | Pub/Sub engine (`memory` or `redis`)                                                               | `memory`                   | `PUBSUB_DRIVER`      |
| `--redis-dsn`             | Redis server DSN (required if storage or pub/sub driver is `redis`)                                | `redis://127.0.0.1:6379/0` | `REDIS_DSN`          |
| `--ignore-header-prefix`  | Ignore incoming webhook header prefix (case insensitive; example: `X-Forwarded-`)                  | `[]`                       |                      |
| `--max-request-body-size` | Maximal webhook request body size (in bytes; `0` = unlimited)                                      | `65536`                    |                      |
| `--max-requests`          | Maximum stored requests per session (max `65535`)                                                  | `128`                      | `MAX_REQUESTS`       |
| `--session-ttl`           | Session lifetime (examples: `48h`, `1h30m`)                                                        | `168h`                     | `SESSION_TTL`        |
| `--ws-max-clients`        | Maximal websocket clients (`0` = unlimited)                                                        | `0`                        | `WS_MAX_CLIENTS`     |
| `--ws-max-lifetime`       | Maximal single websocket lifetime (examples: `3h`, `1h30m`; `0` = unlimited)                       | `0`                        | `WS_MAX_LIFETIME`    |

> Redis DSN format: `redis://<user>:<password>@<host>:<port>/<db_number>`

Server starting command example:

```shell
$ ./webhook-tester serve \
    --port 8080
    --storage-driver redis
    --pubsub-driver redis
    --redis-dsn redis://redis-host:6379/0
    --max-requests 512
    --ignore-header-prefix X-Forwarded-
    --ignore-header-prefix X-Reverse-Proxy-
    --ws-max-clients 30000
    --ws-max-lifetime 6h
```

After that you can navigate your browser to `http://127.0.0.1:8080/` try to send your first HTTP request for the webhook-tester!

### Using docker

| Registry                               | Image                                |
|----------------------------------------|--------------------------------------|
| [GitHub Container Registry][link_ghcr] | `ghcr.io/tarampampam/webhook-tester` |
| [Docker Hub][link_docker_hub]          | `tarampampam/webhook-tester`         |

> Using the `latest` tag for the docker image is highly discouraged because of possible backward-incompatible changes during **major** upgrades. Please, use tags in `X.Y.Z` format

Just execute in your terminal:

```shell
$ docker run --rm -p 8080:8080/tcp tarampampam/webhook-tester serve
```

Where `X.X.X` is image tag _(application version)_.

#### Docker-compose

For running this app using docker-compose and if you want to keep the data after restarts, you can use the following example with a Redis server as a backend for the data:

```yaml
version: '3.8'

volumes:
  redis-data: {}

services:
  webhook-tester:
    image: tarampampam/webhook-tester
    command: --log-json serve --port 8080 --storage-driver redis --pubsub-driver redis --redis-dsn redis://redis:6379/0
    ports: ['8080:8080/tcp'] # Open <http://127.0.0.1:8080>
    depends_on:
      redis: {condition: service_healthy}

  redis:
    image: redis:7-alpine
    volumes: [redis-data:/data:rw]
    ports: ['6379/tcp']
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 1s
```

Or you can use in-memory data storage only:

```yaml
version: '3.8'

services:
  webhook-tester:
    image: tarampampam/webhook-tester
    command: serve --port 8080 --create-session 00000000-0000-0000-0000-000000000000
    ports: ['8080:8080/tcp'] # Open <http://127.0.0.1:8080/#/00000000-0000-0000-0000-000000000000>
```

## Changes log

[![Release date][badge_release_date]][link_releases]
[![Commits since latest release][badge_commits_since_release]][link_commits]

Changes log can be [found here][link_changes_log].

## Releasing

New versions publishing is very simple - just "publish" new release using repo releases page.

> Release version _(and git tag, of course)_ MUST starts with `v` prefix (eg.: `v0.0.1` or `v1.2.3-RC1`)

## Support

[![Issues][badge_issues]][link_issues]
[![Issues][badge_pulls]][link_pulls]

If you find any package errors, please, [make an issue][link_create_issue] in current repository.

## License

This is open-sourced software licensed under the [MIT License][link_license].

[badge_build]:https://img.shields.io/github/workflow/status/tarampampam/webhook-tester/tests?maxAge=30&logo=github
[badge_release]:https://img.shields.io/github/workflow/status/tarampampam/webhook-tester/release?maxAge=30&label=release&logo=github
[badge_coverage]:https://img.shields.io/codecov/c/github/tarampampam/webhook-tester/master.svg?maxAge=30
[badge_release_version]:https://img.shields.io/github/release/tarampampam/webhook-tester.svg?maxAge=30
[badge_size_latest]:https://img.shields.io/docker/image-size/tarampampam/webhook-tester/latest?maxAge=30
[badge_language]:https://img.shields.io/github/go-mod/go-version/tarampampam/webhook-tester?longCache=true
[badge_license]:https://img.shields.io/github/license/tarampampam/webhook-tester.svg?longCache=true
[badge_release_date]:https://img.shields.io/github/release-date/tarampampam/webhook-tester.svg?maxAge=180
[badge_commits_since_release]:https://img.shields.io/github/commits-since/tarampampam/webhook-tester/latest.svg?maxAge=45
[badge_issues]:https://img.shields.io/github/issues/tarampampam/webhook-tester.svg?maxAge=45
[badge_pulls]:https://img.shields.io/github/issues-pr/tarampampam/webhook-tester.svg?maxAge=45

[link_coverage]:https://codecov.io/gh/tarampampam/webhook-tester
[link_build]:https://github.com/tarampampam/webhook-tester/actions
[link_docker_hub]:https://hub.docker.com/r/tarampampam/webhook-tester/
[link_docker_tags]:https://hub.docker.com/r/tarampampam/webhook-tester/tags
[link_license]:https://github.com/tarampampam/webhook-tester/blob/master/LICENSE
[link_releases]:https://github.com/tarampampam/webhook-tester/releases
[link_commits]:https://github.com/tarampampam/webhook-tester/commits
[link_changes_log]:https://github.com/tarampampam/webhook-tester/blob/master/CHANGELOG.md
[link_issues]:https://github.com/tarampampam/webhook-tester/issues
[link_create_issue]:https://github.com/tarampampam/webhook-tester/issues/new/choose
[link_pulls]:https://github.com/tarampampam/webhook-tester/pulls
[link_ghcr]:https://github.com/users/tarampampam/packages/container/package/webhook-tester
