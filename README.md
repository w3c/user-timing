User Timing
===========

This specifications defines an interface to help web developers measure the
performance of their applications by giving them access to high precision
timestamps and enable them to do comparisons between them.

* Read the latest draft: https://w3c.github.io/user-timing/
* Discuss on [public-webperf](http://www.w3.org/Search/Mail/Public/search?keywords=%5BUserTiming%5D&hdr-1-name=subject&hdr-1-query=&index-grp=Public_FULL&index-type=t&type-index=public-web-perf)

See also [Web performance README](https://github.com/w3c/web-performance/blob/gh-pages/README.md)

## PerformanceMark

User Timing enables developers to create a `PerformanceMark`, which contains a
provided string in `name`, with a high resolution timestamp (see
[hr-time](https://w3c.github.io/hr-time/) in its `startTime`, and optionally
some additional metadata in its `detail`.
A developer can create such an object via `performance.mark()` and can query
existing entries via `performance.getEntriesByType('mark')` (or other getters;
see [performance-timeline](https://w3c.github.io/performance-timeline/) or via
the `PerformanceObserver`.

The following examples illustrate usage of `performance.mark()` with various
parameters:

* `performance.mark('mark1')`: Creates a `PerformanceMark` whose name is 'mark1'
and whose `startTime` is the current high resolution time.
* `performance.mark('mark2', {startTime: 5.4, detail: det})`: Creates a
`PerformanceMark` whose name is 'mark2', whose `startTime` is 5.4, and whose
`detail` is the object `det`.

### clearMarks

Every time `performance.mark()` is invoked, a new `PerformanceMark` entry needs
to be stored by the browser so that it can be later queried by the developer.
The `performance.clearMarks()` method enables developers to clear some of the
memory used by such calls. In particular:

* `performance.clearMarks()`: Clears all marks from the `Window` or `Worker` from
which the method is invoked.
* `performance.clearMarks('mark1')`: Clears all marks from `Window` or `Worker`
from which the method is invoked, whose name is 'mark1'.

## PerformanceMeasure

User Timing also enables developers to create a `PerformanceMeasure`, which is an
object that represents some time measurement between two points in time, and each
point in time may be represented by a `PerformanceMark`. A `PerformanceMeasure`
has an associated `name`, a high resolution timestamps corresponding to the initial
point in time in `startTime`, the delta between the end-time and the `startTime` in
`duration`, and optionally some additional metadata in its `detail`.
A developer can create a `PerformanceMeasure` entry via `performance.measure()` and
it may query existing entries via `performance.getEntriesByType('measure')` or via
`PerformanceObserver` (i.e. in the same way as for `PerformanceMark`).

The following examples illustrate usage of `performance.measure()` with various
parameters:

* `performance.measure('measure1')`: Creates a `PerformanceMeasure` whose `name` is
'measure1', whose `startTime` is 0, and whose `duration` is the current high
resolution time.
* `performance.measure('measure2', 'requestStart')`: Creates a
`PerformanceMeasure` whose name is 'measure2', whose `startTime` is equal to
`performance.timing.requestStart - performance.timing.navigationStart` (see the
[PerformanceTiming](https://w3c.github.io/navigation-timing/#the-performancetiming-interface)
interface for all the strings that are treated as special values in
`peformance.measure()` calls), and whose end-time is the current high resolution
timestamp --- `duration` will be the delta between the end-time and `startTime`.
* `performance.measure('measure2', 'myMark')`: Creates a `PerformanceMeasure` where
name is 'measure2', where `startTime` is the timestamp from the `PerformanceMark`
whose `name` is `'myMark'`, and where end-time is the current high resolution timestamp.
If there are no marks with such a `name`, an error is thrown. If there are multiple marks
with such a `name`, the latest one is used.
* `performance.measure('measure3', 'startMark', 'endMark')`: Creates a `PerformanceMeasure`
whose `name` is 'measure3', whose `startTime` is the `startTime` of the latest
`PerformanceMark` with `name` 'startMark', and whose end-time is the `startTime` of the
latest `PerformanceMark` with `name` 'endMark'.
* `performance.measure('measure4', 'startMark', 'domInteractive')`: Creates a
`PerformanceMeasure` whose `name` is 'measure4', `startTime` is as in the above example,
and end-time is `performance.timing.domInteractive - performance.timing.navigationStart`.
* `performance.measure('measure5', {start: 6.0, detail: det})`: Creates a
`PerformanceMeasure` whose `name` is 'measure5', `startTime` is 6.0, end-time is the
current high resolution timestamp, and `detail` is the object `det`.
* `performance.measure('measure6', {start: 'mark1', end: 'mark2'})`: Creates a
`PerformanceMeasure` whose `name` is 'measure6', `startTime` is the `startTime` of
the `PerformanceMark` with `name` equal to 'mark1', and end-time is the `startTime`
of the `PerformanceMark` with `name` equal to 'mark2'.
* `performance.measure('measure7', {end: 10.5, duration: 'mark1')`: Creates a
`PerformanceMeasure` where `name` is 'measure7', `duration` is the `startTime` of the
`PerformanceMark` with `name` equal to 'mark1', and `startTime` is equal to
`10.5 - duration` (since end-time is 10.5).
* `performance.measure('measure8', {start: 20.2, duration: 2, detail: det}`: Creates a
`PerformanceMeasure` with `name` set to 'measure8', `startTime` set to 20.2, `duration`
set to 2, and `detail` set to the object `det`.

The following examples would throw errors and hence illustrate incorrect usage of
`peformance.measure`:

* `performance.measure('m', {start: 'mark1'}, 'mark2')`: If the second parameter is a
dictionary, then the third parameter must not be provided.
* `performance.measure('m', {duration: 2.0})`: In the dictionary, one of `start` or `end`
must be provided.
* `performance.measure('m', {start: 1, end: 4, duration: 3})`: In the dictionary, not all
of `start`, `end`, and `duration` should be provided.

### clearMeasures

This is the analogue to `performance.clearMarks()` for `PerformanceMeasure` cleanup:

* `performance.clearMeasures()` clears all `PerformanceMeasure` objects.
* `performance.clearMeasures('measure1')` clears `PerformanceMeasure` objects whose `name`
is 'measure1'.

## Relationship with hr-time

A developer can obtain a high resolution timestamp directly via `performance.now()`, as
defined in [hr-time](https://w3c.github.io/hr-time/#now-method). However, User Timing enables
tracking timestamps that may happen in very different parts of the page by enabling the
developer to use names to identify these timestamps. Using User Timing instead of variables
containing `performance.now()` enables the data to be surfaced automatically by analytics
providers that have User Timing support as well as in the developer tooling of browsers that
support exposing these timings. This kind of automatic surfacing is not possible directly via
HR-Time.

For instance, the following could be used to track the time it takes for a user from the time a
cart is created to the time that the user completes their order, assuming a Single-Page-App
architecture:

```js
// Called when the user first clicks the "Add to cart" button.
function onBeginCart() {
  const initialDetail = // Compute some initial metadata about the user.
  performance.mark('beginCart');
}

// Called after the user clicks on "Complete transaction" button in checkout.
function onCompleteTransaction() {
  const finalDetail = // Compute some final metadata about the user and the transaction.
  performance.measure('transaction', {start: 'beginCart', detail: finalDetail});
}
```

While developers could calculate those time measurements using `performance.now()`, using User
Timing enables both better ergonomics and standardized collection of the results. The latter
enables Analytics providers and developer tools to collect and report site-specific measurements,
without requiring any knowledge of the site or its conventions.
