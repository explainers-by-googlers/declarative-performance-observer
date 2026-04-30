# Explainer for the Declarative Performance Observer

This proposal is an early design sketch by the Chrome loading team to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Shunya Shishido (sisidovski@chromium.org)

## Participate

- Discussion: [Issue tracker on GitHub](https://github.com/explainers-by-googlers/declarative-performance-observer/issues)

## Table of Contents

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
  - [Use case 1: Detecting Early Network Failures](#use-case-1-detecting-early-network-failures)
  - [Use case 2: Measuring Application Journeys Terminated by OOM Crashes](#use-case-2-measuring-application-journeys-terminated-by-oom-crashes)
  - [Use case 3: Reliable Session End Recording](#use-case-3-reliable-session-end-recording)
- [Potential Solution](#potential-solution)
  - [Syntax](#syntax)
  - [Report Format](#report-format)
  - [Example Report Payload](#example-report-payload)
  - [Report Timing (Session Termination)](#report-timing-session-termination)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
- [Detailed design discussion](#detailed-design-discussion)
  - [Deferred Reporting for Early Failures](#deferred-reporting-for-early-failures)
  - [Explicit `PerformanceSessionEndTiming` vs spec minimalism](#explicit-performancesessionendtiming-vs-spec-minimalism)
- [Considered alternatives](#considered-alternatives)
  - [Network Error Logging (NEL)](#network-error-logging-nel)
  - [ServiceWorker](#serviceworker)
  - [Combination of Existing Technologies (NEL + JS PerformanceObserver + fetchLater() + Renderer Crash Reporting)](#combination-of-existing-technologies-nel--js-performanceobserver--fetchlater--renderer-crash-reporting)
  - [Extend performance.mark() instead of include-user-timing](#extend-performancemark-instead-of-include-user-timing)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
  - [Denial of Service (DoS) Prevention (Memory)](#denial-of-service-dos-prevention-memory)
  - [Disk Quota for Buffered Reports (Storage DoS)](#disk-quota-for-buffered-reports-storage-dos)
  - [Safe API Design and Telemetry Hijacking](#safe-api-design-and-telemetry-hijacking)
  - [Telemetry Pollution by Third-Party Scripts](#telemetry-pollution-by-third-party-scripts)
  - [Timer Resolution and Side-Channels](#timer-resolution-and-side-channels)
  - [Reporting Endpoint Security](#reporting-endpoint-security)
  - [Data Minimization and Opt-in](#data-minimization-and-opt-in)
  - [Cross-Site Correlation and Storage Partitioning](#cross-site-correlation-and-storage-partitioning)
  - [Incognito / Private Browsing Mode](#incognito--private-browsing-mode)
  - [Crash Detection Heuristic vs. Explicit Field](#crash-detection-heuristic-vs-explicit-field)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

This proposal aims to fill a gap in web observability regarding the full lifecycle of user journeys in scenarios where current APIs have limited visibility. Existing APIs, such as [PerformanceObserver](https://w3c.github.io/performance-timeline/#performanceobserver) in JavaScript, are inherently bound to the execution environment of the page. Consequently, some early abandoned navigations or failures are not captured:

1. **JavaScript Dependency:** If a navigation fails early (e.g., DNS timeout, connection refused), no JavaScript runs, and the site remains unaware of the failure, making it impossible to calculate accurate success rates for user journeys.
1. **Reporting Reliability:** Existing methods to ensure all telemetry data is sent before a page disappears are complex and often unreliable. Beacons sent during abrupt terminations (like renderer crashes or tab closures on mobile) are often lost.
1. **End-of-Session Detection:** Sites cannot reliably detect when a user leaves a page. `unload` handlers are deprecated because they break BFCache, and on mobile OSs, renderer processes are frequently killed under memory pressure without invoking any JavaScript handlers.

Currently, the web platform does not allow websites to reliably record user journeys from navigation start to page close. This document proposes the **Declarative Performance Observer**, a browser-based telemetry system activated via a declarative HTTP Response Header. It instructs the browser to capture performance metrics and application events out-of-band and report them reliably at session termination via the [Reporting API](https://w3c.github.io/reporting).

## Goals

- Capture end-to-end user journey events from the navigation start to the page close without relying on JavaScript execution.
- Capture custom application-defined events during the page lifecycle.
- Automatically and reliably dispatch a consolidated report of these events when the session ends, surviving renderer crashes and early network failures.

## Non-goals

- Replacing the JavaScript `PerformanceObserver` API.
- Replacing existing APIs to report errors, e.g., [Network Error Logging](https://www.w3.org/TR/network-error-logging/) or Crash Reporting.

## User research

[If any user research has been conducted to inform your design choices,
discuss the process and findings. User research should be more common than it is.]

## Use cases

### Use case 1: Detecting Early Network Failures

A web application needs to know how many users attempt to visit a critical page but fail due to DNS errors or TLS negotiation timeouts before any payload is delivered. Because JavaScript never loads, traditional analytics miss 100% of these users. While Network Error Logging (NEL) can capture network errors, it lacks the ability to correlate these failures with the full application context or specific user journeys.

### Use case 2: Measuring Application Journeys Terminated by OOM Crashes

Pages routinely suffer from Out-Of-Memory (OOM) crashes on low-end mobile devices. Traditional unload or pagehide beaconing fails because the OS terminates the process abruptly. The developer needs to see the sequence of performance events (e.g., FCP, LCP, or custom marks) immediately preceding the crash to debug the cause.

### Use case 3: Reliable Session End Recording

A developer wants to accurately track the total time a user spends on a page. While pagehide events can handle this in many cases, they may fail to run during abrupt terminations or crashes, leaving the session end time invisible.

## Potential Solution

We propose a new declarative HTTP Response Header: `Performance-Observer`. This header integrates with the Reporting API.

### Syntax

```
Performance-Observer: report-to="telemetry"; entry-types=("navigation" "mark" "visibility-state"); include-user-timing=("hero-image-loaded" "next-link-clicked"); capture-early-failures=true
Reporting-Endpoints: telemetry="https://log.com/v1"
```

- **report-to:** Routes the payload to the designated endpoint defined in `Reporting-Endpoints`.
- **entry-types:** Indicates which built-in performance events or visibility events should be recorded (e.g., "navigation", "mark", "visibility-state", matching [PerformanceObserver.supportedEntryTypes](https://w3c.github.io/performance-timeline/#supportedentrytypes-attribute)).
- **include-user-timing:** Specifies an allowlist of user-defined `performance.mark()` or `performance.measure()` events to sync to the browser.
- **capture-early-failures:** A boolean directive. If enabled, the browser persists an origin-level flag to record future early navigation failures.

### Report Format

The payload is a JSON array of reports, delivered via the Reporting API (**application/reports+json**). Each report contains an `entries` array with [PerformanceEntry](https://w3c.github.io/performance-timeline/#performanceentry) objects.

When network errors or early abandons happen before the response is complete, they are reported as synthesized [PerformanceNavigationTiming](https://w3c.github.io/navigation-timing/#sec-PerformanceNavigationTiming) entries within the entries array. Milestones that were not reached (e.g., domainLookupEnd, responseStart, loadEventEnd) will be set to `0` to indicate where the failure occurred.

Note: CORS preflight requests are not exposed independently in reports, aligning with `PerformanceResourceTiming`.

### Example Report Payload

Here is an example of the entire response payload sent to the reporting endpoint. The sample submission contains two reports bundled together in a single HTTP request.

```
  POST / HTTP/1.1
  Host: telemetry.example.com
  Content-Type: application/reports+json
  
  [
    {
      "type": "performance-observer",
      "age": 1000,
      "url": "https://www.example.com/first",
      "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36",
      "body": {
        "entries": [
          {
            "name": "https://www.example.com/first",
            "entryType": "navigation",
            "startTime": 0,
            "domainLookupStart": 50,
            // DNS error happened, all subsequent fields are zeroed out.
            "domainLookupEnd": 0,
            "connectStart": 0,
            "responseStart": 0
          },
          {
            "name": "session-end-event",
            "entryType": "session-end",
            "startTime": 50,
            "duration": 0
          }
        ]
      }
    },
    {
      "type": "performance-observer",
      "age": 100,
      "url": "https://www.example.com/second",
      "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36",
      "body": {
        "entries": [
          {
            "name": "https://www.example.com/second",
            "entryType": "navigation",
            "startTime": 0,
            "domainLookupStart": 68,
            "domainLookupEnd": 120,
            "connectStart": 122,
            "secureConnectionStart": 160,
            "requestStart": 196,
            "responseStart": 562,
            "activationStart": 0
          },
          {
            "name": "hero-image-loaded",
            "entryType": "mark",
            "startTime": 780,
            "duration": 0,
            // https://w3c.github.io/user-timing/#performancemarkoptions-detail
            "detail": { "additionalinfo": "user defined arbitrary data" }
          },
          {
            "name": "hidden",
            "entryType": "visibility-state",
            "startTime": 13870,
            "duration": 0
          },
          {
            "name": "session-end-event",
            "entryType": "session-end",
            "startTime": 240200,
            "duration": 0
          }
        ]
      }
    }
  ]
```

### Report Timing (Session Termination)

The browser automatically finalizes and dispatches the compiled report payload upon detecting critical lifecycle terminal events:

- Tab Closure / Navigation Away
- BFCache Entry
- Early Errors (e.g., network errors)
- Renderer Crash

### How this solution would solve the use cases

- **Use case 1 (Early Network Failures):** By using the `capture-early-failures` directive, the browser remembers the origin's intent to record performance. On subsequent navigations, if a network error occurs before the response is received, the browser records the failure as an incomplete `PerformanceNavigationTiming` entry and saves it to a persistent buffer. This report is flushed on the next successful navigation to the same origin.
- **Use case 2 (OOM Crashes):** As events occur, they are streamed to the browser process. If the renderer crashes, the browser process still holds the accumulated payload and safely flushes it to the network.
- **Use case 3 (Session End Recording)**: The browser directly observes renderer termination and tab closures, automatically finalizing the report at the exact moment the session ends, without relying on JS execution.

## Detailed design discussion

### Deferred Reporting for Early Failures (capture-early-failures)

For the very first navigation to a site, events are reported if the browser successfully receives the response header because it’s activated via the `Performance-Observer` response header. If a network error occurs before receiving the response, the browser wouldn't know how to track it.

To solve this, the Performance-Observer has the `capture-early-failures` directive. If enabled, from the next navigation, the browser will collect early navigation failures as `PerformanceNavigationTiming` entries happening before receiving the response, and persist them to the storage. 

However, when the browser doesn’t receive the `Reporting-Endpoints` header, it’s impossible to send the report. Because Reporting API V1 is designed to be ephemeral, and tied to a  document response. This means the endpoint info doesn’t outlive the document lifecycle, thus not available from the navigation start until response received.

To minimize this case but also respect the Reporting API V1 principle, we propose a deferred reporting approach. Specifically, we don't persist the reporting endpoint. Instead, we persist report data if the report is not available to be sent.

**How it works:**

1. The user navigates to your site, and it fails due to a DNS error etc.
1. The browser process records the failure and saves the report data in its internal persistent buffer.
1. The user tries again later, and the navigation succeeds. The server responds with: Reporting-Endpoints: telemetry="https://example.com/reporting-endpoint"
1. Now that a valid document and endpoint exist, the browser flushes both the old failure reports and current reports from the buffer to that endpoint at the timing of session termination.

![High-level workflow how entries are managed](https://github.com/user-attachments/assets/cd7bde5e-ad34-4faa-b070-4bd068934642)

By doing that, we can report network errors while respecting the Reporting API's current principle. As a tradeoff, if the user never returns to the site, the failure report is lost, but this might still capture the vast majority of failures. Also, to keep the memory and disk usage limited, the browser should have some quota. **Note that we have the `capture-early-failures directive` for the opt-in signal. By default, we don't persist events.**

### Explicit `PerformanceSessionEndTiming` vs spec minimalism

To explicitly flag a session boundary and provide precise duration, we propose a new struct, `PerformanceSessionEndTiming` (inheriting from `PerformanceEntry`). We considered not defining this to keep the API surface minimal and avoid adding a new type to the web platform specification. However, without it, we would lose the ability to measure the precise total duration of the session (dwell time after the last recorded event). Since the goal is to measure the life of a user journey from end-to-end, we decided to keep this explicit terminal event to provide accurate duration metrics.

## Considered alternatives

Before arriving at the proposed design, several existing systems and alternative API shapes were evaluated:

### Network Error Logging (NEL)

NEL provides a mechanism for collecting client-side network errors via an HTTP response header, capturing early failures before JS execution. However, NEL is strictly scoped to network errors. We need a solution that captures the entire end-to-end events (network timings, page visibility, and user-defined application marks), not just network errors. Also, NEL depends on the Reporting API V0, which is already deprecated.

### ServiceWorker

ServiceWorkers can intercept network requests and observe closures, enabling end-to-end recording. However, adopting ServiceWorker introduces new complexity and performance overhead.

### Combination of Existing Technologies (NEL + JS PerformanceObserver + fetchLater() + Renderer Crash Reporting)

One might consider stitching together existing APIs: using `NEL` to catch early network failures, JS `PerformanceObserver` to record performance events, [fetchLater()](https://fetch.spec.whatwg.org/#dom-window-fetchlater) to guarantee beacon transmission when the page closes, and Renderer Crash Reporting to capture crashes. However, this fragmented approach fails to meet the requirements because `NEL` only captures network-level errors, while `fetchLater` and JS Performance APIs require the JavaScript environment to be successfully initialized. If the user closes the tab after the network response but before JS executes and registers the `fetchLater` beacon, the failure is completely invisible. Furthermore, `NEL` reports are generated independently by the network stack and have a fundamentally different structure than JS-generated beacons. There is no reliable way to correlate a `NEL` failure report with an intended application journey on the server side to calculate accurate success rates.

### Extend performance.mark() instead of include-user-timing

This alternative allows developers to specify which events are recorded via JavaScript by extending `markOptions` in the `performance.mark()` API. One concern is that any third-party scripts can set this option as well.

## Security and Privacy Considerations

For Security and Privacy considerations, please check [self-review questionnaires](https://github.com/explainers-by-googlers/declarative-performance-observer/blob/main/security_privacy_questionnaire_answers.md) as well.

### Denial of Service (DoS) Prevention (Memory)

To prevent malicious scripts from spamming performance events and causing an Out-Of-Memory (OOM) crash in the browser process, the API enforces a strict **640KB buffer size limit per document** lifecycle. If the buffer fills, newest incoming entries are silently dropped, preserving critical early-load metrics. This protects the privileged browser process from memory exhaustion.

### Disk Quota for Buffered Reports (Storage DoS)

The `capture-early-failures` feature requires persisting failure reports to disk when no active endpoint is available. To prevent a malicious site from filling the user's disk by repeatedly triggering failed navigations, the browser must enforce a strict **disk quota** for these buffered reports. Reports exceeding the quota should be dropped following a FIFO (First-In, First-Out) policy.

The spec will state that the user agent SHOULD enforce a disk quota for buffered reports, leaving the exact size implementation-defined.

### Safe API Design and Telemetry Hijacking

The API is activated exclusively via declarative HTTP response headers, not through a JavaScript API. This prevents third-party scripts (such as malicious ads or trackers) from arbitrarily activating performance observation on a page without the origin's explicit server-side consent.

Also, this API is strictly limited to **first-party activation**. The API is driven by HTTP response headers on the main document navigation. It is primarily intended for first-party use. Third-party resources cannot trigger these reports for the main document.

### Telemetry Pollution by Third-Party Scripts

While third-party scripts cannot activate the feature or modify the list of allowed marks, they can still call performance.mark() with names that are listed in the header's include-user-timing. If they do so, they could potentially pollute the telemetry data.

**Mitigation:** Origins should use specific, non-generic names for their critical performance marks to reduce the risk of accidental or intentional collision by third-party scripts. Future iterations could consider restricting mark capture to scripts originating from the same origin, though this adds complexity.

### Timer Resolution and Side-Channels

Time-based fields in the report are subject to standard clock resolution limits (coarsening) applied to high-resolution timers in the web platform. This mitigates the risk of using these reports as high-precision timers for side-channel attacks such as Spectre.

### Reporting Endpoint Security

The API relies on the infrastructure of the Reporting API for report delivery. The primary protection is isolation: reports generated for a document are only sent to the endpoints explicitly configured by that document's headers. This prevents unintentional data leakage. Note that if multiple origins choose to share the same reporting endpoint, that endpoint can correlate user activity across those sites (similar to sharing third-party analytics scripts), as noted in the Reporting API specification.

### Data Minimization and Opt-in

The API follows data minimization principles by only collecting entries explicitly requested by the developer via the entry-types and include-user-timing directives. The session-end event is anonymous and only contains the timestamp of the session end.

### Cross-Site Correlation and Storage Partitioning

Any state introduced by this API (such as the capture-early-failures flag and buffered failure reports) is stored in partitioned storage. It is isolated by the top-level site and cannot be used to link a user's activity across different sites. By default, events are not persisted unless the user opts in via capture-early-failures.

### Incognito / Private Browsing Mode

Persisted events and origin-level flags are isolated to partitioned storage and never leak between regular and Incognito modes. Data is purged when the Incognito profile is destroyed.

### Crash Detection Heuristic vs. Explicit Field

The proposal intentionally omits closure type details (such as whether the session ended due to a crash or a normal close) in the session-end event to adhere to privacy minimalism. However, as noted by reviewers, if the heuristic (inferring a crash from the absence of a visibility-state: hidden event before session end) is $100%$ reliable, it exposes equivalent information to an explicit field. This remains an active area of discussion with privacy teams to ensure the design meets privacy expectations.