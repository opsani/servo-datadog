# servo-datadog

Optune servo driver for Datadog

Note: this driver requires `measure.py` base class from the Optune servo core. It can be copied or symlinked here as part of packaging

Datadog metric queries are comprised of a datadog metric name and, typically, one or more modifications.  For example, the query:

```none
docker.net.bytes_sent{kube_namespace:abc}by{kube_deployment}
```

returns a *group* of time series pointlists, one for *each* k8s deployment in the `abc` namespace, where each point of each pointlist includes the value of the metric `docker.net.bytes_sent`.  

The driver's `measure` command presently returns one or more timeseries metrics with each configured metric name containing a `values` list with items grouped by the tag value of the tag key specified by the configuration of `use_as_instance_id`. Each `values` list item contains an `id` of said tag value and a `data` list in the form of `[[POSIX_timestamp, numeric_value], ...]`. The `measure` command requires a `control` dictionary be included in its input; this section may optionally specify a `warmup` and/or `duration` value to override the driver's defaults

The `describe` command returns the list of configured metric names as well as their units. By itself, a configured metric name is often not a valid query, and such a query will return an error.

## servo configuration

A servo which uses this measure driver requires the Datadog api key and app key, which are used to authenticate with the Datadog API server, to exist in the following files:

* api_key:  /etc/optune-datadog-auth/api_key
* app_key:  /etc/optune-datadog-auth/app_key

These files can be automatically created on a Kubernetes servo using a secret mounted as `/etc/optune-datadog-auth`.  See the following example.

Create a secret in namespace `abc` using kubectl:

```bash
kubectl -n abc create secret generic optune-datadog-auth \
--from-literal=api_key='<my_value>' \
--from-literal=app_key='<my_value>'
```

Configure the servo deployment YAML descriptor:

```yaml
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

## driver configuration

The driver requires a `datadog` section be included in the config file (usually located in a config.yaml file in the local folder though this location can be overriden with the environment variable `CONFIG_FPATH`). The section must contain a `metrics` sub-section of one or more metrics; each key within will be used as the metric name during the `describe` and `measure` commands. The values of each metric name key must include the following:

* A `query` to be issued to the datadog API
* A `use_as_instance_id` which identifies the tag key to be used in grouping the returned query data
* (optional) A `unit` to be returned with the metric name during the `describe` command

For example:

```yaml
datadog:
  metrics:
    cpu_usage:
      query: kubernetes.cpu.usage.total{kube_namespace:default}by{pod_name}
      use_as_instance_id: pod_name
      unit: millicores
    memory_usage:
      query: kubernetes.memory.usage{kube_namespace:default}by{pod_name}
      use_as_instance_id: pod_name
      unit: bytes
    replicas:
      query: kubernetes.pods.running{kube_namespace:default}by{kube_namespace}
      use_as_instance_id: kube_namespace
      unit: bytes
```
