- Feature Name: trace_end_at_lcp_option
- Start Date: 2022-7-26
- RFC PR:
- TracerBench Issue:

# Summary
[summary]: #summary

I will add configuration and command-line options to explicitly allow users to choose end tracing at LCP when PerformanceObserver receives a callback from the **largest-contentful-paint** event. This additional option will be added to the **compare** and **record-har** cli commands. It will overwrite the default behavior of the end trace at the last marker in the **markers** array configuration. Note, even users don't provide **markers** configuration, currently we add **navigationStart** and **loadEventEnd** in the **markers** configuration programatically. So **loadEventEnd** is the default trace end condition. This enhancement will be backward compatible.

# Motivation
[motivation]: #motivation

Some single-page apps choose to lazy render components in order to achieve faster **loadEventEnd** metrics. The page might not have a significant amount of content for users to consume. It's not a good user experience. Before this proposal, to measure users' actual performance experience, the developers need to instrument their page's main component with a marker that reflects the render complete. Then add the marker into tbconfig.json's markers array or pass it in from command-line options. This is a less optimal developer experience. To simplify the goal of measuring actual user experience, I will take the advantage of chrome's performance timeline API and add the options to signal trace end at **largest-contentful-paint** event callback. By doing this, even if developers don't instrument their page with a proper marker or configure the **markers** array, they can still get benchmark results that reflect their user experience.

Also, using LCP as trace end can avoid us to generate artificial paint event like this [PR](https://github.com/TracerBench/tracerbench/pull/336) in order to address [this issue](https://github.com/TracerBench/tracerbench/issues/305).

# Pedagogy
[pedagogy]: #pedagogy

## Configuration Change
I will add the following optional configurations into tbconfig.json.
```
  {
    //default to false, if it's true
    //it overrides the default loadEventEnd or the last marker in markers array as trace end
    traceEndAtLcp?: boolean;
    //the regex pattern to identify the LCP candidate element user want to measure, if undefined, I use the first LCP candidate
    lcpRegex?: string;
  }
```

## Cli Change
### compare command
I will add the following two command-line options to the **tracerbench compare** command. The page load's tracing will stop at the desired largestContentfulPaint event.
```
  //type of boolean, default to false
  --traceEndAtLcp
  //type of string for regex pattern to match the desired largestContentfulPaint candidate. if undefined, the first one is chosen.
  --lcpRegex
```

### record-har command
I will add the following two command-line options to the **tracerbench record-har** command. The page har recording will stop at the desired largestContentfulPaint event.
```
  //type of boolean, default to false
  --traceEndAtLcp
  //type of string for regex pattern to match the desired largestContentfulPaint candidate. if undefined, the first one is chosen.
  --lcpRegex
```
# Details
[details]: #details
## How to end trace at largest-contentful-paint
For the **compare** command's tracing, I will inject the following script into page via the chrome dev tool protocol API "Page.addScriptToEvaluateOnLoad".
```
var __tracerbenchLCP =
      self === top &&
      opener === null &&
      new Promise((resolve) =>
        new PerformanceObserver((entryList, observer) => {
          var entries = entryList.getEntries();
          var pattern = 'theElementRegexPatternUserConfig';
          for (var i = 0; i < entries.length; i++) {
            if ( pattern !== 'undefined') {
              var elm = entries[i].element.outerHTML;;
              var regex = new RegExp(pattern);
              if (regex.test(elm)){
                requestAnimationFrame(() => {
                  resolve();
                });
              }
            } else {
              requestAnimationFrame(() => {
                resolve();
              });
              break;
            }
          }
          observer.disconnect();
        }).observe({type: 'largest-contentful-paint', buffered: true})
      );;
  ```
I will make the trace end wait for the above promise to fulfill if the user configures the **traceEndAtLcp** to true.
For the **record-har** command, I will use a similar technique as above. To wait for LCP before end har recording.

## How to identify a specific largestContentfulPaint candidate
I want to take an iterative approach on how to allow the user to identify a specific largestContentfulPaint candidate. In the first pass, I will make it generic. The user can provide **lcpRegex** configuration, a regex pattern string that can match the HTML content of the largestContentfulPaint candidate's element. Any match will resolve the promise above. If **lcpRegex** is undefined, the first largestContentfulPaint candidate will end tracing. Depending on the user's feedback, I can make it more restrictive to match using the following rules
1. the top-level element's id. Downside, the HTML of the element needs to have a id.
2. the element's CSS selector. Downside, different selector combination is possible, it's harder for the user to identify the correct selector that matches the largestContentfulPaint candidate that chrome reports.

Other possibility is to add **lcpSize: number** configuration. The user just need to specify the ideal element size, if a candidate is larger than the configured size, the trace can end.

# Critique
[critique]: #critique
I have explored the possibility of simulating a marker e.g. **largestContentfulPaint** when the **largest-contentful-paint** event is observed. This could reuse the existing **waitForMark** promise as the trace end condition. However, I found injecting a marker using **performance.mark** API inside the **PerformanceObserver callback** is not possible.


# Unresolved questions
[unresolved]: #unresolved-questions
I don't understand why **performance.mark** doesn't work inside **PerformanceObserver callback**. I choose a different approach as described in the **Details** section that doesn't add much coding overhead and works. This doesn't block the PR from merging.
