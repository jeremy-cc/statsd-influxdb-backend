CurrencyCloud StatsD InfluxDB backend
-------------------------------------

## Preamble
This is a fork of several upstream statsd-influxdb-backend projects which have been merged into a single npm module
in order to permit collection of influxdb tags via statsd on AWS.

It is offered as-is in the hope it will prove useful.

Original implementation: [https://github.com/bernd/statsd-influxdb-backend](https://github.com/bernd/statsd-influxdb-backend)

Upstream parents: 
* [https://github.com/guydou/statsd-influxdb-backend](https://github.com/guydou/statsd-influxdb-backend)
* [https://github.com/gillesdemey/statsd-influxdb-backend](https://github.com/gillesdemey/statsd-influxdb-backend)

## Installation

    $ cd /path/to/statsd
    $ npm install currencycloud-statsd-influxdb-backend

## Configuration

You can configure the following settings in your StatsD config file.

```js
{
  graphitePort: 2003,
  graphiteHost: "graphite.example.com",
  port: 8125,
  backends: [ "./backends/graphite", "currencycloud-statsd-influxdb-backend" ],

  influxdb: {
    host: '127.0.0.1',   // InfluxDB host. (default 127.0.0.1)
    port: 8086,          // InfluxDB port. (default 8086)
    version: 0.8,        // InfluxDB version. (default 0.8, can be 1.0)
    ssl: false,          // InfluxDB is hosted over SSL. (default false)
    database: 'dbname',  // InfluxDB database instance. (required)
    username: 'user',    // InfluxDB database username.
    password: 'pass',    // InfluxDB database password.
    flush: {
      enable: true       // Enable regular flush strategy. (default true)
    },
    proxy: {
      enable: false,       // Enable the proxy strategy. (default false)
      suffix: 'raw',       // Metric name suffix. (default 'raw')
      flushInterval: 1000  // Flush interval for the internal buffer.
                           // (default 1000)
    },
    includeStatsdMetrics: false, // Send internal statsd metrics to InfluxDB. (default false)
    includeInfluxdbMetrics: false, // Send internal backend metrics to InfluxDB. (default false)
    keyNameSanitize: false         // permits tags to be passed through without stripping the separating commas between them
                                   // Requires includeStatsdMetrics to be enabled.
  }
}
```

## Activation

Add the `currencycloud-statsd-influxdb-backend` to the list of StatsD backends in the config
file and restart the StatsD process.

```js
{
  backends: [..., 'currencycloud-statsd-influxdb-backend']
}
```

## Unsupported Metric Types

#### Proxy Strategy

* Counter with sampling.
* Signed gauges. (i.e. `bytes:+4|g`)
* Sets

## InfluxDB Event Mapping

StatsD packets are currently mapped to the following InfluxDB events. This is
a first try and I'm open to suggestions to improve this.

### Set

StatsD package `client_version:1.1|c`, `client_version:1.2|c` as Influx event:

```js
[
  {
    name: 'visior',
    columns: ['value', 'time'],
    points:  [['1.1', 1384798553000], ['1.2', 1384798553001]]
  }
]
```

If you are using Grafana to visualize a Set, then using this query or
something similar

```
SELECT version, count(version) FROM client_version GROUP BY version, time(1m)
```

Also, to count for the size of unique value, another InfluxDB event is
also pushed

```js
[
  {
    name: 'visitor_count',
    columns: ['value', 'time'],
    points:  [set.length, 1384798553001]
  }
]
```

### Counter

StatsD packet `requests:1|c` as InfluxDB event:

#### Flush Strategy

```js
[
  {
    name: 'requests.counter',
    columns: ['value', 'time'],
    points: [[802, 1384798553000]]
  }
]
```

#### Proxy Strategy

```js
[
  {
    name: 'requests.counter.raw',
    columns: ['value', 'time'],
    points: [[1, 1384472029572]]
  }
]
```

### Timing

StatsD packet `response_time:170|ms` as InfluxDB event:

#### Flush Strategy

```js
[
  {
    name: 'response_time.timer.mean_90',
    columns: ['value', 'time'],
    points: [[445.25761772853184, 1384798553000]]
  },
  {
    name: 'response_time.timer.upper_90',
    columns: ['value', 'time'],
    points: [[905, 1384798553000]]
  },
  {
    name: 'response_time.timer.sum_90',
    columns: ['value', 'time'],
    points: [[321476, 1384798553000]]
  },
  {
    name: 'response_time.timer.std',
    columns: ['value', 'time'],
    points: [[294.4171159604542, 1384798553000]]
  },
  {
    name: 'response_time.timer.upper',
    columns: ['value', 'time'],
    points: [[998, 1384798553000]]
  },
  {
    name: 'response_time.timer.lower',
    columns: ['value', 'time'],
    points: [[2, 1384798553000]]
  },
  {
    name: 'response_time.timer.count',
    columns: ['value', 'time'],
    points: [[802, 1384798553000]]
  },
  {
    name: 'response_time.timer.count_ps',
    columns: ['value', 'time'],
    points: [[80.2, 1384798553000]]
  },
  {
    name: 'response_time.timer.sum',
    columns: ['value', 'time'],
    points: [[397501, 1384798553000]]
  },
  {
    name: 'response_time.timer.mean',
    columns: ['value', 'time'],
    points: [[495.6371571072319, 1384798553000]]
  },
  {
    name: 'response_time.timer.median',
    columns: ['value', 'time'],
    points: [[483, 1384798553000]]
  }
]
```

#### Proxy Strategy

```js
[
  {
    name: 'response_time.timer.raw',
    columns: ['value', 'time'],
    points: [[170, 1384472029572]]
  }
]
```

### Gauges

StatsD packet `bytes:123|g` as InfluxDB event:

#### Flush Strategy

```js
[
  {
    name: 'bytes.gauge',
    columns: ['value', 'time'],
    points: [[123, 1384798553000]]
  }
]
```

#### Proxy Strategy

```js
[
  {
    name: 'bytes.gauge.raw',
    columns: ['value', 'time'],
    points: [['gauge', 123, 1384472029572]]
  }
]
```

## Proxy Strategy Notes

### Event Buffering

To avoid one HTTP request per StatsD packet, the InfluxDB backend buffers the
incoming events and flushes the buffer on a regular basis. The current default
is 1000ms. Use the `influxdb.proxy.flushInterval` to change the interval.

This might become a problem with lots of incoming events.

The payload of a HTTP request might look like this:

```js
[
  {
    name: 'requests.counter.raw',
    columns: ['value', 'time'],
    points: [
      [1, 1384472029572],
      [1, 1384472029573],
      [1, 1384472029580]
    ]
  },
  {
    name: 'response_time.timer.raw',
    columns: ['value', 'time'],
    points: [
      [170, 1384472029570],
      [189, 1384472029572],
      [234, 1384472029578],
      [135, 1384472029585]
    ]
  },
  {
    name: 'bytes.gauge.raw',
    columns: ['value', 'time'],
    points: [
      [123, 1384472029572],
      [123, 1384472029580]
    ]
  }
]
```

## Backend Metrics

The following internal metrics are calculated for each flush:

- `statsd.influxdbStats.flush_time` - Time taken to process a complete flush in ms. Excluding the asynchronous HTTP Post.
- `statsd.influxdbStats.http_response_time` - Response time in ms of the InfluxDB HTTP endpoint when POSTing data.
- `statsd.influxdbStats.payload_size` - The size in bytes of the JSON payload.
- `statsd.influxdbStats.num_stats` - The number of metrics sent to InfluxDB in the last flush.

These are added to the set of internal statsd metrics. If both `influxdb.includeStatsdMetrics` and `influxdb.includeInfluxdbMetrics` are enabled, then these will be sent to InfluxDB when using the flush strategy.

The internal metrics can also can be viewed using the `stats` command on the [StatsD TCP Admin Interface](https://github.com/etsy/statsd/blob/master/docs/admin_interface.md)

## Contributing

All contributions are welcome: ideas, patches, documentation, bug reports,
complaints, and even something you drew up on a napkin.
