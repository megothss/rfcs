---
stage: accepted
start-date: 2026-03-06T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted:
project-link:
suite:
---

# ErrorBoundary Component

## Summary

Introduce a built-in `<ErrorBoundary>` component that catches synchronous errors during Glimmer VM render (initial and rerender), displays fallback UI via named blocks, and supports automatic recovery via a `@retryWith` argument. This gives Ember apps a declarative way to isolate render failures and recover gracefully, instead of letting a single broken component take down an entire page.

## Motivation

Today, when a component throws during render, the error propagates up uncaught and can leave the page in a broken or unresponsive state. There's no declarative mechanism to catch these errors, display fallback UI, or recover without a full page reload. This is a significant gap in Ember's component model.

### No built-in error recovery

If a component's getter throws during render, or a helper invocation fails, the entire render pass aborts. The DOM may be left partially rendered, and there's no way for the app to recover gracefully. The user is left staring at a broken page.

```gjs
import Component from '@glimmer/component';

class BuggyWidget extends Component {
  get title() {
    return this.args.data.title; // throws if @data is undefined
  }

  <template>
    <h2>{{this.title}}</h2>
  </template>
}

<template>
  <Header />
  <Sidebar />
  <BuggyWidget />  {{! this error takes down the entire page }}
  <Footer />
</template>
```

In this example, `BuggyWidget` throws because `@data` was not passed. But the failure isn't isolated to the widget. The entire page, including `<Header>`, `<Sidebar>`, and `<Footer>`, is left broken with no way to recover.

### Real-world need: plugin architectures

Large applications with plugin or extension systems allow third-party code to render components within the host app's component tree. A bug in a single plugin can take down the entire page: the sidebar, the header, the content area, everything. ErrorBoundary would let the host app isolate plugin-rendered sections so that a failure in one plugin only affects that plugin's UI.

### Framework parity

Every other major frontend framework provides error boundaries:

