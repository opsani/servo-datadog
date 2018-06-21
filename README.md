# servo-datadog
Optune servo driver for Datadog

Note: this driver requires `measure.py` base class from the Optune servo core. It can be copied or symlinked here as part of packaging

Datadog metric queries are comprised of a metric name and, typically, one or more modifications.  For example, the query:
```
docker.net.bytes_sent{kube_namespace:abc}by{kube_deployment}
```
returns a *group* of time series pointlists, one for *each* k8s deployment in the `abc` namespace, where each point of each pointlist includes the value of the metric `docker.net.bytes_sent`.  

The driver presently returns a single value for a metric named `perf`.  This value is computed from the time series pointlist(s) returned from a single query as follows:

* time aggregation:  compute a value for each time series pointlist using one of these methods:  avg, max, min, sum
* space aggregation:  compute a single value for the entire *group* of time series by aggregating the time aggregations using one of these methods:  avg, max, min, sum (if there is just one pointlist, these are all the same)

The Datadog driver does not presently support multiple queries (e.g., so that a performance metric may be created by a formula using multiple query results).  The `describe` command returns the list of active metric names.  By itself, a metric name is often not a valid query, and such a query will return an error.

# servo configuration

A servo which uses this measure driver requires the Datadog api key and app key, which are used to authenticate with the Datadog API server, to exist in the following files:

* api_key:  /etc/optune-datadog-auth/api_key
* app_key:  /etc/optune-datadog-auth/app_key

These files can be automatically created on a Kubernetes servo using a secret mounted as `/etc/optune-datadog-auth`.  See the following example.

Create a secret in namespace `abc` using kubectl:
```
kubectl -n abc create secret generic optune-datadog-auth \
--from-literal=api_key='<my_value>' \
--from-literal=app_key='<my_value>'
```

Configure the servo deployment YAML descriptor:
```
spec:
  template:
    spec:
      volumes:
      - name: datadog
        secret:
          secretName: optune-datadog-auth   
      containers:
      -name main
        ...
        volumeMounts:
        - name: datadog
          mountPath: '/etc/optune-datadog-auth'
          readOnly: true               
```

# driver configuration

In addition to the standard `warmup` and `duration` configuration, the driver measurement control specification provides for configuring the Datadog query, and the method(s) for computing a single `perf` result, using the free-from `userdata` section of the operator override descriptor (e.g., an app-desc.yaml provided to the server).  For example:

```
measurement:
  control:
    warmup:    30
    duration:  130
    userdata:
      query:  'docker.net.bytes_sent{kube_namespace:abc}by{kube_deployment}'
      time_aggr:   avg       # compute time-series value as the average
      space_aggr:  sum       # compute group value (the perf metric) as
                             # the sum of all time series aggregate values
```

* `warmup`:  period after adjustment when a measurement is not taken (sleep).  Default 0 seconds.
* `duration`:  period of measurement.  Default 120 seconds.
* `query`: a Datadog metric time series query.  Required. 
* `time_aggr`:  aggregation method for time series pointlists.  One of avg|max|min|sum.  Default `avg`.
* `space_aggr`:  aggregation method for time series aggregate values - used to create a single `perf` metric value.  One of avg|max|min|sum.  Default `avg`.

__FUTURE__:  when this driver supports multiple queries (and user-named metrics), the configuration for `query`, `time_aggr` and `space_aggr` may be provided for each user-named metric, in the `metrics` section of the descriptor rather than the `control` section, e.g.:

```
measurement:
  control:
    warmup:    30
    duration:  130
  metrics:
    my_tx:  # user-named metric
      ...   # max, min, value, etc.
      userdata:
        query: 'docker.net.bytes_sent{kube_namespace:abc}by{kube_deployment}'
        time_aggr:   avg
        space_aggr:  sum
    my_cpu_usage:  # user-named metric
      ...
      userdata:
        query: 'docker.cpu.usage{kube_namespace:abc}'
        time_aggr:   avg
        space_aggr:  avg
```
