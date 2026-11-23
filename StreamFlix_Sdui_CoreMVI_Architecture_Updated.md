# StreamFlix SDUI + CoreMVI / Reflex-Inspired Architecture

## Architecture decisions, concepts, contracts, and implementation roadmap

**Status:** Architecture proposal consolidated from our discussions\
**Date:** 21 November 2026\
**Project:** Netflix-inspired Server-Driven UI (SDUI) application,
initially using mock JSON

------------------------------------------------------------------------

## Companion architecture document

This document focuses on the StreamFlix application of CoreMVI: SDUI
schema, component registry, bindings, dependencies, and Shopify
Reflex-inspired headless integration.

For the complete reusable CoreMVI foundation and its
evolution---including the original package goals, base contracts,
Store/Reducer/IntentHandler/EffectHandler responsibilities, Registration
Wizard example, repository/state ownership, concurrency, cancellation,
instrumentation, composition, testing, introspection, and implementation
decisions---see the master document:

**[CoreMVI --- Complete Architecture &
Evolution](CoreMVI_Complete_Architecture_Evolution.md)**

Recommended reading order: read the CoreMVI master document first, then
use this document for SDUI/Reflex and StreamFlix-specific integration.

------------------------------------------------------------------------

## 1. Project goal

Build a Netflix-inspired streaming application whose screens and
component composition are delivered by a server-controlled JSON
definition.

The first version uses mock JSON rather than a production backend. The
architecture should support:

-   iOS and Android with native UI.
-   Server-defined screen composition and content.
-   Reusable SDUI components.
-   Component-to-component dependencies expressed declaratively.
-   A headless application core that can be tested and driven without
    rendering UI.
-   Action/effect interception for cross-cutting behavior.
-   Introspection so tools can inspect state and invoke supported
    actions.
-   A clean path from mock data to real APIs.

The intention is not to build one shared UI implementation. The
platforms share contracts, schemas, behavior expectations, and tests
where practical, while each platform renders with its native UI
framework.

## 2. Decisions already made

  ---------------------------------------------------------------------
  Area                               Decision
  ---------------------------------- ----------------------------------
  Product direction                  Netflix-inspired streaming app

  Platforms                          Cross-platform: iOS and Android

  Initial backend                    Mock JSON

  State architecture                 Core MVI, evolving with
                                     Reflex-inspired capabilities

  Intent handling                    Separate `IntentHandler`

  Composition                        Tree

  Testing                            Unit tests and
                                     integration/behavior tests

  Initial implementation focus       Build and validate the iOS
                                     vertical slice, then port the same
                                     contracts and behavior to Android

  UI technology                      SwiftUI on iOS; Jetpack Compose on
                                     Android

  Core principle                     Native UI is an adapter; business
                                     behavior belongs in a headless
                                     core
  ---------------------------------------------------------------------

These decisions are the baseline for the design below. The proposed
additions do not mean that every detail is already implemented.

## 3. High-level architecture

``` text
                    Mock JSON / Future Backend
                              |
                              v
                    SDUI Screen Definition
                              |
                              v
+----------------------------------------------------------------+
|                         Native UI                              |
|                                                                |
|             iOS SwiftUI             Android Compose             |
+--------------------------+-------------------------------------+
                           |
                           v
+----------------------------------------------------------------+
|                       CoreSDUI Runtime                          |
|                                                                |
|  Schema Decoder       Component Registry       Native Renderer  |
|  Binding Resolver     Dependency Evaluator     Action Adapter   |
|  Design Tokens        Validation               SDUI Introspection|
+--------------------------+-------------------------------------+
                           |
                    dispatch feature action
                           |
                           v
+----------------------------------------------------------------+
|                     Headless CoreMVI                            |
|                                                                |
|  Intent -> IntentHandler -> Action -> Action Interception      |
|                                      |                         |
|                                      v                         |
|                                  Reducer                        |
|                                      |                         |
|                                      v                         |
|                                    State                        |
|                                                                |
|  Effect -> EffectHandler -> Effect Interception -> Services    |
|                                                                |
|  Introspection Tree       State Snapshot       Action Catalog  |
+--------------------------+-------------------------------------+
                           |
                 injected platform services
                           |
             Network / Persistence / Auth / Media
```

### Responsibility boundaries

