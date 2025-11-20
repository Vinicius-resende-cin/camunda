# Monitoring

## Metrics

Zeebe and other Camunda components export several metrics to facilitate monitoring a cluster.
Currently, metrics are exported using Prometheus. You can find
documentation about the different zeebe metrics
[here](https://docs.camunda.io/docs/product-manuals/zeebe/deployment-guide/operations/metrics).

### Testing

You can easily test metrics locally by using the standard provided [docker compose
file](../docker/compose/docker-compose.yaml) in combination with the one [here](docker-compose.yml), e.g.:

```sh
docker-compose --project-directory ./ -f docker-compose.yml -f ../docker/compose/docker-compose.yaml up -d
```

This will start the usual 3 brokers cluster, as well as a Grafana [instance](http://localhost:3000/) (on port 3000; login: u `admin`, p `camunda`) and a Prometheus instance on
port 9090. The Prometheus instance is configured to scrape the brokers every 5 seconds, and pre-assigns them the
namespace and pod label as `local` and `broker-*`.

> Remember that docker-compose does not remove volumes on the down command, so if you are completely done with it you
> will need to run either `docker-compose --project-directory ./ -f docker-compose.yml -f ../docker/compose/docker-compose.yaml down -v`
> or `docker volume prune`

### Testing with local Zeebe

When you want to use a local Zeebe Broker, you need to locally modify the config:
- enable Prometheus Docker container to access localhost ports:

```yaml
# add to the prometheus service
extra_hosts:
- "host.docker.internal:host-gateway"
```

- add the local Zeebe broker to Prometheus config:

  ```yaml
  # add to scrape_configs
  - job_name: 'zeebe_local'
    metrics_path: /actuator/prometheus
    static_configs:
         - targets: ['host.docker.internal:9600']

  ```

## Grafana

You can find a pre-built Grafana dashboard [here](grafana/zeebe.json) to
visualize most metrics. This is the dashboard that we use to test and
monitor our own Zeebe installations.

> NOTE: this dashboard is used for development and can serve as a
> starting point for your own dashboard, but may not be tailored for your
> particular use case.

See the Grafana documentation on
[how to import a dashboard](https://grafana.com/docs/grafana/latest/reference/export_import/#importing-a-dashboard).

#### Variables

The dashboards at the moment have most visualizations scoped to the
following variables: `namespace` (the k8s namespace), `pod` (the k8s pod),
and [partition](https://docs.camunda.io/docs/product-manuals/zeebe/technical-concepts/partitions).

### Contributing

This guide focuses on making use of existing metrics to create visualizations in Grafana.
If you want to learn how to add new metrics, a good starting point can be found in the observability documentation [here](../docs/observability/metrics.md).

By default, we use [Prometheus](https://prometheus.io/) to collect metrics, so queries for our dashboards are written in [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/).

#### Dashboard

**Modifications** can be made and tested in Grafana (dev or int environment) but should then be exported as JSON and updated in the codebase:

1) In edit mode, save your modifications as a copy
2) Then exit edit mode and make sure to export as described
[here](https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/#export-a-dashboard-as-json),
checking `Export the dashboard to use in another instance` as you do.
3) Paste the modified dashboard JSON into the codebase, replacing the existing one (see note about deployment below)

**Note**: For local development, you can use [Grizzly](https://grafana.com/blog/2024/10/29/edit-your-git-based-grafana-dashboards-locally/) to run Grafana locally, which can be started through the `make grizzly` command (see [Makefile](Makefile)) and instead make the modifications directly in the codebase.

**Deployment**

Our dashboards are deployed from the (Camunda internal) [grafana-dashboards repository](https://github.com/camunda/grafana-dashboards). You can follow the information there for creating new dashboards or folders, which also includes some best practices for designing Grafana dashboards that should be followed before deployment.
Some dashboards are directly maintained in that repository (e.g. SRE and Controller) and need to be updated there. Others (e.g. Zeebe, Data Layer and Core Features) are defined and maintained in this repository in the [grafana folder](grafana) and get fetched by linking the raw Github content of the dashboard definition like:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-dashboard
data:
  example-dashboard.json.url: https://raw.githubusercontent.com/camunda/camunda/main/monitor/grafana/dashboards/example-team/example-dashboard.json
```

You can refer to existing dashboards in that repository for examples if you are deploying a new dashboard, or use it to check where the deployed dashboards are maintained.

**Lessons Learnt**

Here are some tips for working with Grafana that may serve as guidance for someone new to metrics and PromQL:
* Build queries iteratively "from the inside out": Start with just the metric you want to display and understand what it represents even if you already have the operators you want to use on it in mind.
* Think of metrics as database queries. If there is a way you would want to operate on the data in SQL, it probably also exists in PromQL - including joins on other metrics.
* There is always some small way to improve every panel you make, either through refining the queries or tweaking little parts of how it's visualized. At some point you need to make the decision that it's good enough, push it, and then iterate over it later with feedback from actual usage.

#### Zeebe Preview

![Zeebe Grafana Dashboard Preview](grafana/preview.png)

