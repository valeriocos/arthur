# Arthur [![Build Status](https://travis-ci.org/grimoirelab/arthur.svg?branch=master)](https://travis-ci.org/grimoirelab/arthur)[![Coverage Status](https://img.shields.io/coveralls/grimoirelab/arthur.svg)](https://coveralls.io/r/grimoirelab/arthur?branch=master)

King Arthur commands his loyal knight Perceval on the quest to fetch
data from software repositories.

Arthur is a distributed job queue platform that schedules and executes
Perceval. The platform is composed by two components: `arthurd`, the server
that schedules the jobs and one or more instances of `arthurw`, the work horses
that will run each Perceval job.

The repositories whose data will be fetched are added to the
platform using a REST API. Then, the server transforms these repositories into
Perceval jobs and schedules them between its job queues.

Workers are waiting for new jobs checking these queues. Workers only execute
a job at a time. When a new job arrives, an idle worker will take and run
it. Once a job is finished, if the result is successful, the server will
re-schedule it to retrieve new data.

By default, items fetched by each job will be published using a Redis queue.


## Usage

### arthurd
```
usage: arthurd [-c <file>] [-g] [-h <host>] [-p <port>] [-d <database>]
               [--log-path <path>] [--archive-path <cpath>]
               [--no-archive] [--no-daemon] | --help

King Arthur commands his loyal knight Perceval on the quest
to retrieve data from software repositories.

This command runs an Arthur daemon that waits for HTTP requests
on port 8080. Repositories to analyze are added using an REST API.
Repositories are transformed into Perceval jobs that will be
scheduled and run using a distributed job queue platform.

optional arguments:
  -?, --help            show this help message and exit
  -c FILE, --config FILE
                        set configuration file
  -g, --debug           set debug mode on
  -h, --host            set the host name or IP address on which to listen for connections
  -p, --port            set listening TCP port (default: 8080)
  -d, --database        URL database connection (default: 'redis://localhost/8')
  -s, --sync            work in synchronous mode (without workers)
  --log-path            path where logs are stored
  --archive-path        path to archive manager directory
  --no-archive          do not archive fetched raw data
  --no-daemon           do not run arthur in daemon mode
```

#### arthurd configuration file

To run `arthurd` using a configuration file:

```
$ arthurd [-c <file>]
```

Where `<file>` is the path to an `ini` file which uses the same parameters as in command line, but replacing underscores by hyphens. This configuration file has the following structure:

```
[arthur]
archive_path=/tmp/.arthur/archive
debug=True
log_path=/tmp/logs/arthurd
no_archive=True
sync_mode=True

[connection]
host=127.0.0.1
port=8080

[elasticsearch]
es_index=http://localhost:9200/items

[redis]
database=redis://localhost/8
```

### arthurw
```
usage: arthurw [-g] [-d <database>] [--burst] [<queue1>...<queueN>] | --help

King Arthur's worker. It will run Perceval jobs on the quest
to retrieve data from software repositories.

positional arguments:
   queues               list of queues this worker will listen for
                        ('create' and 'update', by default)

optional arguments:
  -?, --help            show this help message and exit
  -g, --debug           set debug mode on
  -d, --database        URL database connection (default: 'redis://localhost/8')
  -b, --burst           Run in burst mode (quit after all work is done)
```

## Requirements

* Python >= 3.4
* Redis >= 2.3
* python3-dateutil >= 2.6
* python3-redis >= 2.10
* python3-rq >= 0.6
* python3-cherrypy >= 8.1.0
* grimoirelab-toolkit >= 0.1.0
* perceval >= 0.8

## Installation

```
$ pip3 install -r requirements.txt
$ python3 setup.py install
```

## How to run it

The first step is to run a Redis server that will be used for communicating
Arthur's components. Please refer to its documentation to know how to
install and run it.

To run Arthur server:
```
$ arthurd -g -d redis://localhost/8 --log-path /tmp/logs/arthud --no-archive
```

To run a worker:

```
$ arthurw -d redis://localhost/8
```

## Adding tasks

To add tasks to Arthur, create a JSON object containing the tasks needed
to fetch data from a set of repositories. Each task will run a Perceval
backend, thus, backend parameters will also needed for each task.

```
$ cat tasks.json
{
    "tasks": [
        {
            "task_id": "arthur.git",
            "backend": "git",
            "backend_args": {
                "gitpath": "/tmp/git/arthur.git/",
                "uri": "https://github.com/grimoirelab/arthur.git",
                "from_date": "2015-03-01"
            },
            "category": "commit",
            "scheduler": {
                "delay": 10
            }
        },
        {
            "task_id": "bugzilla_mozilla",
            "backend": "bugzillarest",
            "backend_args": {
                "url": "https://bugzilla.mozilla.org/",
                "from_date": "2016-09-19"
            },
            "category": "bug",
            "archive": {
                "fetch_from_archive": true,
                "archived_after": "2018-02-26 09:00"
            },
            "scheduler": {
                "delay": 60,
                "max_retries_job": 5
            }
        }
    ]
}
```

Then, send this JSON stream to the server calling `add` method.

```
$ curl -H "Content-Type: application/json" --data @tasks.json http://127.0.0.1:8080/add
```

## Listing tasks

The list of tasks currently scheduled can be obtained using the method `tasks`.

```
$ curl http://127.0.0.1:8080/tasks

{
    "tasks": [
        {
            "backend_args": {
                "from_date": "2015-03-01T00:00:00+00:00",
                "uri": "https://github.com/grimoirelab/arthur.git",
                "gitpath": "/tmp/santi/"
            },
            "backend": "git",
            "category": "commit",
            "created_on": 1480531707.810326,
            "task_id": "arthur.git",
            "scheduler": {
                "max_retries_job": 3,
                "delay": 10
            }
        }
    ]
}
```

## Removing tasks

Scheduled tasks can also be removed calling to the server using the `remove`
method. A JSON stream must be provided setting the identifiers of the
tasks to be removed.

```
$ cat tasks_to_remove.json

{
    "tasks": [
        {
            "task_id": "bugzilla_mozilla"
        },
        {
            "task_id": "arthur.git"
        }
    ]
}

$ curl -H "Content-Type: application/json" --data @tasks_to_remove.json http://127.0.0.1:8080/remove
```

## License

Licensed under GNU General Public License (GPL), version 3 or later.
