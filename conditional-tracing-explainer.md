# Lightweight and Conditional Tracing

## Contacts
<nrosenthal@chromium.org> or [noamr](https://github.com/noamr)<br>
<guohuideng@microsoft.com> or [guohuideng2024](https://github.com/guohuideng2024)

## Participate

Please join the discussion at this [link](https://github.com/w3c/long-animation-frames/issues/3).

## Goal

This proposal introduces a lightweight and conditional tracing to facilitate investigation into performance incidents such as Long Animation Frames (LoAF).

## User Research

A number of Performance APIs monitor web site performance, but the information provided by these APIs by themselves may not be sufficient to locate the root cause of performance problems.

One example is the [wrapper problem](w3c/long-animation-frames#3) of the LoAF API. When application JavaScript execution causes a LoAF, the [script entry point](https://w3c.github.io/long-animation-frames/#create-script-entry-point) in the `PerformanceLongAnimationFrameTiming` exposes the responsible script entry point. Such a script entry point is often not granular enough, e.g. it can be a React callback that hides all the underlying slow functions, and also creates faux attribution: the wrapper function is "blamed" for slowness when it just passes the execution to an underlying function. A similar issue exists for event timing, where the provided timing attributes don't always capture the whole story, and don't always coincide with a more attributable long animation frame.

The [performance User Timing API](https://www.w3.org/TR/user-timing/) can be used to diagnose performance problems further when additional information is needed to locate the root cause. However, it has some drawbacks:

- The browser doesn't know the performance user timing points are set to diagnose a particular kind of performance incidents, so the browser cannot manage the coresponding data efficiently. The data cannot be filtered properly based on the the absence of performance incidents. Consequently the sites must process and upload more metric data back to the server than needed. This causes considerable performance overhead on the client.  

- The data collected from performance User Timing API is separate from the data collected from perfornance monitoring APIs like LoAF, requiring developers to relate them after careful analysis.

This document proposes a lightweight User Timing API extension specifically designed to support LoAF. Compared to the standard User Timing API, this proposal offers lower runtime overhead and improved developer ergonomics. A similar enhancement could be applied to the Event Timing API in the future.


## API Changes and Example Code

This proposal introduces a new type of object that represents a "track"- a namespace of sorts for low-overhead user measurements.

A track is like a mini performance timeline. The entries on a track are not buffered in the regular performance timeline. Instead, they are buffered and reported by a relevant performance entry type specified by the following:

```js
PerformanceTrack.startConditionalBuffering({entryType: ...});
```

The `PerformanceTrack` can add markers and measure the time between them, similar to the user timing API. The difference is that the track doesn't take the dictionary-type parameters like `detail`, `markOptions` and `measureOptions`.

```js
PerformanceTrack.mark(Name);
PerformanceTrack.measure(Name, startMark/*optional*/, endMark/*optional*/)
```
Apart from the entry name, all other annotations are per-track rather than per-entry, to avoid overhead.

Unlike the User Timing API, neither `mark` nor `measure` returns a `PerformanceMark` or a `PerformanceMeasure` entry. These `mark`s and `measure`s are tracked for the relevant performance incidents(such as LoAF) only. Therefore they are only provided in the relevant PerformanceEntry(i.e., `PerformanceLongAnimationFrameTiming`) if they occur during a LoAF.


In addition, to include these entries, the `PerformanceTrack` must be explicitly attached to the `PerformanceObserer` that observes the relevant Performance entries like this:

```js
PerformanceObserver.attachTrack(track);
```

We report the tracing points with `TrackMark` and `TrackMeasure` entries. They are similar to `PerformanceMark` and `PerformanceMeasure` entries. But they don't have a `detail` field. Instead it has an additional `trackName` field to indicate which track the tracing point is from.

A new field, `trackEntries` is added to `PerformanceLongAnimationFrameTiming` interface. It's an array consisting of relevant `TrackMark` and `TrackMeasure` entries that occurs during a LoAF.


### Sample code

```js
const track0 = new PerformanceTrack("framework0");
const track1 = new PerformanceTrack("framework1");

// This buffers new entries for not-yet-created LoAF observers.
track0.startConditionalBuffering({entryType: "long-animation-frame"});
track1.startConditionalBuffering({entryType: "long-animation-frame"});

// At some point, add a mark. This mark does not appear in the performance timeline
track0.mark("mark1");   // at time t1
// At some other point, add another mark
track0.mark("mark2");   // at time t2
// Combines mark->mark as a single entry with duration. This measure does not appear in the performance timeline.
track0.measure("myMeasure", "mark1", "mark2");

// Same with the other track.
// The mark and measure points are on |track1|, not the ones on
// |track0|.
track1.mark("mark1");    // at time t3
//...
track1.mark("mark2");    // at time t4
//...
track1.measure("myMeasure", "mark1", "mark2");

// Observe long animation frame entries, and print out the
// conditional tracing results.
const observer = new PerformanceObserver(entries => {
    for (const loaf : entries.getEntriesByType("long-animation-frame") {
      // This will have all the marks & measures from the attached tracks that occurs during LoAF.
      for (const trackEntry of loaf.trackEntries) {
        if( trackEntry.entryType === "mark"){
          console.log(trackEntry.entryType, trackEntry.trackName, trackEntry.name, trackEntry.startTime);
        }
        else if ( trackEntry.entryType == "measure"){
          console.log(trackEntry.entryType, trackEntry.trackName, trackEntry.name, trackEntry.duration);
        }
      }
    }
});
observer.observe({entryType: "long-animation-frame"});

// This adds "tracks" to the LoAF entries observed by `observer`.
observer.attachTrack(track0);
observer.attachTrack(track1);

/* Assuming we have a LoAF, during which all the mark and measure points are executed once
at the time indicated in the comment, The output would be:
"TrackMark", "frameWork0", "mark1", t1
"TrackMark", "frameWork0", "mark2", t2
"TrackMeasure", "frameWork0", "myMeasure", t2-t1
"TrackMark", "frameWork1", "mark1", t3
"TrackMark", "frameWork1", "mark2", t4
"TrackMeasure", "frameWork1", "myMeasure", t4-t3
*/
```
### Notes on the API design
We use an object rather than something like `performance.trace` for the following reasons:
- It allows adding detailed annotations without having to create a dictionary during performance-critical execution.
- It creates a natural namespace: both the reporter and the observer have to access the same object, rather than rely on (leaky) strings
- From an ergonomic perspective, adding a global `performance.*` function might be confusing as the result doesn't end up in the performance timeline directly.

## Alternatives Considered

[User-defined script entry point (UDSEP)](https://github.com/w3c/long-animation-frames/blob/main/user-defined-script-entry-point-explainer.md) is another way to annotate code for LoAF. UDSEP monitors script entry point enter and exit timing and keeps track of nesting entry points as well as time spent by microtasks that are not annotated but dispatched within UDSEP. This approach requires [an efficient V8 extension](https://docs.google.com/document/d/1wEXU0nv8DhzN7XpfmlfbRcbsJ2JNWjfAF-3J7qtWiEw/edit?usp=sharing) that allows Blink to be notified at the microtask enter and exit points. Due to the complexty involved, we prefer this conditional tracing approach.

## Future extention to this proposal

We can apply the same enhancement to event timing in the future.

```js
track.startConditionalBuffering({entryType: "event"});
```

## Stakeholder Feedback/Opposition
TBD.

## Security/Privacy Considerations
The information exposed by this API is similar to what the User Timing API exposes. The difference is that the data exposed by this API is filtered by performance incidents.

### [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

>1.	What information does this feature expose, and for what purposes?

This feature extends the existing performance User Timing API. The app can add `mark` and `measure` points in a similar fasion but the `PerformanceMark` and `PerformanceMeasure` entries can now be specified for a particular kind of performance incidents, so the UA filters out unrelevant entries and report the relevant ones in the corresponding `PerformanceEntry`.

The purpose of this new feature is to improve the computational efficiency and ergonomics of the User Timing API when diagnosing the cause of LoAF.

>2.	Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

Yes
>3.	Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No.
>4.	How do the features in your specification deal with sensitive information?

It does not deal with sensitive information.
>5.	Does data exposed by your specification carry related but distinct information that may not be obvious to users??

No.
>6.	Do the features in your specification introduce state that persists across browsing sessions?

No.
>7.	Do the features in your specification expose information about the underlying platform to origins?

No.
>8.	Does this specification allow an origin to send data to the underlying platform?

No.
>9.	Do features in this specification enable access to device sensors?

No.
>10.	Do features in this specification enable new script execution/loading mechanisms?

No.
>11.	Do features in this specification allow an origin to access other devices?

No.
>12.	Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.
>13.	What temporary identifiers do the features in this specification create or expose to the web?

No.
>14.	How does this specification distinguish between behavior in first-party and third-party contexts?

No distinction.
>15.	How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

No difference.
>16.	Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes.
>17.	Do features in your specification enable origins to downgrade default security protections?

No.
>18.	What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

No difference.

>19.	What happens when a document that uses your feature gets disconnected?

No difference.
>20.	Does your spec define when and how new kinds of errors should be raised?

No new errors should be raised.
>21.	Does your feature allow sites to learn about the users use of assistive technology?

Not directly.
>22.	What should this questionnaire have asked?

Nothing else.
