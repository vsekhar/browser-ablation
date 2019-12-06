# browser-ablation

This document proposes a low-level mechanism to support ablation experiments when measuring web site performance.

## Background

Measuring site performance benefits from a close connection with publisher business outcomes. Establishing this connection is difficult and often confounded by concurrent changes to site UX, content and product offerings.

**Causal connections** between performance and business outcomes require randomized A/B testing on live traffic. Sites, frameworks and analytics providers may support such experiments within the scope of the components they control, e.g. by varying the image or javascript payloads, or delaying server response. However, these methods do not always have a direct relationship to user-observable performance and/or require large interventions affecting many users to produce measurable effects.

Internally, Chrome has used browser-based experiments that modify memory use and establish causal connections between memory use and user experience. These studies are known as ablation studies, and are common in other black- or grey-box testing environments like machine learning.

## Proposal

Provide certain experimental ablations in the browser, closer to the user, to reduce the degree of ablation, population size and duration needed to see statistically valid results.

### Diversion

User diversion into experiment groups is the responsibility of sites. It is recommended that sites use cookies to ensure diversions are stable.

### Triggering

A site (or more likely, their experiments provider/CDN) can indicate desired ablation values for a page load via an HTTP response header:

```http
Ablation: first-contentful-paint=20ms;first-input-delay=10ms;
```

Alternatively, an HTML meta tag can be used to similar effect:

```http
<meta name="ablation" content="first-contentful-paint=20ms, first-input-delay=10ms">
```

### Ablation types

Ablation types include:

 * `first-contentful-paint`
 * `first-input-delay`
 * `time-to-first-byte`
 * `javascript-time-to-first-byte`
 * `javascript-first-parse`
 * `javascript-first-compile`
 * Something scrolling?
 * … others?

### Ablation semantics

Ablations are applied by the browser in addition to any delays organically present during the page load.

For example, setting a non-zero ablation value for `first-input-delay` would be semantically equivalent to inserting a one-time busy wait at the top of all input event handlers:

```js
document.getElementById("myButton").addEventListener("click", clickHandler);

function clickHandler() {
  busyWaitIfFirstInput(ablation_first_input_delay_ms);

  // … remaining event handler code
}
```

Ablation of `time-to-first-byte` is approximated by applying a delay as soon as the ablation parameter is read. For greater accuracy, we may recommend (require?) that ablation of `time-to-first-byte` be declared via a header.

Javascript ablation delays script execution at various stages. Abaltion of `javascript-time-to-first-byte` delays the availability of the payload of the first JS fetch. Ablation of `javascript-first-{parse,compile}` delays the start of parsing or compiling the first javascript payload.

### Subresources

Ablation values specified when fetching first-party subresources are applied to that sub-resource.

Ablation values specified when fetchin third-party subresources are ignored.

> TODO: COEP-style permission for third-party subresource ablation?

### Performance metrics

Performance metrics are reported as usual via the Performance API. For page loads where ablations are applied, performance metrics will include those ablations. Performance APIs will also report the ablation values, if any, applied to the page load.

### Business metrics

Sites can choose business events and metrics relevant to their page (e.g. sub-page navigation, email newsletter sign-up, shopping cart activity, purchase completion). Defining business metrics, collecting them, and correlating them to experimental cohorts (and thus performance metrics) is the responsibility of the site or their analytics provider.

## Open issues

 * Should the UA be able to override the ablation request? Some cases where this may be needed:
   * Egregious ablation amounts (e.g. 100,000ms)
   * Repeated ablation on the same domain, possibly an error in user diversion
   * Repeated ablation for a given user across domains, possibly due to overuse of ablation among popular sites

## Major edits

 * 2019-12-06: Add subresource ablation policy.
 * 2019-12-06: Add potential javascript ablation values
   * Tag providers (ad networks, analytics providers) can assess impact on business metrics for 
