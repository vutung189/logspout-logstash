[![GoDoc](https://godoc.org/github.com/looplab/logspout-logstash?status.svg)](https://godoc.org/github.com/looplab/logspout-logstash)
[![Go Report Card](https://goreportcard.com/badge/looplab/logspout-logstash)](https://goreportcard.com/report/looplab/logspout-logstash)

# logspout-logstash

A minimalistic adapter for github.com/gliderlabs/logspout to write to Logstash

Follow the instructions in https://github.com/gliderlabs/logspout/tree/master/custom on how to build your own Logspout container with custom modules. Basically just copy the contents of the custom folder and include:

```go
package main

import (
  _ "github.com/looplab/logspout-logstash"
  _ "github.com/gliderlabs/logspout/transports/udp"
  _ "github.com/gliderlabs/logspout/transports/tcp"
)
```

in modules.go.

Use by setting a docker environment variable `ROUTE_URIS=logstash://host:port` to the Logstash server.
The default protocol is UDP, but it is possible to change to TCP by adding ```+tcp``` after the logstash protocol when starting your container.

```bash
docker run --name="logspout" \
    --volume=/var/run/docker.sock:/var/run/docker.sock \
    -e ROUTE_URIS=logstash+tcp://logstash.home.local:5000 \
    localhost/logspout-logstash:v3.1
```

In your logstash config, set the input codec to `json` e.g:

```bash
input {
  udp {
    port  => 5000
    codec => json
  }
  tcp {
    port  => 5000
    codec => json
  }
}
```

## Available configuration options

For example, to get into the Logstash event's @tags field, use the ```LOGSTASH_TAGS``` container environment variable. Multiple tags can be passed by using comma-separated values

```bash
  # Add any number of arbitrary tags to your event
  -e LOGSTASH_TAGS="docker,production"
```

The output into logstash should be like:

```json
    "tags": [
      "docker",
      "production"
    ],
```

You can also add arbitrary logstash fields to the event using the ```LOGSTASH_FIELDS``` container environment variable:

```bash
  # Add any number of arbitrary fields to your event
  -e LOGSTASH_FIELDS="myfield=something,anotherfield=something_else"
```

The output into logstash should be like:

```json
    "myfield": "something",
    "another_field": "something_else",
```

Both configuration options can be set for every individual container, or for the logspout-logstash
container itself where they then become a default for all containers if not overridden there.

By setting the environment variable DOCKER_LABELS to a non-empty value, logspout-logstash will add all docker container
labels as fields:
```json
    "docker": {
        "hostname": "866e2ca94f5f",
        "id": "866e2ca94f5fe11d57add5a78232c53dfb6187f04f6e150ec15f0ae1e1737731",
        "image": "centos:7",
        "labels": {
            "a_label": "yes",
            "build-date": "20161214",
            "license": "GPLv2",
            "name": "CentOS Base Image",
            "pleasework": "okay",
            "some_label_with_dots": "more.dots",
            "vendor": "CentOS"
        },
        "name": "/ecstatic_murdock"
```

To be compatible with Elasticsearch, dots in labels will be replaced with underscores.

By setting `INCLUDE_CONTAINERS` you can specify a comma separated list of container names to only get logs from those containers.  You can also set `INCLUDE_CONTAINERS_REGEX` to use regex to describe the containers to include.

### Using Logspout-Logstash in a swarm

In a swarm, logspout is best deployed as a global service. To support this mode of deployment, the logstash adapter will look for the file `/etc/host_hostname` and, if the file exists and it is not empty, will configure the hostname field with the content of this file. You can then use a volume mount to map a file on the docker hosts with the file `/etc/host_hostname` in the container. The sample compose file below illustrates how this can be done:

```
version: "3"
services:
  logspout:
    image: localhost/logspout-logstash:latest
    volumes:
      # Logspout reads this in and attaches it to the log
      - /etc/hostname:/etc/host_hostname:ro
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      # IP and port for logstash host
      - ROUTE_URIS=logstash://host:port
      # Include all docker labels
      - DOCKER_LABELS=true
      # Add environment field to all logs sent to logstash
      - LOGSTASH_FIELDS=environment=${NODE_ENV}
    deploy:
      mode: global
      resources:
        limits:
          cpus: '0.20'
          memory: 256M
        reservations:
          cpus: '0.10'
          memory: 128M
```

### Retrying

Two environment variables control the behaviour of Logspout when the Logstash target isn't available:
```RETRY_STARTUP``` causes Logspout to retry forever if Logstash isn't available at startup,
and ```RETRY_SEND``` will retry sending log lines when Logstash becomes unavailable while Logspout is running.
Note that ```RETRY_SEND``` will work only
if UDP is used for the log transport and the destination doesn't change;
in any other case ```RETRY_SEND``` should be disabled, restart and reconnect instead
and let ```RETRY_STARTUP``` deal with the situation.
With both retry options, log lines will be lost when Logstash isn't available. Set the
environment variables to any nonempty value to enable retrying. The default is disabled.


This table shows all available configurations:

| Environment Variable | Input Type | Default Value |
|----------------------|------------|---------------|
| LOGSTASH_TAGS        | array      | None          |
| LOGSTASH_FIELDS      | map        | None          |
| INCLUDE_CONTAINERS   | array      | None          |
| DOCKER_LABELS        | any        | ""            |
| RETRY_STARTUP        | any        | ""            |
| RETRY_SEND           | any        | ""            |
| DECODE_JSON_LOGS     | bool       | true          |


### Update fix lỗi lấy hostname
1. Cài đặt ubuntu trên windown chạy được centos.
2. Cập nhật file module.go ở repo https://github.com/gliderlabs/logspout
   	_ "github.com/vutung189/logspout-logstash"
3. 	docker build --no-cache --pull -t tungvt/logspout .
4. 	save and deploy
docker save tungvt/logspout > tungvt-logspout.tar
docker load < tungvt-logspout.tar
docker image tag tungvt/logspout:latest registry:5000/tungvt/logspout
docker image push registry:5000/tungvt/logspout:latest
