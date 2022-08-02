- Feature Name: trace_end_at_lcp_option
- Start Date: 2022-7-26
- RFC PR: https://github.com/TracerBench/tracerbench/pull/430
- TracerBench Issue:

# Summary
[summary]: #summary
This proposal allows the user to configure **largestContentfulPaint** or **largestContentfulPaint::Candidate** as a marker in the existing **markers** config options. If the user chooses to add **largestContentfulPaint** as the last marker, during the benchmark the tracing ends when PerformanceObserver receives a callback from the **largest-contentful-paint** event. When the users don't provide **markers** configuration, currently we add **navigationStart** and **loadEventEnd** in the **markers** configuration by default. This proposal will change the default to add **navigationStart** and **largestContentfulPaint::Candidate**. If the tracing ends at LCP, the paint phase duration will be the difference between the LCP event end and the marker prior to LCP's start. If the last marker in markers array is a regular marker, we require the users to configure a marker that can trigger rendering (paint event), otherwise, we throw an exception with a proper message.

We will enhance **record-har** command's existing option **--marker** to accept **largestContentfulPaint** and make **largest-contentful-paint** event as recording end signal.

# Motivation
[motivation]: #motivation

Some single-page apps choose to lazy render components to achieve faster **loadEventEnd** metrics. The page might not have a significant amount of content for users to consume. It's not a good user experience. Before this proposal, to measure users' actual performance experience, the developers need to instrument their page's main component with a marker that reflects the render complete. Then add the marker as the last element in tbconfig.json's markers array or pass it in from command-line **--markers** options. This proposal reduces the developer's effort. We will take the advantage of chrome's performance timeline API and observe **largest-contentful-paint** event callback to signal trace ends. We will use the end of LCP to calculate the last paint phase from the prior marker's start. By doing this, developers don't need to instrument their page with a rendering marker or configure the **markers** array, they can still get benchmark results that reflect their user experience.

Also, using LCP as a trace end can avoid us to generate an artificial paint event for benchmarking static pages. See this [PR](https://github.com/TracerBench/tracerbench/pull/336) tries to address [this issue](https://github.com/TracerBench/tracerbench/issues/305).

# Pedagogy
[pedagogy]: #pedagogy

## Configuration Change

The users can add **largestContentfulPaint** as the last marker in tbconfig.json.

```jsonc title="Markers Configuration Example" showLineNumbers
  {
    ...
     "markers": [
    {
      "start": "navigationStart",
      "label": "load"
    },
    {
      "start": "mark_app_start",
      "label": "boot"
    },
    {
      "start": "mark_transition_start",
      "label": "transition"
    },
    {
      "start": "mark_transition_end",
      "label": "render"
    },
    {
      "start": "largestContentfulPaint",
      "label": "paint"
    }
  ],
  ...
}
```

## Cli Commands Enhancement

### compare command

The users can provide **markers** as a comma-separated string from the command-line option. The string is the marker name and label name pair. The last marker label defaults to **paint**. Therefore, no need to provide a label.

```scripts --markers option example
  tracerbench compare --controlURL http://localhost:3000/release/index.html --experimentURL http://localhost:3001/regression/index.html --fidelity 10 --tbResultsFolder ./tbResult --config ./tbResult/tbconfig.json --regressionThreshold 50 --markers 'navigationStart,load,jqueryLoaded,jquery,emberLoaded,application,startRouting,routing,willTransition,transition,largestContentfulPaint' --headless
```

### record-har command

The users can provide **largestContentfulPaint** as the value of the **--marker** option

```scripts title="--marker option example" showLineNumbers
tracerbench record-har --config=./testHar/tbconfig.json --cookiespath=./testHar/cookie.json
--headless --dest=./testHar --filename=myHarFile --url=https://www.tracerbench.com --marker=largestContentfulPaint
```

# Details

[details]: #details

## How to end trace at largest-contentful-paint

For the **compare** command's tracing, I will inject the following script into the page via the chrome dev tool protocol API "Page.addScriptToEvaluateOnLoad". The script will resolve the promise when the **largest-contentful-paint** event is observed after the prior configured marker.

```javascript title="Inject promise for largest-contentful-paint event" showLineNumbers
var __tracerbenchPriorMarkerObserved = (typeof ${priorMarker} === 'undefined')? true : false;
    if (!__tracerbenchPriorMarkerObserved){
      new PerformanceObserver((entryList, observer) => {
        if (!__tracerbenchPriorMarkerObserved) {
          var markerEntries = entryList.getEntriesByName(${priorMarker});
          if (markerEntries.length > 0) {
            __tracerbenchPriorMarkerObserved = true;
            observer.disconnect();
          }
        }
      }).observe({ type: 'mark' });
    }
    var __tracerbenchLCP =
      self === top &&
      opener === null &&
      new Promise((resolve) =>
        new PerformanceObserver((entryList, observer) => {
          var lcpEntries = entryList.getEntriesByType('largest-contentful-paint');
          if (lcpEntries.length > 0 && __tracerbenchPriorMarkerObserved) {
            requestAnimationFrame(() => {
              resolve();
            });
          }
          observer.disconnect();
        }).observe({ type: 'largest-contentful-paint', buffered: true })
      );
  ```

I will make the trace end wait for the above promise to fulfill if the user configures the **largestContentfulPaint** as the last marker.
For the **record-har** command, I will use a similar technique as above. To wait for LCP before end har recording.

## Trace events' processing changes

Currently, during trace event processing, we will throw an exception if we can't find the paint event after the last marker. This proposal will search for the event name  **largestContentfulPaint::Candidate**** instead if the last configured marker is **largestContentfulPaint**.

Since **largest-contentful-paint** is a point in the performance timeline, we won't double count the phase duration. The **paint** phase will be calculated as the LCP end minus the prior marker start.

# Critique

[critique]: #critique
This proposal derives the users' trace ending preference from the last marker in the configuration of **markers**. Depending on the users' feedback, we can add an explicit CLI option.

In the next step, we can explicitly add standalone LCP data into the tracerbench report for convenience. Currently, it's implied as the last phase end time. The report format update is not discussed in this RFC.
