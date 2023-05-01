# Grafana Dashboard for monitoring InfluxDB Enterprise

This Grafana template visualises statistics about an [InfluxDB Enterprise 1.x](https://www.influxdata.com/products/influxdb-enterprise/) cluster.

It's intended to be complimentary to the process described in [Monitor InfluxDB Enterprise with InfluxDB OSS](https://docs.influxdata.com/enterprise_influxdb/v1.10/administration/monitor/monitor-with-oss/), providing the means to visualise the collected metrics via an existing Grafana deployment.

![Screenshot of the top half of the dashboard](/screenshots/dashboard.png)

----

### Exposed Metrics

The dashboard visualises the following metrics

* License expiry dates
* System resource allocations (disk/cpu/RAM)
* System resource usage (disk/cpu/RAM)
* Series Cardinality
* Shard count
* Hinted Handoff Queue sizes
* Point Throughput
* HTTP request rates and durations
* Continuous query metrics
* Query rates and durations
* Write rates and durations
* Batch size information
* IOPS and I/O Queue utilisation
* Storage bandwidth utilisation

----

### Pre-Requisites

- An InfluxDB Enterprise cluster (some stats will also work with OSS)
- An InfluxDB OSS deployment
- A Grafana deployment

----

### Setup

Follow the steps detailed in [Monitor InfluxDB Enterprise with InfluxDB OSS](https://docs.influxdata.com/enterprise_influxdb/v1.10/administration/monitor/monitor-with-oss/) to begin collecting metrics.

Once metrics are being collected, a new datasource needs to be configured in Grafana so that it can query against the OSS deployment.

Generate a token in OSS:

1. Log into your OSS deployment's UI
1. Click the upward arrow
1. Choose `API Tokens`
1. Click `Generate API Token`
1. `Custom API Token`
1. Call the token "Grafana Read" or similar
1. Enable read access on the relevant bucket
1. Click `Generate` and note the token value

In Grafana:

1. Click the settings cog (or, in newer Grafana versions `Connections`)
1. Add an InfluxDB datasource
1. Set an appropriate name
1. Set `Query Language` to `Flux`
1. Enter the URL Grafana should use to reach your OSS deployment
1. Disable Basic Auth
1. Provide the organisation name you configured during OSS setup
1. Paste the token generated in part one
1. Set the default bucket to `telegraf` (unless you changed the name in Telegraf's config)
1. Click `Save & Test`

Import the dashboard:

1. Choose Dashboards
1. Click `New`
1. Choose `Import`
1. Click to upload JSON
1. Choose `InfluxDB_Enterprise.json` from this repo
1. Select the InfluxDB datasource that was created above

----

### Dashboard Variables

The dashboard only exposes a single variable: `bucket`. This defaults to `telegraf`, but allows the source bucket name to be overridden if another was used whilst configuring telegraf.

----

### Monitoring Multiple Clusters

If multiple InfluxDB Enterprise clusters are in use, it's possible to amend the deployed configuration and template to facilitate monitoring of these clusters via a single OSS instance and dashboard.

The Telegraf configuration will need to be amended to add a tag identifying the cluster

```ini
[global_tags]
   cluster_id = "prod1"
```

Telegraf will need to be restarted after adding this.


The Dashboard itself then needs to be amended to add a variable


1. Browse into the dashboard
1. Click the settings cog
1. Choose Variables
1. Click `New Variable`

Use the following settings

* Variable type: `Query`
* Name: `cluster_id`
* Data Source: Your OSS Instance

And set the following query
```
from(bucket: "telegraf")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> keyValues(keyColumns: ["cluster_id"])
  |> group()
  |> keep(columns: ["_value"])
```

Set `Refresh` to `On time range change` and click `Apply`.

Edit each of the dashboard cells to add the following under the measurement on `r._measurement`
```
  |> filter(fn: (r) => r.cluster_id == "${cluster_id}")
```

Save the changes.

