# VESPA METRICS EXPORTER

## Exporting Vespa metrics to Prometheus

Specify vespa-configserver (hostname:port) in environment VESPA_CONFIGSERVER

Usage with docker

    # docker build -t vespa-exporter .
    # docker run -e VESPA_CONFIGSERVER=vespa-configserver.example.com:19071 vespa-exporter

# VESPA APPLICATION MONITORING 
The vespa metrics dashboard in is available in [grafana](https://fy.grafana.net/d/PG6bV_rmz/vespa-prod).

## Metrics Collection

### Overview

The query latency metrics are collected by the `fy.api` service, 
check them on [grafana](https://fy.grafana.net/d/PG6bV_rmz/vespa-prod).
You can see there the 95th, 90th, 75th and median percentile along with the average.

The most trustworthy metric for overload exposed by `vespa-exporter` is the prioritized 
partition queues on the content nodes.
Most notable metrics are:
```asciidoc
vespa_searchnode_vds_filestor_alldisks_queuesize
vespa_searchnode_vds_visitor_allthreads_queuesize
```
Icreased size of this metrics means the cluster can't keep up with the requests.
refer to vespa [docs](https://docs.vespa.ai/documentation/operations/admin-procedures.html#detecting-overload)
for more details how to detect the overload.

Vespa has a metric of `ideal state` which relates to [distribution algorithm](https://docs.vespa.ai/documentation/content/idealstate.html)
`idealstate.idealstate_diff` metric tries to summarise this state indicating distance to the ideal state.
A value of zero indicates that the cluster is in the ideal state. Graphed values of this metric gives a good indication for how fast the cluster gets back to the ideal state after changes.
More detailed on this in vespa [docs](https://docs.vespa.ai/documentation/operations/admin-procedures.html).

## Vespa-exporter
`vespa-exporter-container` is a dockerized service that collects the metrics from the 
vespa services.It runs as part of `prometheus-stack-srv` serivce in `fy-bang` cluster on `ecs`.
Check its repo [here](https://github.com/Project-J/vespa_exporter).

### Configure vespa-exporter to include new metrics
By defaut the service is exposing all known vespa metrics from all vespa services which can
be overwhelming. That's why there is a file `whitelist.txt` where you can specify the desired
metrics to collect.

### Deploy vespa-exporter

```asciidoc
docker build -t vespa-exporter .
docker tag vespa-exporter:latest 243905809578.dkr.ecr.eu-west-1.amazonaws.com/vespa-metrics-exporter:production
eval $(aws ecr get-login --region eu-west-1 --no-include-email)
docker push  243905809578.dkr.ecr.eu-west-1.amazonaws.com/vespa-metrics-exporter:production

```

### Exploring new metrics

Vespa is composed of several services. To explore them run:
```bash
$ vespa-model-inspect services

```
Example output:

```bash
[root@ip-172-31-42-62 /]# vespa-model-inspect services
config-sentinel
configproxy
configserver
container
container-clustercontroller
distributor
logd
logserver
searchnode
slobrok
storagenode
topleveldispatch
transactionlogserver

```

To find the status page port for a service run:
```asciidoc
$ vespa-model-inspect service <service-name>

```

Ex:

```asciidoc
[root@ip-172-31-42-62 /]# /opt/vespa/bin/vespa-model-inspect service searchnode
searchnode @ ip-172-31-41-126.eu-west-1.compute.internal : search
catalogue/search/cluster.catalogue/0
    tcp/ip-172-31-41-126.eu-west-1.compute.internal:19103 (STATUS ADMIN RTC RPC)
    tcp/ip-172-31-41-126.eu-west-1.compute.internal:19104 (FS4)
    tcp/ip-172-31-41-126.eu-west-1.compute.internal:19105 (UNUSED)
    tcp/ip-172-31-41-126.eu-west-1.compute.internal:19106 (UNUSED)
    tcp/ip-172-31-41-126.eu-west-1.compute.internal:19107 (STATE HEALTH JSON HTTP)
searchnode @ ip-172-31-10-30.eu-west-1.compute.internal : search
catalogue/search/cluster.catalogue/1
    tcp/ip-172-31-10-30.eu-west-1.compute.internal:19103 (STATUS ADMIN RTC RPC)
    tcp/ip-172-31-10-30.eu-west-1.compute.internal:19104 (FS4)
    tcp/ip-172-31-10-30.eu-west-1.compute.internal:19105 (UNUSED)
    tcp/ip-172-31-10-30.eu-west-1.compute.internal:19106 (UNUSED)
    tcp/ip-172-31-10-30.eu-west-1.compute.internal:19107 (STATE HEALTH JSON HTTP)
```

Make a request to `http://ip-172-31-41-126.eu-west-1.compute.internal:19107/state/v1/metrics`
to inspect the metrics.

Add desired metrics to `vespa-exporter` in the `whitelist.txt` file and redeploy service.

### Vespa custom metrics
You can also set up Vespa to report your own custom metrics in custom components.
See vespa [metrics](https://docs.vespa.ai/documentation/jdisc/metrics.html)