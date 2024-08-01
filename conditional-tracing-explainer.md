# Explainer: conditional tracing

# Overview
Recently, there were two discussions that seem separate but might end up being related to the same use case.

## More granular information in Long Animation Frames & event timing
Context: w3c/long-animation-frames#3

A script entry point is often not granular enough, e.g. it can be a React callback that hides all the underlying slow functions,
and also creates faux attribution: the wrapper function is "blamed" for slowness when it just passes the execution to an underlying function.
A similar issue exists for event timing, where the provided timing attributes don't always capture the whole story, and don't always coincide with a more attributable long animation frame.

## Annotated user timing in devtools
Context: #111. 

Chromium devtools already provides a way to annotate user timing entries by adding details to the user timing `detail` attribute.
Apart from the desire to make those web-exposed annotations somewhat interoperable/standard (kind of like the `console` APIs), a question was raised about the efficiency of user timing entries for the purpose of tracing at high granularity.
Creating user timing entries (mark/measure) has several issues:
- Those functions receive a dictionary with each call, causing a GC load
- User timing entries are buffered in an infinite array by default.
- User timing entries have one global namespace, making it difficult for separate libraries to use them without stepping on each other's toes.

# Proposal: conditional tracing using a `PerformanceTrack`

This proposal is to have a new type of object, that represents a "track" - a namespace of sorts for low-overhead user measurements.
A track is like a mini performance timeline, with the following characteristics:
- Entries are not buffered in the regular performance timeline
- Relevant entry types (`long-animation-frame`, `event`) can buffer and report track entries that are reported during the lifetime of their corresponding entries
- Apart from the entry name, all the other annotations are per-track rather than per-entry, to avoid overhead.
- All devtools-specific annotations are done using the `console` object.

## Shape:

```js
const track = new PerformanceTrack("framework");

// At some point, add a mark. This mark does not appear in the performance timeline
track.mark(name);

// At some other point, add another mark
track.mark(name);

// Combines mark->mark as a single entry with duration. This measure does not appear in the performance timeline.
track.measure(name);

// Adds devtools-specific information about a given mark/measure
console.describe(trace.mark(name), details);

// Returns a function that calls inner_function and measures it duration.
// Addeded to LoAF scripts as an entry point
track.bind(inner_function, name, [additional_args]);

const observer = new PerformanceObserver(entries => {
    for (const loaf : entries.getEntriesByType("long-animation-frame") {
      // This will have all the marks & measures from the attached tracks
      for (const trackEntry of loaf.trackEntries) {
        // report or use the track entry for attribution
      }
      
      // This will have now also include functions captured with track.bind
      loaf.scripts
    }
});
observer.observe({entryType: "long-animation-frame"});

// This adds "tracks" to the LoAF entries observed by `observer`.
observer.attachTrack(track);

// Gives devtools additional information about this track
console.describeTrack(track, { color: "blue" });
```

## Some notes:
We use an object rather than something like `performance.trace` for the following reasons:
- It allows adding detailed annotations without having to create a dictionary during performance critical execution.
- It creates a natural namespace: both the reporter and the observer have to access the same object, rather than rely on (leaky) strings
- From an ergonomic perspective, adding a global `performance.*` function might be confusing as the result doesn't end up in the performance timeline directly.
- Using the `bind` function doesn't guarantee attribution, because it doesn't include microtasks (unlike script entry points). This needs to be handled with care when using it for "blame"
- It might look confusing to mix web-facing and `console` APIs in the same function, however it's intentional as devtools is an "augmentation" here.

  