- **React**: [`componentDidCatch` / `getDerivedStateFromError`](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) (since v16, 2017)
- **Solid**: [`<ErrorBoundary>`](https://docs.solidjs.com/concepts/control-flow/error-boundary)
- **Vue**: [`onErrorCaptured`](https://vuejs.org/api/composition-api-lifecycle.html#onerrorcaptured)
- **Preact**: [`componentDidCatch`](https://preactjs.com/guide/v10/components/#error-boundaries)
- **Svelte**: [`<svelte:boundary>`](https://svelte.dev/docs/svelte/svelte-boundary) (since v5.3, 2024)

Ember is one of the few remaining major frameworks without declarative render-error recovery.

### Graceful degradation

ErrorBoundary enables progressive enhancement patterns. Wrap non-critical UI sections (widgets, sidebars, third-party embeds) in boundaries so that failures degrade gracefully while the rest of the app stays interactive.

## Detailed design

### Import

```js
import { ErrorBoundary } from '@ember/component';
```

ErrorBoundary would be a built-in component shipped as part of the framework, not an addon. Available in any Ember app without additional installation.

### Basic usage

ErrorBoundary uses Ember's named blocks syntax:

```gjs
import { ErrorBoundary } from '@ember/component';

<template>
  <ErrorBoundary>
    <:default>
      <RiskyComponent />
    </:default>
    <:error as |error retry|>
      <p>Something went wrong: {{error.message}}</p>
      <button {{on "click" retry}}>Retry</button>
    </:error>
  </ErrorBoundary>
</template>
```

- `<:default>` is rendered when there's no error. This is the "happy path" content.
- `<:error as |error retry|>` is rendered when an error is caught during the default block's render.
  - `error` (`unknown`): the caught error object. You'll want to narrow the type before accessing properties.
  - `retry` (`() => void`): a function that clears the error state and re-renders the default block. If the underlying issue isn't resolved, the error will be caught again.

When used without an explicit `<:default>` block, the implicit default block is used:

```gjs
<ErrorBoundary>
  <RiskyComponent />
</ErrorBoundary>
```

Without an `<:error>` block, caught errors won't have any visible fallback (the boundary will simply render nothing when in error state). In practice, you should always provide both blocks.

### The `@retryWith` argument

```gjs
import { ErrorBoundary } from '@ember/component';
import { service } from '@ember/service';

class MyComponent {
  @service router;

  <template>
    <ErrorBoundary @retryWith={{this.router.currentRouteName}}>
      <:default>
        {{outlet}}
      </:default>
      <:error as |error retry|>
        <p>This page encountered an error.</p>
        <button {{on "click" retry}}>Retry</button>
      </:error>
    </ErrorBoundary>
  </template>
}
```

`@retryWith` accepts any value: a primitive, an array, or a plain object. When the boundary is in error state and the `@retryWith` value changes (determined by shallow equality), the error is automatically cleared and the default block re-renders.

Shallow equality semantics:
- Primitives: compared with `===`
- Arrays: compared element-wise (same length, each element `===`)
- Plain objects: compared by own-property values (same keys, each value `===`)

When the boundary is *not* in error state, `@retryWith` value changes are tracked but don't trigger any re-render or recovery logic. There's no performance cost in the happy path beyond the tracking overhead.

### Internal template

The ErrorBoundary component's template is minimal:

```hbs
{{#if this.hasError}}
  {{yield this.error this.retry to="error"}}
{{else}}
  {{yield}}
{{/if}}
```

The actual error-catching behavior would be implemented at the Glimmer VM level, not in the template. A new `errorBoundary: true` capability flag on the component manager would signal to the VM that this component should catch errors during its child tree's render.

### What IS caught

ErrorBoundary catches synchronous errors that occur during the Glimmer VM's execution phase:

- Component getter/render errors: errors thrown from tracked getters accessed during render, both initial render and rerender
- Helper invocation errors: errors thrown from helpers invoked in templates
- `{{#each}}` list sync errors: errors thrown during iteration key computation or list reconciliation
- Conditional branch transitions: errors thrown when entering an `{{#if}}` / `{{else}}` branch

### What is NOT caught

ErrorBoundary doesn't catch:

- Modifier install/update errors: modifiers run in `transaction.commit()` after the VM execution phase completes, outside the boundary's try/catch scope
- Async errors: errors in `setTimeout`, `requestAnimationFrame`, Promise rejections, `ember-concurrency` tasks, etc.
- Errors during component destruction: destructor callbacks run outside the render pass
- Errors in event handlers: `{{on "click" this.handleClick}}` errors aren't render errors

This is intentional. ErrorBoundary catches errors during the synchronous Glimmer VM execution pass. Async error handling is a separate concern best addressed by application-level patterns (e.g., `ember-concurrency`, route error substates).

### Nesting

ErrorBoundaries can be nested. When an error occurs, the **innermost** enclosing boundary catches it:

```gjs
<ErrorBoundary>
  <:default>
    <ErrorBoundary>
      <:default>
        <ThrowingComponent /> {{! caught by inner boundary }}
      </:default>
      <:error as |error|>
        <p>Inner caught: {{error.message}}</p>
      </:error>
    </ErrorBoundary>
  </:default>
  <:error as |error|>
    <p>Outer caught: {{error.message}}</p>
  </:error>
</ErrorBoundary>
```

If the `<:error>` block itself throws, the error bubbles to the next enclosing boundary (or goes uncaught if there's no parent boundary).

### Error recovery and DOM cleanup

When an error is caught:

1. The VM's updating opcode list is rolled back to the state before the failed render
2. Any DOM nodes created during the failed render pass are removed
3. The debug render tree is rolled back (`debugRenderTree.rollbackTo()`)
4. The boundary transitions to error state and renders the `<:error>` block

When `retry()` is called (or `@retryWith` triggers automatic recovery):

1. The error state is cleared
2. The boundary re-renders the `<:default>` block from scratch
3. If the default block throws again, the error is caught again and the boundary returns to error state

### Development mode behavior

In development builds (`DEBUG` is true), ErrorBoundary logs caught errors to the console:

```
console.error('An error was caught by <ErrorBoundary>:', error)
```

This way developers are still aware of caught errors during development, even though the UI recovers gracefully.

### Ecosystem implications

**ember-template-lint:** No new lint rules needed. ErrorBoundary uses standard named blocks syntax, which is already supported.

**Ember Inspector:** The implementation would maintain the debug render tree during error recovery. `debugRenderTree.rollbackTo()` would keep the Inspector's view of the component tree consistent after an error is caught.

**Server-side rendering (FastBoot):** ErrorBoundary operates at the Glimmer VM level and catches synchronous render errors. It should work in FastBoot without modification since FastBoot uses the same Glimmer VM for rendering. This should be verified in practice.

**Ember Engines:** ErrorBoundary should work within a single engine's component tree. This needs real-world verification.

**TypeScript:** ErrorBoundary would be fully typed. The `error` block parameter would be typed as `unknown`, so you'd need to narrow the type before accessing properties, which is standard TypeScript practice.

**Addons:** Addon authors can use ErrorBoundary to make their components more resilient. Host apps can wrap addon-provided components in boundaries to isolate failures.

## How we teach this

### Naming

"ErrorBoundary" is established terminology across the frontend ecosystem. React introduced the concept in 2017, and Solid, Vue, and Preact all use the same or similar naming. Using `ErrorBoundary` in Ember:

- Reduces cognitive overhead for developers coming from other frameworks
- Makes the feature immediately searchable and discoverable
- Aligns with existing community expectations

### Guides

A new section should be added to the Ember guides under "Components":

**"Handling Render Errors with ErrorBoundary"**

The guide should cover:

1. Why you need error boundaries: what happens when a component throws during render, and why you want to isolate failures
2. Basic usage: wrapping a section of UI in `<ErrorBoundary>` with `<:default>` and `<:error>` blocks
3. The retry pattern: using the `retry` function to let users attempt recovery
4. Automatic recovery with `@retryWith`: binding to route name or other reactive values for automatic recovery when context changes
5. What is and isn't caught: clearly explaining the synchronous render scope, and directing developers to other patterns for async errors
6. Nesting boundaries: using multiple boundaries to isolate different sections of the page

### API documentation

The `@ember/component` module documentation should include:

- `ErrorBoundary`: the component class (import path and usage)
- `@retryWith`: the argument for automatic recovery
- `<:default>` and `<:error as |error retry|>`: the named blocks and their parameters

### Teaching approach

ErrorBoundary should be presented as a **progressive enhancement** tool:

- "Identify sections of your UI that could fail independently, and wrap them in `<ErrorBoundary>`"
- Start with the manual `retry` pattern as the primary recovery mechanism
- Introduce `@retryWith` as an advanced pattern for route-level or context-dependent recovery
- Emphasize the limitations clearly: ErrorBoundary is for render errors only, not a general-purpose error handling mechanism

## Drawbacks

### False sense of safety

Developers may assume that wrapping content in `<ErrorBoundary>` catches all possible errors. In practice, modifier errors, async errors, and event handler errors all escape the boundary. This needs to be documented clearly and taught explicitly to avoid a false sense of security.

### VM complexity

The implementation touches sensitive internal parts of the Glimmer VM: state restoration during error recovery, DOM cleanup of partially-rendered trees, and tracking system integration. This increases the maintenance surface area for the VM team and introduces new code paths that need to be considered during future VM changes.

### Swallowed errors

In production, caught errors aren't surfaced to the user beyond the fallback UI. `console.error` is emitted in development builds, but production builds catch errors silently. If ErrorBoundary is overused, especially without logging, it could mask bugs that should be fixed. Best practices should recommend pairing ErrorBoundary with error reporting (e.g., sending caught errors to a monitoring service).

## Alternatives

### Prior discussion

[RFC issue #513](https://github.com/emberjs/rfcs/issues/513) (2019) and [RFC issue #518](https://github.com/emberjs/rfcs/issues/518) (2019) both requested error boundaries for Ember but were never formalized into an RFC. There's no known community addon that provides render-level error boundaries, likely because the feature requires changes to the Glimmer VM itself. Existing error handling addons operate at the application level (via `Ember.onerror` / `window.onerror`), not within the component tree.

### React's class-based API

React implements error boundaries via class component lifecycle methods (`componentDidCatch`, `getDerivedStateFromError`). This RFC proposes named blocks instead, which:

- Is more declarative and template-centric
- Doesn't require a class component. ErrorBoundary works in any template context
- Provides the `retry` function directly as a block parameter, so recovery doesn't require extra wiring

### `@key` instead of `@retryWith`

An alternative name `@key` was considered for the automatic retry argument. This was rejected because `@key` in Ember's `{{#each}}` helper represents a property path for identity tracking, not a reactive value for triggering side effects. `@retryWith` communicates the "retry" intent clearly and avoids confusion with existing Ember concepts.

### Status quo (no built-in boundary)

Doing nothing leaves Ember as one of the few major frameworks without declarative render-error recovery. This is particularly painful for apps with plugin architectures, where third-party code can break the host app's UI. Without error boundaries, developers either accept the risk of full-page failures or implement fragile workarounds.

## Unresolved questions

- **Should ErrorBoundary catch modifier errors?** Modifiers currently run in `transaction.commit()` after VM execution. Catching modifier errors would require changes to the transaction commit phase and careful consideration of DOM state consistency. This could be addressed in a follow-up RFC.

- **FastBoot and Ember Engines compatibility:** ErrorBoundary should work in FastBoot since it uses the same Glimmer VM for rendering, and within engine component trees. These environments should be tested before the feature is marked as stable.

## Proof of concept

A working proof-of-concept implementation and interactive demo are available:

- **Implementation**: [megothss/ember.js#2](https://github.com/megothss/ember.js/pull/2), a fork of ember-source with the Glimmer VM changes, ErrorBoundary component, and test coverage
- **Live demo**: [ember-error-boundary-demo](https://megothss.github.io/ember-error-boundary-demo/), a standalone Ember app with scenarios covering render errors, retry/recovery, nested boundaries, sibling isolation, `@retryWith`, and more