**CoreMVI** - Owns feature behavior, state transitions, actions,
effects, and action dispatch. - Has no dependency on SwiftUI, UIKit,
Jetpack Compose, or Android UI. - Is executable in a headless process. -
Exposes a stable introspection contract.

**CoreSDUI** - Decodes and validates server screen definitions. -
Resolves component types through a registry. - Resolves bindings between
component properties and state. - Evaluates declared dependencies. -
Converts component events into allowed feature actions. - Does not own
business state or call repositories directly.

**Platform UI** - Renders native components. - Subscribes to the state
or view models it needs. - Sends user events through the CoreMVI
dispatch path. - Does not contain business rules.

**Services and repositories** - Are injected into the core. - Own API
access, persistence, caching, and external capabilities. - Keep fetched
content collections out of lightweight feature state when those
collections are better owned by repositories.

------------------------------------------------------------------------

## 4. CoreMVI: the feature execution model

The basic unidirectional flow remains:

``` text
User Event
   |
   v
Intent
   |
   v
IntentHandler
   |
   v
Action
   |
   v
Action Interceptor Pipeline
   |
   v
Reducer ----------------------> New State
   |                               |
   |                               v
   |                         UI observes state
   |
   v
Effect (when required)
   |
   v
EffectHandler
   |
   v
Effect Interceptor Pipeline
   |
   v
Repository / Service / API
   |
   v
Result Action
   |
   v
Action Pipeline -> Reducer -> New State
```

### Why keep `IntentHandler` separate?

The `IntentHandler` is the boundary that translates UI-oriented intent
into domain/feature actions.

For example:

-   UI intent: `CategoryTapped("Drama")`
-   Action: `SelectCategory(categoryID: "drama")`

This separation keeps view event details out of reducers and allows
intent mapping to evolve independently. It also gives the feature a
clear place to validate or normalize user input before producing
actions.

### Reducer

The reducer is the deterministic state transition function:

``` swift
func reduce(
    state: inout HomeState,
    action: HomeAction
) -> [HomeEffect]
```

It: - Receives the current state and an action. - Applies synchronous
state changes. - Returns effects that need asynchronous work. - Does not
make network calls. - Does not directly navigate or access UI
frameworks.

The reducer is the single authority for feature state transitions.

### EffectHandler

The EffectHandler performs asynchronous work requested by the reducer:

-   Fetch content.
-   Load details.
-   Persist a watch-list change.
-   Request playback preparation.
-   Observe a repository or service.

When work completes, it dispatches a result action. The reducer then
decides how state changes.

### Feature state should stay lightweight

A key design preference is to avoid storing large API-owned collections
in feature state merely because a screen displays them.

For example, for a form with country and city dropdowns: - The
repository can own the country list and cache it. - The feature state
can contain selected country ID, selected city ID, loading/error flags,
and other UI-relevant values. - A repository lookup can return the
required display data to the UI through an appropriate read model or
projection.

For StreamFlix: - The content repository owns the catalogue and cached
content. - Home state holds values needed to describe the current
interaction, such as selected category, loading state, selected item,
and relevant screen-level identifiers. - The SDUI document describes
which content slots and components to render.

Avoid duplicating the entire catalogue in multiple component states.

------------------------------------------------------------------------

## 5. CoreSDUI: server-driven screen contract

The server sends a declarative screen definition. The client validates
it and renders it using a registry of supported native components.

A screen document should include concepts such as:

-   `schemaVersion`: version of the JSON contract.
-   `screenId`: stable identity of the screen.
-   `screenVersion`: version of this screen definition.
-   `theme`: design-token or theme reference.
-   `components`: ordered component tree.
-   Component `id`: stable identity within the screen.
-   Component `type`: registered component type.
-   Component `props`: display configuration.
-   Component `children`: nested components where applicable.
-   `bindings`: connections from properties to state.
-   `actions`: declarative events that map to allowlisted client
    actions.
-   `dependencies`: relationships that cause derived values or effects
    to update.

### Illustrative JSON

This is a conceptual example of the contract, not a final JSON Schema:

