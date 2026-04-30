## Answers to Security and Privacy Self-Review Questionnaire: Declarative Performance Observer

This document provides answers to the questions listed in the W3C Self-Review Questionnaire: Security and Privacy.

### 2.1 What information does this feature expose, and for what purposes?

It exposes performance metrics (navigation timings, visibility states) and application-defined marks to the origin's designated reporting endpoint. The purpose is to measure end-to-end performance events with high reliability, capturing data that is currently lost due to early network failures or renderer crashes.

### 2.2 Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

Yes. It only exposes entries explicitly requested in the entry-types and include-user-timing directives of the header. The session-end event only contains the time when the user closed the page (or the session ended).

### 2.3 Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No. It exposes performance timings and application-defined marks. While application marks could theoretically contain identifying data if misused by the developer, the API itself does not collect PII. Timings are subject to clock resolution limits to prevent fingerprinting.

### 2.4 How do the features in your specification deal with sensitive information?

It does not intentionally expose sensitive information. Timings are relative to navigation start and are coarsened.

### 2.5 Does data exposed by your specification carry related but distinct information that may not be obvious to users?

No. The payload consists of standard performance entries that are already observable via JavaScript, just delivered more reliably.

### 2.6 Do the features in your specification introduce state that persists across browsing sessions?

Yes, if capture-early-failures is enabled, an origin-level flag is persisted to record future early failures. Also, report data for failed navigations is persisted in a buffer until flushed. This state is stored in partitioned storage, meaning it is isolated by the top-level site and cannot be used to link the user's activity across different sites.

### 2.7 Do the features in your specification expose information about the underlying platform to origins?

No new platform information is exposed.

### 2.8 Does this specification allow an origin to send data to the underlying platform?

No.

### 2.9 Do features in this specification enable access to device sensors?

No.

### 2.10 Do features in this specification enable new script execution/loading mechanisms?

No.

### 2.11 Do features in this specification allow an origin to access other devices?

No.

### 2.12 Do features in this specification allow an origin some measure of control over a user agent’s native UI?

No.

### 2.13 What temporary identifiers do the features in this specification create or expose to the web?

None.

### 2.14 How does this specification distinguish between behavior in first-party and third-party contexts?

The API is driven by HTTP response headers on the main document navigation. It is primarily intended for first-party use. Third-party resources cannot trigger these reports for the main document.

### 2.15 How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

Persisted events and origin-level flags are isolated to partitioned storage and never leak between regular and Incognito modes. Data is purged when the Incognito profile is destroyed.

### 2.16 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes, they are included in the explainer.

### 2.17 Do features in your specification enable origins to downgrade default security protections?

No.

### 2.18 What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

When a page enters BFCache, it is considered a terminal event, and the current session report is flushed. If the page is restored from BFCache, a new session begins with a new PerformanceNavigationTiming entry.

### 2.19 What happens when a document that uses your feature gets disconnected?

If the document is disconnected (e.g., iframe removed or tab closed), it is treated as a session termination, and the report is flushed.

### 2.20 Does your spec define when and how new kinds of errors should be raised?

Yes, it defines how early network failures are recorded as incomplete PerformanceNavigationTiming entries with zeroed-out milestones.

### 2.21 Does your feature allow sites to learn about the user’s use of assistive technology?
No.

### 2.22 What should this questionnaire have asked?

It might have asked about denial-of-service risks to the browser process due to large payloads, which we addressed by proposing a 640KB buffer size limit per document lifecycle.