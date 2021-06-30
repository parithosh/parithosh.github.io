# Logging with Loki 

Loki is a log aggregation system inspired by Prometheus. It is extremely easy to integrate it with Grafana as well as 
the alertmanager. 

In this blogpost, we will look at deploying Loki, deploying Promtail and collecting docker logs with the log driver. 

## Loki
We will be deploying Loki as a docker container. 

- Create a directory to store the config
- Copy the loki configuration inside this folder called `loki-config.yml` with the below contents:  


```
# Enables authentication through the X-Scope-OrgID header, which must be present
# if true. If false, the OrgID will always be set to "fake".
auth_enabled: false

# Configures the server of the launched module(s).
server:
  http_listen_port: 3100
  grpc_listen_port: 9096

# Configures the ingester and how the ingester will register itself to a
# key value store.
ingester:
  wal:
    enabled: true
    dir: /tmp/wal
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 1h       # Any chunk not receiving new logs in this time will be flushed
  max_chunk_age: 1h           # All chunks will be flushed when they hit this age, default is 1h
  chunk_target_size: 1048576  # Loki will attempt to build chunks up to 1.5MB, flushing first if chunk_idle_period or max_chunk_age is reached first
  chunk_retain_period: 30s    # Must be greater than index read cache TTL if using an index cache (Default index read cache TTL is 5m)
  max_transfer_retries: 0     # Chunk transfers disabled

# Configures the chunk index schema
schema_config:
  configs:
    - from: 2021-06-23
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

# Configures where Loki will store data.
storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/boltdb-shipper-active
    cache_location: /tmp/loki/boltdb-shipper-cache
    cache_ttl: 24h         # Can be increased for faster performance over longer query periods, uses more disk space
    shared_store: filesystem
  filesystem:
    directory: "/mnt/config/chunks"

# Configures the compactor component which compacts index shards for performance.
compactor:
  working_directory: /tmp/loki/boltdb-shipper-compactor
  shared_store: filesystem

# Configures limits per-tenant or globally
# ingestation when promtail is run for the first time can be a lot of data, hence the high burst limit
limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 50
  ingestion_burst_size_mb: 150

# Configures how Loki will store data in the specific store.
chunk_store_config:
  max_look_back_period: 0s

# Configures the table manager for retention
table_manager:
  retention_deletes_enabled: false
  retention_period: 0s

# The ruler_config configures the Loki ruler.
ruler:
  storage:
    type: local
    local:
      directory: /tmp/loki/rules
  rule_path: /tmp/loki/rules-temp
  alertmanager_url: http://localhost:9093
  ring:
    kvstore:
      store: inmemory
  enable_api: true
```
- Run the docker container with the below command:  


```
docker run -d --name loki --restart unless-stopped -p 3100:3100 -v {{ loki_dir }}:/mnt/config grafana/loki:latest --config.file=/mnt/config/loki-config.yaml
```
- Run `curl localhost:3100/ready` to check if the container is up as expected

Now we have a running version of Loki, we just need to get logs to it. 

## Promtail
Promtail is a collector agent that scrapes the logs on a
local machine, parses them and sends the logs to a Loki
instance for longer term storage. 

We will also run promtail as a docker container. We use a volume mapping in read-only mode to allow Promtail to read the
system logs of the machine. These logs are then scraped as specified in the `scrape_configs`, similar to Prometheus.
Any extra labels can be added at this stage. 

- Create a directory to store the config
- Copy the promtail configuration inside this folder called `promtail-config.yml` with the below contents:  


```
# speficies the promtail server ports
server:
  http_listen_port:9080
  grpc_listen_port:0

# specifies the temp positions directory
positions:
  filename: /tmp/positions.yaml

# specifies the loki server URL to which logs are sent
clients:
  - url: http://localhost:3100/loki/api/v1/push

# specifies the scrape locations for the logs as well as any postprocessing
scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
```
- Run the docker container with the below command:  

```
docker run -d --name promtail --restart unless-stopped  -v {{ promtail_dir }}:/mnt/config -v /var/log:/var/log:ro grafana/promtail:2.2.1 --config.file=/mnt/config/promtail-config.yaml
```

## Docker logs

While it is easy for promtail to forward docker logs and tag them, it is relatively hard for it to label the logs with the
associated `container_name` or other labels. This could be done before running the docker container itself, but for sake 
of simplicity as well as not needing to always add labels, we can set it as the default docker log method. 

We achieve this by installing the loki docker plugin and then by setting the default docker log method to it. That way 
all newly created containers will use the new log plugin. 

- Install the docker loki plugin with:  
`docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions`  
- Either modify or create a file at `/etc/docker/daemon.json` with the following content. Add any extra `log-opts` if needed:  


```
{
  "debug" : true,
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "http://localhost:{{ loki_http_listen_port }}/loki/api/v1/push",
    "loki-batch-size": "400",
    "loki-external-labels": "job=dockerlogs,container_name={{.Name}}"
  }
}
```
- Restart the docker engine to apply the changed with: `sudo systemctl restart docker`
- Now we can try running a new container and send logs directly to loki with `docker run -it --name test alpine echo hi`

As long as Loki has been added as a data source on Grafana, we should be able to see the logs on the `Explore tab` on Grafana.