``` json
{
  "schemaVersion": "1.0",
  "screenId": "home",
  "screenVersion": 12,
  "theme": "streamflix-dark",
  "components": [
    {
      "id": "category-selector",
      "type": "categorySelector",
      "props": {
        "title": "Browse by category"
      },
      "bindings": {
        "selectedCategoryId": {
          "path": "home.selectedCategoryId",
          "mode": "twoWay"
        }
      },
      "actions": {
        "onSelect": {
          "type": "dispatch",
          "action": "home.selectCategory"
        }
      }
    },
    {
      "id": "trending-carousel",
      "type": "contentCarousel",
      "props": {
        "title": "Trending"
      },
      "bindings": {
        "items": {
          "path": "home.trendingItems",
          "mode": "read"
        }
      },
      "dependencies": [
        {
          "on": "home.selectedCategoryId",
          "effect": "home.loadTrendingForCategory"
        }
      ]
    }
  ]
}
```

The exact field names and semantics should be frozen in a versioned
schema before production use.

### Component registry

The registry maps a server-provided component type to a platform-native
renderer.

Examples:

  -----------------------------------------------------------------------
  JSON type               iOS renderer            Android renderer
  ----------------------- ----------------------- -----------------------
  `heroBanner`            SwiftUI hero view       Compose hero

  `contentCarousel`       SwiftUI horizontal      Compose lazy row
                          carousel                

  `posterCard`            SwiftUI poster card     Compose poster card

  `categorySelector`      SwiftUI selector        Compose selector

  `continueWatching`      SwiftUI progress        Compose progress
                          carousel                carousel
  -----------------------------------------------------------------------

Unknown component types should fail gracefully: - Skip unsupported
optional components or render a fallback. - Log schema/registry
diagnostics. - Never crash the whole screen because one noncritical
component is unknown.

### Recursive rendering

The renderer walks the component tree: 1. Read the component type. 2.
Resolve the renderer from the registry. 3. Validate and decode its
props. 4. Resolve bindings. 5. Render the component. 6. Recursively
render children.

A server-provided type must never be used to instantiate arbitrary code.
The registry is an explicit allowlist.

------------------------------------------------------------------------

## 6. Component dependencies: how one component updates another

A component should not hold a direct reference to another component.
Components communicate through shared feature/screen state and declared
bindings.

Example: selecting a category updates the carousel.

``` text
CategorySelector
       |
       | user selects "Drama"
       v
dispatch SelectCategory("drama")
       |
       v
CoreMVI Action Pipeline
       |
       v
Reducer updates selectedCategoryId
       |
       +-------------------------------+
       |                               |
       v                               v
Selector binding refreshes      Dependency evaluator sees
selectedCategoryId              selectedCategoryId changed
                                       |
                                       v
                              Effect: loadTrending("drama")
                                       |
                                       v
                              Repository returns content
                                       |
                                       v
                              TrendingLoaded(items)
                                       |
                                       v
                                    Reducer
                                       |
                                       v
                              State projection updates
                                       |
                                       v
                              Carousel binding refreshes
```

### Two kinds of dependency

**1. Pure state binding**

If a component displays a value derived from state, it should update
when that state changes. No additional effect is needed.

Example: - Selector's selected category label. - Button enabled/disabled
state. - A badge count.

**2. Data dependency**

If a state change requires new data, the dependency declaration triggers
an effect through CoreMVI.

Example: - Selected category changes. - Trending carousel needs
category-specific content. - Dependency evaluator requests the
corresponding feature action/effect. - Repository fetches or returns
cached content. - Result action updates the state projection used by the
carousel.

### Dependency rules and safeguards

The runtime should validate: - Referenced state paths exist. -
Referenced actions are allowlisted. - Component IDs are unique within
the screen. - Dependency graphs do not contain unintended cycles. -
Duplicate requests are coalesced where appropriate. - Old requests are
cancelled or ignored when a newer selection supersedes them. - Late
responses cannot overwrite newer state. - Removing a component cleans up
its subscriptions. - A malformed dependency does not crash the whole
screen.

The server describes relationships; the client remains responsible for
executing them safely.

------------------------------------------------------------------------

## 7. Reflex-inspired capabilities

Shopify's Reflex architecture is a useful inspiration for evolving
CoreMVI. The important distinction is that we are adopting ideas, not
copying a Shopify implementation or claiming every proposed extension is
part of Reflex.

### 7.1 Headless core

The feature core should run without rendering a UI.

For Swift: - Core is a pure Swift package. - No UIKit or SwiftUI imports
in Core. - Platform capabilities are injected through protocols. - Core
can run in unit tests, integration tests, or a command-line process.

