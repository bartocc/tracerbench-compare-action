# @ember-performance-monitoring/tracerbench-compare-action

Commit over Commit Performance Analysis Automation for Web Applications.

Samples and Analysis are gathered using [Tracerbench](https://github.com/TracerBench/tracerbench) [Compare](https://github.com/TracerBench/tracerbench/tree/master/packages/cli#tracerbench-compare)

## What is it?

Think "Lighthouse CI" but with statistical rigor and more meaningful data.

This library is general enough it could be used to benchmark any Web
 Application with any CI setup via Tracerbench; however, it comes 
 finely tuned for benchmarking Ember applications and Addons via a 
 Github Action.

## Initial Setup

To use this, place markers with `performance.mark(<markerName>)` in your 
application at key points. You can then conigure `tracerbench` to use a
subset (or all) of these markers to create a segmented analysis.

Currently, in order for Tracerbench to know when to stop analyzing your application
 it needs to redirect to `about:blank` after the `Paint` event following the last 
 `marker` you care about. Typically in an Ember Application this means adding the
 following in whichever route is being benchmarked. This constraint may be removed
in the future.

 ```ts
class MyRoute extends Route {
  afterModel() {
      if (document.location.href.indexOf('?tracing') !== -1) {
      endTrace();
    }
  }
}

 function endTrace() {
  // just before paint
  requestAnimationFrame(() => {
    // after paint
    requestAnimationFrame(() => {
      document.location.href = 'about:blank';
    });
  });
}
```

## Usage as a GithubAction

You can use this action by adding it to an existing workflow
 or creating a new workflow in your project.

For example, the below adds a check for the `users` route to all
 pull requests to the master branch.

```yml
name: PerformanceCheck

on:
  pull_request:
    branches:
      - master

jobs:
  analyze-users-route:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
        with:
          fetch-depth: 0
      - uses: ember-performance-monitoring/tracerbench-compare-action@master
        with:
          experiment-url: 'http://localhost:4201/users'
          control-url: 'http://localhost:4200/users'
          markers: 'end-users-model-load'
          regression-threshold: 25
          fidelity: high
```

## Local Usage and Usage with other CI Systems

The GithubAction this project provides is actually a small wrapper that pipes
 the configuration into the project's `main`. This allows for easy use in any 
 setup (local or CI) by adding this action as a dependency.

 ```cli
 yarn add @ember-performance-monitoring/tracerbench-compare-action
 ```

For example, you could mirror the above check in an Ember Application by doing the following.

```json
{
  "experiment-url": "http://localhost:4201/users",
  "control-url": "http://localhost:4200/users",
  "markers": "end-users-model-load",
  "regression-threshold": 25,
  "fidelity": "high"
}
```

```js
const analyze = require('@ember-performance-monitoring/tracerbench-compare-action');
const config = require('./perf-test-config.json');

analyze(config);
```

## Configuration Options

| Option | Default | Description |
| ------ | ------- | ----------- |
| build-control | true | Whether to build assets for the control case |
| build-experiment | true | Whether to build assets for the experiment case |
| control-dist | ./dist-control | The location of the control assets once a build has been performed (or if build-control is false the location they are already) |
| experiment-dist | ./dist-experiment | The location of the experiment assets once a build has been performed (or if build-control is false the location they are already) |
| control-sha | git rev-parse --short=8 origin/master | SHA to be built for the control commit |
| experiment-sha | git rev-parse --short=8 HEAD | SHA to be built for the experiment commit |
| experiment-ref | current branch or tag | The reference being built for the experiment |
| control-build-command | ember build -e production --output-path ${control-dist} | command to execute to build control assets if build-control is true
| experiment-build-command | ember build -e production --output-path ${experiment-dist} | command to execute to build experiment assets if build-experiment is true
| use-yarn | true | When building control/experiment whether to use yarn for install (npm is used otherwise) |
| control-serve-command | ember s --path=${control-dist} | command to execute to serve the control assets |
| experiment-serve-command | ember s --path=${experiment-dist} | command to execute to serve the experiment assets |
| clean-after-analyze | true if experiment-ref is preset, false otherwise | whether to try to restore initial repository state after the benchmark is completed. Useful for local runs. |

## Tracerbench Config

The following options are supplied directly to the `tracerbench compare` command that is run once the 
assets for control and experiment are being served.

| Option | Default | Description |
| ------ | ------- | ----------- |
| control-url | http://localhost:4200?tracing=true | url to benchmark at which the control assets are being served |
| experiment-url | http://localhost:4201?tracing=true | url to benchmark at which the experiment assets are being served |
| fidelity | low | how many runs to perform. high (50) is recommended for CI |
| markers | domComplete | comma separated list of markers to consider in the report |
| runtime-stats | false | whether to analyze chrome runtime stats (expensive and degrades the rigor of the rest of the test) }
| report | true | whether to produce a report PDF |
| headless | true | whether to run Chrome in headless mode |
| regression-threshold | 50 | milliseconds of change at which to fail the test |
