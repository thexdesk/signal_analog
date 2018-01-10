# Signal Analog

A troposphere-like library for building and composing SignalFx SignalFlow
programs into Charts, Dashboards, and Detectors.

The rest of this document assumes familiarity with the SignalFx API, SignalFlow
language, and Chart/Dashboard models.

For more information about the above concepts consult the
[upstream documentation].

If you're looking for pre-built dashboards for existing application frameworks
or tools then please consult the ***REMOVED*** documentation.

## TOC

  - [Features](#features)
      - [Planned Features](#planned-features)
  - [Installation](#installation)
  - [Usage](#usage)
      - [Building Charts](#charts)
      - [Building Dashboards](#dashboards)
          - [A note on Dashboard names](#dashboard-names)
      - [Updating Dashboards](#dashboards-updates)          
      - [Talking to the SignalFlow API Directly](#signalflow)
  - [Contributing](#contributing)
  - [Credits](#credits)

<a name="features"></a>
## Features

  - Provides bindings for the SignalFlow DSL
  - Provides abstractions for building Charts
  - Provides abstractions for building Dashboards

<a name="planned-features"></a>
### Planned Features

  - High-level constructs for building Detectors

<a name="installation"></a>
## Installation

Add `signal_analog` to the requirements file in your project:

```
# requirements.txt
# ... your other dependencies
signal_analog
```

Then run the following command to update your environment:

```
pip install -r requirements.txt
```

If you are unable to install the package then you may need to check your Python
configuration so that it conforms to Nike's environment. Consult the
following documentation for more info:

***REMOVED***

<a name="usage"></a>
## Usage

<a name="charts"></a>
### Building Charts

`signal_analog` provides constructs for building charts in the
`signal_analog.charts` module.

Consult the [upstream documentation][charts] for more information on the
Chart API.

Let's consider an example where we would like to build a chart to monitor
memory utilization for a single Riposte applicaton in a single environment.

Riposte reports metrics for application name as `app` and environment as `env`
with memory utilization reporting via the `memory.utilization` metric name.

In a timeseries chart, all data displayed on the screen comes from at least one
`data` definition in the SignalFlow language. Let's begin by defining our
timeseries:

```python
from signal_analog.flow import Data

ts = Data('memory.utilization')
```

In SignalFlow parlance a timeseries is only displayed on a chart if it has been
"published". All stream functions in SignalFlow have a `publish` method that
may be called at the _end_ of all timeseries transformations.

```python
ts = Data('memory.utilization').publish()
```

As a convenience, all transformations on stream functions return the callee,
so in the above example `ts` remains bound to an instance of `Data`.

Now, this timeseries isn't very useful by itself; if we attached this program
to a chart we would see _all_ timeseries for _all_ Riposte applications
reporting to SignalFx.

We can restrict our view of the data by adding a filter on application name:

```python
from signal_analog.flow import Data, Filter

app_filter = Filter('app', '***REMOVED***')

ts = Data('memory.utilization', filter=app_filter).publish()
```

Now if we created a chart with this program we would only be looking at metrics
that relate to the `***REMOVED***` application. Much better, but we're still
looking at instance of `***REMOVED***` _regardless_ of the environment it
lives in.

What we'll want to do is combine our `app_filter` with another filter for the
environment. The `signal_analog.combinators` module provides some helpful
constructs for achieving this goal:

```python
from signal_analog.combinators import And

env_filter = Filter('env', 'prod')

all_filters = And(app_filter, env_filter)

ts = Data('memory.utilization', filter=all_filters).publish()
```

Excellent! We're now ready to create our chart.

First, let's give our chart a name:

```python
from signal_analog.charts import TimeSeriesChart

memory_chart = TimeSeriesChart().with_name('Memory Used %')
```

Like it's `flow` counterparts, `charts` adhere to the builder pattern for
constructing objects that interact with the SignalFx API.

With our name in place, let's go ahead and add our program:

```python
memory_chart = TimeSeriesChart().with_name('Memory Used %').with_program(ts)
```

Each Chart understands how to serialize our SignalFlow programs appropriately,
so it is sufficient to simply pass in our reference here.

Finally, let's change the plot type on our chart so that we see solid areas
instead of flimsy lines:

```python
from signal_analog.charts import PlotType

memory_chart = TimeSeriesChart()\
                 .with_name('Memory Used %')\
                 .with_program(ts)
                 .with_default_plot_type(PlotType.area_chart)
```

[Terrific]; there's only a few more details before we have a complete chart.

In order for any chart to be created we must provide an API token. Contact
your account administrator for the best way to access your account's API
tokens. If you are unsure who your account administrator is consult
[this document to determine the appropriate contact][sfx-contact].

With API token in hand we can now create our chart in the API:

```python
response = memory_chart.with_api_token('my-api-token').create()
```

As of version `0.3.0` one of two things will happen:

  - We receive some sort of error from the SignalFx API and an exception
  is thrown
  - We successfully created the chart, in which case the JSON response is
  returned as a dictionary.

From this point forward we can see our chart in the SignalFx UI by navigating
to https://app.signalfx.com/#/chart/v2/\<chart\_id\>, where `chart_id` is
the `id` field from our chart response.

<a name="dashboards"></a>
### Building Dashboards

`signal_analog` provides constructs for building charts in the
`signal_analog.dashboards` module.

Consult the [upstream documentation][dashboards] for more information on the
Dashboard API.

Building on the examples described in the previous section, we'd now like to
build a dashboard containing our memory chart.

We start with the humble `Dashboard` object:

```python
from signal_analog.dashboards import Dashboard

dash = Dashboard()
```

Many of the same methods for charts are available on dashboards as well, so
let's give our dashboard a memorable name and configure it's API token:

```python

dash.with_name('My Little Dashboard: Metrics are Magic')\
    .with_api_token('my-api-token')
```

See the [note below](#dashboard-names) for caveats on naming dashboards.

Our final task will be to add charts to our dashboard and create it in the API!

```python
response = dash.with_charts(memory_chart).create()
```

At this point one of two things will happen:

  - We receive some sort of error from the SignalFx API and an exception
  is thrown
  - We successfully created the dashboard, in which case the JSON response is
  returned as a dictionary.

<a name="dashboard-names"></a>
#### A note on Dashboard names

`signal_analog` assumes that dashboard names are unique in SignalFx. While at
the time of this writing uniqueness is _not_ enforced by the `v2` API, this
convention _is_ enforced by this library.

We make this assumption so that we avoid easily creating duplicate dashboards
and makes updating existing resources easier without having to manage extra
state outside of the SignalFx API.

<a name="dashboards-updates"></a>
### Updating Dashboards
Once you have dashboard created, you can update the properties like name and descriptions of a dashboard
    
**Note**: For now, you can only update the dashboard name and description and *not* charts. 
Updating charts is coming next and you can track that work [here](https://jira.nike.com/browse/SIP-1035) 

```python
dash.update(name='updated_dashboard_name', description='updated_dashboard_description')
```
At this point one of two things will happen:

  - We receive some sort of error from the SignalFx API and an exception
  is thrown
  - We successfully updated the dashboard, in which case the JSON response is
  returned as a dictionary with the updated properties.

<a name="signalflow"></a>
### Talking to the SignalFlow API Directly

If you need to process SignalFx data outside of their walled garden it may be
useful to call the SignalFlow API directly. Note that you may incur time
penalties when pulling data out due to SignalFx's architecture.

SignalFlow constructs are contained in the `flow` module. The following is an
example SignalFlow program that monitors Riposte RPS metrics for the `foo`
application in the `test` environment.

```python
from signal_analog.flow import Data, Filter
from signal_analog.combinators import And

all_filters = And(Filter('env', 'prod'), Filter('app', '***REMOVED***'))

program = Data('requests.count', filter=all_filters)).publish()
```

You now have an object representation of the SignalFlow program. To take it for
a test ride you can use the official SignalFx client like so:

```python
# Original example found here:
# https://github.com/signalfx/signalfx-python#executing-signalflow-computations

import signalfx
from signal_analog.flow import Data, Filter
from signal_analog.combinators import And

app_filter = Filter('app', '***REMOVED***')
env_filter = Filter('env', 'prod')
program = Data('requests.count', filter=And(app_filter, env_filter)).publish()

with signalfx.SignalFx().signalflow('MY_TOKEN') as flow:
    print('Executing {0} ...'.format(program))
    computation = flow.execute(str(program))

    for msg in computation.stream():
        if isinstance(msg, signalfx.signalflow.messages.DataMessage):
            print('{0}: {1}'.format(msg.logical_timestamp_ms, msg.data))
        if isinstance(msg, signalfx.signalflow.messages.EventMessage):
            print('{0}: {1}'.format(msg.timestamp_ms, msg.properties))
```

<a name="contributing"></a>
## Contributing

Consult the [docs here for more info about contributing](CONTRIBUTING.md).

Activity diagrams for this project are located in the `docs/activity` directory
and can be generated locally by running the `make activity_diagrams` task in
the root of the project. If you don't have `plantuml` installed you can do
so via Homebrew (`brew install plantuml`).

<a name="credits"></a>
## Credits

This package was created with
[Cookiecutter](https://github.com/audreyr/cookiecutter) and the
[nik/nike-python-template](***REMOVED***)
project template.

[upstream documentation]: https://developers.signalfx.com/docs/signalflow-overview
[charts]: https://developers.signalfx.com/reference#charts-overview-1
[sfx-contact]: https://confluence.nike.com/x/GlHiCQ
[terrific]: https://media.giphy.com/media/jir4LEGA68A9y/200.gif
[dashboards]: https://developers.signalfx.com/v2/reference#dashboards-overview
***REMOVED***: https://bitbucket.nike.com/projects/NIK/repos/***REMOVED***/browse