For Kotlin: - Core is a pure Kotlin/JVM module. - No Android UI
dependencies in Core. - Android-specific capabilities are injected
through interfaces.

A headless core makes it possible to: - Test behavior without a
simulator/emulator. - Run deterministic fixture-based tests. - Inspect a
feature's state without opening a screen. - Drive feature actions from a
CLI or test harness. - Reuse behavior expectations across platforms.

### 7.2 Strict action-in / state-out contract

The view should have a narrow contract:

``` text
View reads state
View sends action
Core processes action
View receives updated state
```

The view must not mutate feature state directly.

A conceptual API:

``` swift
protocol FeatureRuntime {
    associatedtype State
    associatedtype Action

    var state: State { get }

    func send(_ action: Action) async
}
```

The actual production implementation may use type erasure, streams,
actors, or another concurrency model, but the contract should remain
clear.

### 7.3 Introspection

A feature exposes machine-readable information: - Stable feature/node
name. - Current state snapshot as JSON. - Supported actions. - Optional
action input schemas. - Child nodes, if the architecture is composed as
a tree.

This lets tests and tools inspect and drive the same feature runtime
that the UI uses.

### 7.4 CLI / headless driver

A future CoreCLI can expose commands such as:

``` text
list
inspect home
actions home
execute home.selectCategory {"categoryID":"drama"}
snapshot home
reset home
```

The CLI should call the same public dispatch path as the UI. It must not
contain a second implementation of business logic.

### 7.5 Fixture-based deterministic tests

External services are replaced with fixtures: - Mock content
repository. - In-memory watch-list store. - Stub playback service. -
Fixed clock or deterministic ID generator where needed.

Tests can verify complete journeys without network access or UI
automation.

### 7.6 Share behavior tests, not implementation code

iOS and Android can implement their native Core runtimes separately
while consuming the same: - SDUI JSON fixtures. - Action names and
payload contracts. - Journey definitions. - Expected state snapshots. -
Behavioral acceptance tests.

This avoids forcing one UI implementation across platforms while
maintaining behavioral consistency.

------------------------------------------------------------------------

## 8. Interception: an intentional extension to our design

Interception is a capability we want to add to CoreMVI. It is inspired
by the need for controlled boundaries and observability, but it should
be treated as **our extension**, not as a documented core pillar of
Shopify Reflex.

The recommended design has two interception points.

### 8.1 Action interceptors

Run before the reducer.

Potential uses: - Logging and diagnostics. - Authorization checks. -
Validation. - Analytics. - Correlation IDs and metadata enrichment. -
Rejecting disallowed actions. - Debug tracing.

``` text
send(action)
    |
    v
Action Interceptor 1
    |
    v
Action Interceptor 2
    |
    v
Action Interceptor N
    |
    v
Reducer
```

Conceptual Swift contract:

``` swift
protocol ActionInterceptor {
    func intercept(
        _ action: AnyAction,
        context: ActionContext,
        next: ActionNext
    ) async -> ActionResult
}

typealias ActionNext =
    (AnyAction) async -> ActionResult

enum ActionResult {
    case proceed
    case reject(ActionRejection)
}
```

For a production API, use typed action envelopes and explicit
error/rejection models rather than relying on string messages.

### 8.2 Effect interceptors

Wrap effect execution.

Potential uses: - Retry policies. - Caching. - Metrics and timing. -
Distributed tracing. - Cancellation. - Error normalization. - Request
deduplication.

``` text
Effect
  |
  v
Effect Interceptor Pipeline
  |
  v
EffectHandler
  |
  v
Repository / Service
```

### 8.3 Critical invariants

-   Interceptors must not mutate feature state directly.
-   All state changes still pass through the reducer.
-   Rejected actions should have explicit, observable outcomes.
-   Interceptor order must be deterministic and testable.
-   Interceptors should not silently swallow errors.
-   Interceptors must preserve cancellation and task-lifetime semantics.
-   Analytics should not block user-facing state transitions unless
    explicitly required.
-   Avoid putting feature-specific business rules into global
    interceptors.

Use feature-level logic for feature behavior; use interceptors for
cross-cutting policies.

------------------------------------------------------------------------

## 9. Introspection contract

A conceptual Swift protocol:

``` swift
protocol IntrospectionNode {
    var name: String { get }

    func stateSnapshot() throws -> JSONValue

    func availableActions() -> [ActionDescriptor]

    var children: [any IntrospectionNode] { get }
}
```

