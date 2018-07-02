# Workshop part 2

## Installation

1. If the agent from part 1 is still running, stop it with the [following command][4]: `sudo service datadog-agent stop`

2. Go in `/vagrant/workshop/part_2/`

3. Build the flask container: `docker-compose build`

## Step 1 : Simple flask app 
### Spawn the flask app 

Launch the first flask app:

`docker-compose up`

Try it out with one of the following command:

* `curl -X GET http://localhost:8080/think/?subject=technology`
* `curl -X GET http://localhost:8080/think/?subject=religion`
* `curl -X GET http://localhost:8080/think/?subject=war`
* `curl -X GET http://localhost:8080/think/?subject=work`
* `curl -X GET http://localhost:8080/think/?subject=music`
* `curl -X GET http://localhost:8080/think/?subject=humankind`

## Step 2 : Implement metric monitoring 
### Setup

Stop and remove all containers:

`docker-compose rm & docker-compose stop`

Add the following line to the `docker-compose.yaml` file to run the agent along side our app:

```
datadog:
    container_name: datadog_agent
    image: datadog/agent:latest
    environment:
      - DD_HOSTNAME=workshop_part_2
      - DD_API_KEY=${DD_API_KEY}
    volumes:
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

In order to avoid any launch issue, make sure that the DD agent is the last container to start by adding:

```
datadog:
  (...)
  depends_on:
        - nginx
        - api
```


In order to start metrics collection we are going to use labels on our containers ([learn more about auto-discovery in our documentation][5]):

For NGINX, according to [the NGINX documentation][6]:

```
    label:
        com.datadoghq.ad.check_names: '["nginx"]'
        com.datadoghq.ad.init_configs: '[{}]'
        com.datadoghq.ad.instances: '[{"nginx_status_url": "http://%%host%%:%%port%%/nginx_status"}]'
```

For Redis, according to [the Redis documentation][7]:

```
    label:
        com.datadoghq.ad.check_names: '["redis"]'
        com.datadoghq.ad.init_configs: '[{}]'
        com.datadoghq.ad.instances: '[{"host": "%%host%%", "port": "6379"}]'
```


Once done, re-spawn your containers and go back to your [Datadog application][]. 

`docker-compose up`

### Explore in Datadog:

Because of the check name, we can see that there are dashboard already created in your datadog application:

* [Nginx Overview][1]
* [Nginx Metrics][2]
* [Redis Overview][3]

We now have a clear state of our system and we could check if there is an issue but let's try to have more insights 

## Step 3: Implement Trace monitoring 

Let's switch to the branch `app-with-tracing` we now have our app that is instrumented with traces.

Let's build it again:

```
docker-compose build
```

Now let's configure the DD agent to gather those traces and send them to Datadog:

let's add the following environment variable to our `datadog` container:

```
- DD_APM_ENABLED=true
```

and then restart the docker-compose:

```
docker-compose up
```

If we go back to our Datadog application we can now start to see some traces flowing in for our `redis`, `thinker-api`, `thinker-microservice`

It's great we now understand the behaviour of our request but it's not enough

## Step 4: Implement log monitoring

### Basic log collection

Our application is already emitting some logs let's catch them with our datadog container:

```
datadog:
    container_name: datadog_agent
    image: datadog/agent:latest
    environment:
      - DD_HOSTNAME=workshop_part_2
      - DD_API_KEY=${DD_API_KEY}
      - DD_APM_ENABLED=true
      - DD_LOGS_ENABLED=true
      - DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true
    volumes:
      - /opt/datadog-agent/run:/opt/datadog-agent/run:rw
      - /proc/mounts:/host/proc/mounts:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - nginx
      - api
```

Let's use the datadog-agent but this time with log-collection enabled

Let's restart everything and start see our logs in our DD app:

```
docker-compose stop
docker-compose rm
docker-compose up
```


As you can see we can see our logs flowing in but they are not parsed yet
Check the info bubble on your logs to see which one are not parsed yet.

Datadog have a range of supported integration, and those are enabled according to the service-attributes value.

### Integration log collection 

In order to know how to parse a log, we just need to pass the source name as a label:

Enhance your docker compose with the following labels

```
redis:
    (...)
    labels:
        (...)
        com.datadoghq.ad.logs: '[{"source": "redis", "service": "docker-redis"}]'
nginx:
    (...)
    label:
        (...)
        com.datadoghq.ad.logs: '[{"source": "nginx", "service": "docker-nginx"}]'
```

restart everything and watch what is happening:

```
docker-compose stop
docker-compose rm
docker-compose up
```


Check if [the integration pipelines][9] are created and are parsing your logs.

### Binding logs traces and metrics

Tags are what is allowing us to bind metrics / traces / logs.

Let's add log tag to our `thinker-api` and `thinker-microservice` in order to be able to bind the traces and the log together

Enhance your docker compose with the following labels:

```
api:
    (...) 
    labels:
        (...)
        com.datadoghq.ad.logs: '[{"source": "webapp", "service": "thinker-api"}]'

thinker:
    (...)
    labels:
        (...)
        labels:
      com.datadoghq.ad.logs: '[{"source": "webapp", "service": "thinker-microservice"}]'
```

Restart everything and watch what is happening:

```
docker-compose stop
docker-compose rm
docker-compose up
```

## (Bonus) Adding new logs

Our log collection and binding is now over, let's update our app with a new log and see what is happening

in `thinker.py` in the `think()` function let's add a dummy log: 

```
    redis_client.incr('hits')
    aiohttp_logger.info('Number of hits is {}' .format(redis_client.get('hits').decode('utf-8')))
```

We just count the amount of hits and store the number in Redis itself

Restart everything and watch what is happening:

```
docker-compose stop
docker-compose rm
docker-compose up
```

we can now see the log flowing in with the right tags.

Let's parse it, add the hits_number as a facet and now add a monitor on its derivative or what-ever. 

[1]: https://app.datadoghq.com/screen/integration/21/nginx---overview
[2]: https://app.datadoghq.com/dash/integration/20/nginx---metrics
[3]: https://app.datadoghq.com/screen/integration/15/redis---overview
[4]: https://docs.datadoghq.com/agent/basic_agent_usage/amazonlinux/#commands
[5]: https://docs.datadoghq.com/agent/autodiscovery/
[6]: https://docs.datadoghq.com/integrations/nginx/
[7]: https://docs.datadoghq.com/integrations/redisdb/
[8]: https://app.datadoghq.com/
[9]: https://app.datadoghq.com/logs/pipelines