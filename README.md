# DNF (dockerization nano framework)

## Dockerization issues

- Multiple services inside one container. You need to use some wrapper/supervisord/etc.
- Environment variables for cronjobs. You need to pass them somehow.
- Shell script as entrypoint. You need to pass signals to its child processes.
- *Service X* ignores SIGTERM, but respects SIGINT. You need to convert signals.
- *Service X* consists of multiple processes, it does not fit *one process per container* philosophy. You need to run it somehow.
- *Service X* does not have a foreground mode. You need to keep container running.

**DNF is all you need**

## How it works

DNF consists of two scripts: `init` and `cron`.

### `init` script

You have to use `init` script as `ENTRYPOINT` or `CMD` for your container.

#### Startup

On startup `init` sources the `start` script. So you have to put all your startup commands to the `start` script.

You can start your services in any way you want **except the foreground mode**.

Example `start` script:
```
service foo start #init script
bar & #background mode
somedaemon #self-daemonizing process
```

**Bad** example:
```
foo --foreground #<-- do not do this
bar & #<-- never executed
```

**Good** example:
```
foo --foreground & #process goes to background
bar &
```

#### Shutdown

Docker uses signals to stop containers.

DNF `init` catches SIGTERM/SIGINT and sources the `stop` script.

If you want to stop container gracefully, put your shutdown commands to the `stop` script. If you don't care about graceful stop, you can leave the `stop` script empty.

`init` script automatically kills (SIGTERM) processes started in background mode.

Mongo container `stop` script example:
```
mongod --shutdown
```

Apache2 container `stop` script example:
```
apache2ctl graceful-stop
```

### `cron` script

DNF `cron` script passes environment variables to you cronjobs and replaces default crontab shell with BASH.

Put `SHELL=/path/to/dnf/cron` as first line in your crontab.

Example:
```
SHELL=/opt/dnf/cron

* * * * * /your/app &>> app.log
```

## Installation

Example `Dockerfile`:

```
FROM ubuntu:14.04

RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y \
    curl

#install dnf
RUN mkdir /opt/dnf
RUN curl -sL https://github.com/polonskiy/dnf/archive/0.1.0.tar.gz | tar -xzvf - --directory /opt/dnf --strip-components=1

#dnf startup/shutdown scripts
ADD dnf/start /opt/dnf/start
ADD dnf/stop /opt/dnf/stop

#add cronjobs
ADD cron/cronjobs /etc/cronjobs
RUN crontab /etc/cronjobs

#start dnf
CMD ["/opt/dnf/init"]
```