An action descriptor might include: - Stable action name. -
Human-readable description. - Input JSON schema or payload
description. - Whether the action is currently available, if
availability is state-dependent.

Example conceptual snapshot:

``` json
{
  "name": "home",
  "state": {
    "selectedCategoryId": "drama",
    "isLoading": false,
    "selectedContentId": null
  },
  "actions": [
    {
      "name": "home.selectCategory",
      "input": {
        "categoryID": "string"
      }
    },
    {
      "name": "home.refresh"
    }
  ],
  "children": []
}
```

### Introspection tree for SDUI

Because composition is a tree, introspection can expose both feature and
screen structure:

``` text
App
└── MainTabNavigator
    └── HomeFeature
        └── HomeSDUIScreen
            ├── HeroBanner
            ├── CategorySelector
            ├── TrendingCarousel
            └── ContinueWatching
```

The exact tree can include runtime nodes, feature nodes, and SDUI
component nodes. Avoid exposing unstable implementation details as
public identifiers.

### Action invocation rule

A CLI, test harness, or future agent must invoke actions through the
same public dispatch path as the UI. It must not call reducers or
repositories directly to simulate user behavior.

------------------------------------------------------------------------

## 10. Suggested repository / package structure

The following is a proposed structure. It is a target organization, not
a claim that these files already exist.

``` text
StreamFlix/
│
├── contracts/
│   ├── sdui/
│   │   ├── schema/
│   │   ├── examples/
│   │   └── fixtures/
│   └── journeys/
│       ├── home/
│       ├── search/
│       └── watchlist/
│
├── ios/
│   ├── Packages/
│   │   ├── CoreMVI/
│   │   │   ├── Runtime/
│   │   │   ├── Execution/
│   │   │   ├── Interception/
│   │   │   ├── Introspection/
│   │   │   └── Testing/
│   │   │
│   │   ├── CoreSDUI/
│   │   │   ├── Schema/
│   │   │   ├── Models/
│   │   │   ├── Validation/
│   │   │   ├── ComponentRegistry/
│   │   │   ├── Renderer/
│   │   │   ├── BindingResolver/
│   │   │   ├── DependencyEvaluator/
│   │   │   └── Introspection/
│   │   │
│   │   └── CoreCLI/
│   │       ├── Commands/
│   │       └── Transport/
│   │
│   └── StreamFlixApp/
│       ├── Features/
│       │   ├── Home/
│       │   ├── Details/
│       │   ├── Search/
│       │   └── Downloads/
│       ├── Components/
│       ├── Platform/
│       │   ├── Networking/
│       │   ├── Persistence/
│       │   └── Media/
│       └── AppComposition/
│
└── android/
    ├── core-mvi/
    ├── core-sdui/
    ├── core-cli/              # Optional, if useful on JVM
    └── app/
        ├── features/
        ├── components/
        ├── platform/
        └── composition/
```

### CoreMVI module responsibilities

  Area              Responsibility
  ----------------- ----------------------------------------------------------
  `Runtime`         Store/runtime, action, state, reducer, effect primitives
  `Execution`       IntentHandler, EffectHandler, feature composition
  `Interception`    Action and effect interceptor pipelines
  `Introspection`   Node tree, snapshots, action descriptors
  `Testing`         Test runtime, fixtures, deterministic test helpers

### CoreSDUI module responsibilities

  Area                    Responsibility
  ----------------------- -----------------------------------------------
  `Schema`                Versioned wire contract
  `Models`                Decoded screen/component models
  `Validation`            Schema and semantic validation
  `ComponentRegistry`     Allowlisted component type resolution
  `Renderer`              Recursive native rendering integration
  `BindingResolver`       Resolve state paths into component properties
  `DependencyEvaluator`   Evaluate declared state dependencies
  `Introspection`         Expose the SDUI component tree

------------------------------------------------------------------------

## 11. Security, validation, and resilience

Server-driven configuration is powerful, so the client must enforce
boundaries.

1.  **Allowlisted component types:** JSON can select only components
    registered by the app.
2.  **Allowlisted actions:** JSON can dispatch only explicitly exposed
    actions.
3.  **Typed payload validation:** Validate action payloads before
    dispatch.
4.  **Schema versioning:** Reject or gracefully downgrade unsupported
    schema versions.
5.  **Safe fallbacks:** One malformed component should not necessarily
    take down the whole screen.
6.  **No arbitrary code execution:** Do not interpret JSON as executable
    code or arbitrary class names.
7.  **Dependency validation:** Validate paths, action references, and
    cycles.
8.  **Request freshness:** Cancel or ignore stale asynchronous results.
9.  **Observability:** Record decoding, validation, fallback, and
    dispatch diagnostics.
10. **Remote configuration safety:** Consider version gates, rollout
    controls, and the ability to fall back to a known-good screen
    definition.

------------------------------------------------------------------------

## 12. Testing strategy

Testing should happen at several levels.

### Core unit tests

-   Reducer transitions.
-   Intent-to-action mapping.
-   Effect generation.
-   Action interceptor ordering and rejection.
-   Effect interceptor behavior.
-   Cancellation and stale-response handling.

### SDUI contract tests

-   Decode valid JSON.
-   Reject invalid schema versions.
-   Validate unknown component types.
-   Validate unknown action names.
-   Validate bindings and dependency references.
-   Verify fallback behavior.

### Headless journey tests

Example journey:

``` text
Given Home screen is loaded
When user selects the Drama category
Then selectedCategoryId becomes "drama"
And a category-content load is requested
When the repository returns Drama content
Then the content projection contains the returned items
And the carousel binding resolves to those items
```

### Cross-platform behavior tests

Run the same fixture/journey definitions against iOS Core and Android
Core. Compare observable outcomes and state snapshots, not internal
implementation details.

### UI tests

Keep a smaller set of native UI tests for: - Rendering and layout. -
Accessibility. - Focus and scrolling. - Navigation integration. -
Device-specific behavior. - Screenshot/regression checks.

Headless tests do not replace visual testing; they reduce the amount of
business behavior that must be tested through UI automation.

------------------------------------------------------------------------

## 13. Suggested implementation roadmap

### Phase 1 --- Freeze contracts

-   Define the first version of the SDUI JSON schema.
-   Define component IDs/types and property models.
-   Define action names and payloads.
-   Define binding path syntax.
-   Define dependency semantics.
-   Add representative Home screen fixtures.

### Phase 2 --- Build headless CoreMVI

-   Implement Store/runtime.
-   Implement Action, State, Reducer, and Effect primitives.
-   Keep IntentHandler separate.
-   Implement EffectHandler and injected service protocols.
-   Add unit tests.
-   Verify the package has no UI framework dependencies.

### Phase 3 --- Add interception and introspection

-   Add action interceptor pipeline.
-   Add effect interceptor pipeline.
-   Define deterministic ordering and rejection semantics.
-   Add introspection nodes, state snapshots, and action descriptors.
-   Test dispatch through the same public path used by the UI.

### Phase 4 --- Build CoreSDUI

-   JSON decoder and schema validation.
-   Component registry.
-   Recursive component tree model.
-   Binding resolver.
-   Dependency evaluator.
-   Safe fallback behavior.
-   SDUI introspection.

### Phase 5 --- iOS Home vertical slice

-   SwiftUI renderers for hero, category selector, carousel, and poster
    card.
-   Mock content repository.
-   Home feature using CoreMVI.
-   Category selection updates shared state.
-   Dependency triggers category-specific content load.
-   Carousel updates from resolved state.
-   Headless journey test covers the same flow.

### Phase 6 --- CLI / developer tooling

-   List nodes.
-   Inspect state.
-   List actions.
-   Execute allowlisted actions.
-   Print snapshots and diagnostics.
-   Use fixtures by default.

### Phase 7 --- Android port

-   Implement Kotlin CoreMVI against the same behavior contracts.
-   Implement CoreSDUI models and runtime.
-   Build Compose renderers.
-   Run shared journey fixtures.
-   Compare observable behavior.

### Phase 8 --- Production backend

-   Replace mock JSON source with a versioned API.
-   Add caching and refresh policies.
-   Add telemetry and rollout controls.
-   Introduce backward-compatible schema evolution.

------------------------------------------------------------------------

## 14. Architecture decisions to keep explicit

  ---------------------------------------------------------------------
  Decision                           Recommendation
  ---------------------------------- ----------------------------------
  Does a component call another      No. Communicate through state
  component?                         bindings and declared
                                     dependencies.

  Who owns business state?           CoreMVI feature runtime.

  Who owns cached API collections?   Repositories/data layer where
                                     appropriate.

  Who performs API calls?            Effects executed by EffectHandler
                                     through injected services.

  Can SDUI JSON invoke arbitrary     No. Only allowlisted action
  methods?                           identifiers and validated
                                     payloads.

  Can interceptors mutate state?     No. Reducer remains the state
                                     mutation boundary.

  Does Core depend on                No.
  SwiftUI/Compose?                   

  Does the server dictate native     No. Server selects registered
  rendering code?                    component types and supplies
                                     configuration.

  Is the same code shared across iOS Not necessarily. Share contracts,
  and Android?                       fixtures, and behavior tests; use
                                     native implementations.

  Is interception a Shopify Reflex   Do not claim that. Treat our
  pillar?                            interceptor pipeline as an
                                     intentional extension.

  Is CoreMVI 3.0 the settled name?   No. We discussed it as a possible
                                     name; retain CoreMVI 2.0 unless
                                     the project explicitly adopts a
                                     new version.
  ---------------------------------------------------------------------

------------------------------------------------------------------------

## 15. Shopify Reflex reference and what we are taking from it

Reference: [Shopify Engineering ---
Reflex](https://shopify.engineering/reflex)

The useful architectural ideas discussed from the article are:

-   A UI-free core that can run headlessly.
-   A narrow action-in/state-out contract between views and the core.
-   Introspection that exposes stable names, state, and supported
    actions.
-   A CLI or other external driver that can inspect and drive the core.
-   Tree-based composition and navigation.
-   Injected services rather than hard-wired platform dependencies.
-   Fixture-driven deterministic testing.
-   Sharing behavior tests across platforms rather than forcing shared
    UI/business implementation code.

Our proposed additions for this project include: - The SDUI schema and
native component registry. - Declarative component bindings and
dependency evaluation. - Separate action and effect interceptor
pipelines. - A CoreSDUI runtime integrated with CoreMVI. - A
StreamFlix-oriented cross-platform journey fixture format.

These are design choices for our project, not claims about Shopify's
exact implementation.

------------------------------------------------------------------------

## 16. Glossary

  ---------------------------------------------------------------------
  Term                               Meaning
  ---------------------------------- ----------------------------------
  SDUI                               Server-Driven UI: the server
                                     supplies a declarative description
                                     of the screen.

  Component registry                 Allowlist mapping JSON component
                                     types to native renderers.

  Binding                            A connection between a component
                                     property and a state value.

  Dependency                         A declared relationship that
                                     reacts to a state change, often by
                                     requesting an effect.

  Intent                             A UI-oriented user event or
                                     request.

  Action                             A feature-level event processed by
                                     the reducer.

  IntentHandler                      Converts intents into actions.

  Reducer                            Deterministically transforms state
                                     in response to actions and emits
                                     effects.

  Effect                             A description of asynchronous
                                     work.

  EffectHandler                      Executes effects using injected
                                     services and returns result
                                     actions.

  Interceptor                        A cross-cutting layer that
                                     observes, transforms, enriches, or
                                     rejects actions/effects within
                                     defined rules.

  Headless                           Runs the feature/core without
                                     rendering UI.

  Introspection                      Machine-readable inspection of
                                     runtime nodes, state, and
                                     supported actions.

  Fixture                            Controlled test data or fake
                                     service response.

  Journey test                       A test that drives a sequence of
                                     actions and verifies observable
                                     outcomes.
  ---------------------------------------------------------------------

------------------------------------------------------------------------

## 17. Immediate next implementation target

Build the smallest complete vertical slice before generalizing every
abstraction:

1.  A pure Swift CoreMVI package.
2.  A Home feature with `IntentHandler`, actions, reducer, effects, and
    injected repository.
3.  Action and effect interceptor pipelines with tests.
4.  An introspection node exposing Home state and supported actions.
5.  A CoreSDUI JSON fixture containing a category selector and content
    carousel.
6.  A binding/dependency flow where selecting a category updates the
    selector and loads the corresponding carousel content.
7.  A SwiftUI renderer that consumes the state and screen definition.
8.  A headless journey test that proves the same behavior without
    SwiftUI.

Once this iOS slice is stable, implement the equivalent Kotlin runtime
and Compose renderer against the same JSON and behavior fixtures.

------------------------------------------------------------------------

*End of architecture notes.*
