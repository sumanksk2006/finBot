# CoreMVI --- Complete Architecture & Evolution

## From a reusable Swift MVI package to Reflex-inspired headless Core and SDUI integration

**Document type:** Master architecture record\
**Project:** Reusable CoreMVI framework and its StreamFlix SDUI
application\
**Consolidated:** 23 November 2026\
**Status:** Architecture history and proposed target design. Code
examples are illustrative contracts unless explicitly identified as
implemented.

> **Relationship to the companion document:** This is the master record
> for the reusable CoreMVI architecture and its evolution. The companion
> [StreamFlix SDUI + CoreMVI / Reflex-Inspired
> Architecture](StreamFlix_Sdui_CoreMVI_Architecture_Updated.md) focuses
> on applying the core to a server-driven streaming app. Read this
> document first for the reusable foundation, then the companion for
> SDUI, component dependencies, and StreamFlix integration.

------------------------------------------------------------------------

# Part I --- Why CoreMVI exists

## 1. Original goal

The initial objective was to create a reusable **CoreMVI Swift Package**
that provides the common execution machinery for every feature module,
rather than having each feature reinvent its own Store, reducer
orchestration, effect execution, cancellation, and instrumentation.

The desired result:

-   A feature defines its own business concepts: Intent, Action, State,
    Effect, Reducer, repository interfaces, and UI.
-   CoreMVI provides the generic machinery to connect those pieces.
-   A feature should not need to implement a custom Store every time.
-   Shared capabilities such as logging, analytics hooks, performance
    metrics, effect execution, and cancellation should be available
    consistently.
-   The package should be usable by multiple feature modules and should
    not know feature-specific business rules.
-   SwiftUI/UIKit views remain presentation adapters and do not own
    business logic.

The first example used to make the architecture concrete was a
multi-step Registration Wizard.

## 2. The central execution flow

The recurring flow we designed is:

``` text
SwiftUI / UIKit View
        |
        | sends a UI intent
        v
      Intent
        |
        v
  IntentHandler
        |
        | maps one intent to one or more feature actions
        v
      Action
        |
        v
  Store / Runtime orchestration
        |
        v
     Reducer
        |
        +------------------------------+
        |                              |
        v                              v
   New State                         Effect
        |                              |
        v                              v
   View observes                 EffectHandler
   updated State                       |
                                       v
                                  Repository
                                       |
                                       v
                                  API / Storage
                                       |
                                       v
                                  Result Action
                                       |
                                       v
                                    Reducer
                                       |
                                       v
                                   New State
```

The core invariant is **unidirectional data flow**:

1.  Views express what happened; they do not decide business outcomes.
2.  Intents are translated into feature actions.
3.  Reducers are the state-transition authority.
4.  Effects describe work that must happen outside the reducer.
5.  Effect handlers execute that work through repositories/services.
6.  Results return as actions.
7.  The reducer produces the next state.
8.  Views render the state they observe.

No API response should directly mutate UI state, and no view should call
a repository to implement feature behavior.

------------------------------------------------------------------------

# Part II --- The foundational concepts

## 3. Intent vs Action

These concepts were deliberately kept separate.

### Intent

An **Intent** represents a presentation-level request or user
interaction.

Examples: - `ContinueTapped` - `CountrySelectionChanged(countryID:)` -
`BackTapped` - `SubmitTapped`

It is usually phrased in the language of the interaction.

### Action

An **Action** represents a meaningful event in the feature's state
machine.

Examples: - `RegistrationAction.countrySelected(countryID:)` -
`RegistrationAction.stepAdvanced` -
`RegistrationAction.submitRequested` -
`RegistrationAction.submissionSucceeded(reference:)` -
`RegistrationAction.submissionFailed(error:)`

Actions can originate from user intents, asynchronous results, lifecycle
events, or other explicitly supported sources.

### Why not send Actions directly from every view?

Sending Actions directly can be a valid simpler MVI design, but the
separate IntentHandler was chosen for this architecture because it
creates a clear presentation-to-feature boundary.

An IntentHandler can: - Map one intent to one or multiple actions. -
Normalize input. - Translate UI-specific events into feature language. -
Keep view naming and interaction details out of reducers. - Provide a
focused unit-test boundary.

It should **not**: - Make network calls. - Mutate State directly. - Own
repositories. - Become a second reducer. - Hide business rules that
belong in the reducer/domain layer.

The architecture decision from the November 4 discussion was to retain a
separate `IntentHandler`. An alternative with the Store mapping Intent
directly to Action was explored, but the separate handler was preferred
for separation of concerns.

## 4. Action vs Message

In the broader Reflex-inspired evolution, the term **Message** may be
useful as a transport/envelope concept for anything entering a runtime.
It should not casually replace the feature-level Action concept.

A possible distinction:

-   `Action`: typed feature event that the reducer understands.
-   `Message`: runtime envelope that may carry an action plus metadata
    such as source, correlation ID, timestamp, or tracing context.

Keep the public vocabulary simple unless the envelope is needed. Avoid
introducing two names for the same concept.

## 5. State

State is the minimum feature-owned information needed to represent the
current feature situation and render its UI.

Typical examples: - Current wizard step. - Selected country ID. -
User-entered form values. - Submission status. - Validation messages. -
Selected content identifier. - Loading/error status.

State should be: - Explicit. - Value-oriented and predictable. -
Suitable for deterministic reducer tests. - Observable by the UI. - Free
of platform UI objects. - Free of service/repository instances.

### State should not become a copy of the database

A recurring design preference was to keep State lightweight and avoid
putting entire API-owned collections into it just because a view needs
to display them.

For example, a country dropdown: - The country catalogue can live in a
repository/catalog/cache. - State can hold the selected country ID and
relevant loading/error/selection information. - A UI projection or query
can provide the display options. - The repository decides whether to use
cached data or fetch remotely.

For a StreamFlix content carousel: - The content repository owns cached
catalogue data. - Feature state tracks interaction and the data
projection required by the current screen. - Avoid duplicating a
complete catalogue in every component's local state.

This is not a prohibition on putting data in State. If a screen needs a
stable, serializable snapshot of a collection to render and test, it may
be appropriate. The decision should be intentional, not automatic.

## 6. Reducer

The Reducer is the deterministic state-transition function.

Conceptual contract:

``` swift
protocol FeatureReducer {
    associatedtype State
    associatedtype Action
    associatedtype Effect

    func reduce(
        state: inout State,
        action: Action
    ) -> [Effect]
}
```

The reducer: - Receives the current State and an Action. - Makes
synchronous state changes. - Returns zero or more Effects. - Does not
call APIs. - Does not access UIKit/SwiftUI. - Does not perform
persistence. - Does not launch untracked asynchronous work.

The reducer is the single authority for feature state transitions.
Interceptors, repositories, views, and effect handlers must not bypass
it to mutate State.

## 7. Effect

An Effect describes work that cannot or should not happen inside the
reducer.

Examples: - Submit registration. - Fetch profile. - Load content for a
category. - Persist a watch-list update. - Prepare playback. - Read
local storage.

An Effect should carry the inputs required to execute the operation. It
should not be a place to hide arbitrary mutable state.

## 8. EffectHandler

The EffectHandler executes Effects.

Responsibilities: - Match an Effect to the appropriate
repository/service call. - Perform asynchronous work. - Map
success/failure into result Actions. - Respect cancellation. - Avoid
directly mutating feature State.

Example:

``` swift
protocol EffectHandling {
    associatedtype Effect
    associatedtype Action

    func handle(_ effect: Effect) async -> [Action]
}
```

A production runtime may emit actions as an asynchronous stream rather
than returning an array. The important contract is that results re-enter
through the normal action dispatch/reducer path.

## 9. Repository and service boundaries

Repositories own data access and data-source policy: - API calls. -
Persistence. - Cache lookup and update. - Mapping network models into
domain/read models. - Deciding whether cached data is sufficient or a
remote fetch is needed.

Services provide capabilities such as: - Authentication. - Clock. -
Analytics transport. - Media/playback integration. - Device-specific
operations through adapters.

The feature should depend on protocols/abstractions, not concrete
networking or platform frameworks.

------------------------------------------------------------------------

# Part III --- Reusable CoreMVI package

## 10. What belongs in CoreMVI vs a feature

  CoreMVI package owns                   Feature module owns
  -------------------------------------- --------------------------------------
  Generic Store/runtime orchestration    Feature Intent types
  Generic dispatch pipeline              Feature Action types
  Base contracts/protocols               Feature State
  Reducer invocation mechanics           Feature Reducer logic
  Effect scheduling/execution contract   Feature Effect types
  Cancellation infrastructure            Feature IntentHandler mapping
  Interception/plugin hooks              Feature EffectHandler implementation
  Instrumentation extension points       Feature repositories/protocols
  Test utilities                         Feature UI and feature composition
  Generic lifecycle mechanics            Feature-specific business rules

The guiding principle:

> CoreMVI executes features; it does not know what a feature means.

Do not put `RegistrationAction`, `MovieAction`, `PaymentState`, or
domain-specific validation into the generic CoreMVI package.

## 11. Base contracts

The initial package proposal used lightweight base protocols.
`BaseIntent` and `BaseAction` were intentionally empty marker protocols
so that feature modules retain their own strongly typed enums and
associated values.

Illustrative contracts:

``` swift
public protocol BaseIntent: Sendable {}

public protocol BaseAction: Sendable {}

public protocol BaseState: Sendable {}

public protocol BaseEffect: Sendable {}
```

The precise constraints (`Sendable`, `Equatable`, `Codable`, etc.)
should be added only where the runtime needs them. Avoid imposing
`Codable` on every State merely for debugging; introspection can use an
explicit snapshot mapper.

A reducer contract can then connect a feature's associated types:

``` swift
public protocol BaseReducer {
    associatedtype State: BaseState
    associatedtype Action: BaseAction
    associatedtype Effect: BaseEffect

    func reduce(
        state: inout State,
        action: Action
    ) -> [Effect]
}
```

This is a conceptual baseline. A compiling package must settle details
such as synchronous vs asynchronous reducer execution, effect identity,
error representation, and whether the reducer is a value or
actor-isolated object.

## 12. Generic BaseFeatureStore

The key reusable abstraction is a generic Store/runtime that coordinates
the pipeline. Features should not each build their own Store.

A conceptual shape:

``` swift
@MainActor
public protocol FeatureRuntimeProtocol {
    associatedtype Intent: BaseIntent
    associatedtype Action: BaseAction
    associatedtype State: BaseState

    var state: State { get }

    func send(_ intent: Intent) async
}
```

A generic implementation needs collaborators such as: - IntentHandler -
Reducer - EffectHandler / EffectExecutor - State publisher/stream -
Cancellation manager - Optional interceptor/plugin pipeline -
Runtime/lifecycle configuration

The runtime's job is orchestration, not feature business logic.

### Typical dispatch algorithm

``` text
send(intent)
  1. Validate runtime is active.
  2. Pass intent to IntentHandler.
  3. Receive zero or more Actions.
  4. For each Action:
       a. Run action interception.
       b. If accepted, invoke Reducer.
       c. Publish the resulting State.
       d. Schedule returned Effects.
  5. Execute Effects through EffectHandler.
  6. Convert outcomes to result Actions.
  7. Send result Actions through the same action pipeline.
```

Important: the asynchronous effect result must not use a shortcut to
mutate state. It re-enters the action pipeline.

## 13. FeatureDefinition / feature assembly

A feature needs a composition point that wires its typed parts together.

A conceptual definition:

``` swift
struct FeatureDefinition<Intent, Action, State, Effect> {
    let intentHandler: any IntentHandling
    let reducer: any Reducing
    let effectHandler: any EffectHandling
}
```

The final Swift implementation may use generic associated types,
closures, type erasure, or a builder. The goal is to make feature wiring
explicit and testable without forcing the generic runtime to understand
each feature.

A feature factory/composition root should: - Construct repositories and
service adapters. - Construct the feature's handlers and reducer. -
Register interceptors/plugins. - Create the generic runtime. - Expose
the runtime to the presentation layer.

Avoid service-locator access from inside feature logic.

## 14. Suggested CoreMVI package organization

``` text
CoreMVI/
├── Contracts/
│   ├── BaseIntent.swift
│   ├── BaseAction.swift
│   ├── BaseState.swift
│   ├── BaseEffect.swift
│   └── FeatureProtocols.swift
│
├── Runtime/
│   ├── BaseFeatureStore.swift
│   ├── StoreState.swift
│   ├── DispatchContext.swift
│   └── RuntimeLifecycle.swift
│
├── Execution/
│   ├── IntentHandler.swift
│   ├── EffectHandler.swift
│   ├── EffectExecutor.swift
│   └── FeatureDefinition.swift
│
├── Concurrency/
│   ├── CancellationManager.swift
│   ├── EffectID.swift
│   └── TaskRegistry.swift
│
├── Interception/
│   ├── Middleware.swift
│   ├── ExecutionPlugin.swift
│   └── InterceptorPipeline.swift
│
├── Instrumentation/
│   ├── Logger.swift
│   ├── Analytics.swift
│   └── PerformanceMetrics.swift
│
└── Testing/
    ├── TestStore.swift
    ├── TestClock.swift
    ├── EffectTestSupport.swift
    └── FixtureServices.swift
```

This is a logical grouping. It may be consolidated into fewer folders in
the actual Swift package.

------------------------------------------------------------------------

# Part IV --- IntentHandler, EffectHandler, Store, and middleware

## 15. Why both IntentHandler and EffectHandler?

They solve different problems.

  -----------------------------------------------------------------------
  Component         Input             Output            Responsibility
  ----------------- ----------------- ----------------- -----------------
  IntentHandler     UI Intent         Feature Action(s) Translate
                                                        presentation
                                                        events

  Reducer           State + Action    New State +       Deterministic
                                      Effect(s)         state transition

  EffectHandler     Effect            Result Action(s)  Execute
                                                        asynchronous work

  Store/runtime     Intent or Action  Updated           Orchestrate the
                    entry             observable State  full pipeline

  Repository        Data request      Domain            Own data access
                                      data/result       and caching
  -----------------------------------------------------------------------

Keeping both handlers makes the boundaries explicit: - IntentHandler
translates user intent. - EffectHandler performs side effects.

They should not become interchangeable generic middleware.

## 16. Store vs Reducer vs EffectHandler

A simple analogy:

-   **Store:** the conductor; coordinates the sequence.
-   **Reducer:** the rulebook for how the state changes.
-   **EffectHandler:** the worker that performs external/asynchronous
    tasks.
-   **IntentHandler:** the translator between UI language and feature
    action language.
-   **Repository:** the data-access owner.

The Store should not become a giant feature object. It should coordinate
generic mechanics.

## 17. Middleware naming and plugin exploration

We explored whether a single Middleware concept was enough and whether a
better name would communicate the extension point.

An early proposal included an `ExecutionPlugin` with lifecycle hooks:

``` swift
protocol ExecutionPlugin {
    func willProcess(/* context */)
    func didProcess(/* context */)
    func willExecute(/* context */)
    func didExecute(/* context */)
}
```

Potential use cases: - Analytics. - Logging. - Performance
measurement. - Tracing. - Diagnostics.

The term `ExecutionPlugin` can communicate a broader lifecycle observer
than `Middleware`. However, it should not become an unrestricted hook
that can secretly alter feature behavior.

A clearer modern distinction is:

-   **Interceptor:** wraps a specific operation and may continue,
    transform, or reject it under a defined contract.
-   **Observer/plugin:** receives lifecycle events for analytics,
    metrics, and diagnostics; ideally cannot change execution.
-   **Effect policy:** retry, caching, deduplication, timeout, and
    cancellation behavior around effect execution.

Do not combine all of these into one loosely defined middleware
protocol.

------------------------------------------------------------------------

# Part V --- Shared capabilities and production concerns

## 18. Cross-cutting capabilities

The reusable package was expected to provide standard extension points
so features do not each reinvent instrumentation.

  -----------------------------------------------------------------------
  Capability              Suggested owner         Notes
  ----------------------- ----------------------- -----------------------
  Logging                 Core instrumentation    Structured events;
                          adapter                 configurable levels

  Analytics               Execution               Feature supplies event
                          observer/plugin         meaning; shared
                                                  transport/pipeline

  Performance metrics     Execution               Measure
                          observer/plugin         action/reducer/effect
                                                  duration

  Networking              Platform/data layer     Inject service or
                                                  repository; avoid
                                                  embedding HTTP in
                                                  CoreMVI

  Persistence             Repository/service      Inject storage
                                                  capability

  Cancellation            Core runtime            Track effect identity
                                                  and task lifetime

  Error mapping           Effect/data boundary    Convert infrastructure
                                                  errors into typed
                                                  feature outcomes

  Navigation              Explicit feature        Avoid hard-coding
                          action + platform       navigation framework in
                          adapter                 Core

  Lifecycle               Runtime + platform      Start, suspend, resume,
                          adapter                 and dispose
                                                  deliberately
  -----------------------------------------------------------------------

### Networking clarification

CoreMVI should not become a networking library. It should provide a
consistent way to execute Effects and inject dependencies. HTTP clients,
authentication interceptors, request encoding, and transport policies
belong in a networking/data package.

A shared networking service can be injected into feature repositories.
The EffectHandler invokes repository methods, not raw URLs.

## 19. Effect identity and cancellation

Cancellation was identified as a production requirement.

Examples: - User selects category A, then quickly selects category B.
The result for A must not overwrite B. - Search query changes rapidly.
Previous searches should be cancelled or ignored. - A feature is
dismissed. Its scoped tasks should be cancelled. - A refresh supersedes
a previous refresh.

A robust effect model should support: - Stable effect IDs. - Optional
cancellation groups/keys. - Replace-latest or cancel-in-flight
policies. - Explicit task ownership and lifetime. - Handling of late
results. - Deterministic cancellation tests.

Conceptual effect metadata:

``` swift
struct EffectMetadata: Sendable {
    let id: EffectID
    let cancellationKey: String?
    let policy: CancellationPolicy
}
```

The exact API is a design decision to finalize during implementation. Do
not use unstructured `Task {}` calls without retaining ownership or
defining cancellation behavior.

## 20. Actor and concurrency strategy

The production implementation explored using Swift actors to protect
runtime state and coordinate concurrent effects.

A sensible separation is:

-   A Store/runtime actor or explicitly isolated runtime owns mutable
    state and serializes action processing.
-   Reducer execution occurs within the chosen state-isolation boundary.
-   Effect work can run asynchronously outside the serialized state
    mutation step.
-   Effect results are sent back to the runtime as actions.
-   UI observation is bridged to `AsyncSequence`, `AsyncStream`, or an
    observable adapter appropriate to the deployment target.

The important guarantee is not "everything must be an actor." It is: 1.
State mutation is serialized. 2. Effects do not block state processing.
3. Results re-enter through action dispatch. 4. Cancellation is
explicit. 5. UI isolation is explicit, commonly `@MainActor` for
presentation-facing observation.

Avoid holding a Store actor across a long network await if that would
serialize unrelated user actions. Schedule the effect, release the
state-processing path, then dispatch its result when complete.

## 21. Error handling

Separate: - Infrastructure errors (network unavailable, timeout,
decoding). - Domain failures (business rule rejected). - Validation
failures (invalid form fields). - Cancellation (usually not a
user-facing failure). - Unexpected programming errors.

The repository/data layer maps transport details into stable
domain-facing results. The feature maps those results to actions. The
reducer decides the state representation.

Do not put arbitrary `Error` objects into long-lived State unless there
is a clear serialization and UI policy. Prefer typed, display-safe error
models.

## 22. Lifecycle and dependency injection

Production runtime should define: - How the feature is constructed. -
Who owns it. - When it starts. - When it is suspended or resumed. - When
it is disposed. - Which effects are cancelled on disposal. - How
repositories/services are scoped.

Dependency injection should happen at the feature composition root. The
core should not reach globally into app singletons.

------------------------------------------------------------------------

# Part VI --- Example: reusable Registration Wizard

## 23. Feature requirements

The Registration Wizard was the first example used to validate the
generic package.

Illustrative steps: 1. Personal Details. 2. Address. 3. Verification. 4.
Confirmation.

Example behavior: - Country defaults to US. - User can choose a
different country. - Country list is owned by a catalogue/repository,
not copied wholesale into feature State without need. - The user enters
form data. - Submit triggers an Effect. - Repository submits
registration. - Success/failure comes back as an Action. - Reducer
updates the feature State.

## 24. Example feature types

``` swift
enum RegistrationIntent: BaseIntent {
    case continueTapped
    case backTapped
    case countrySelected(String)
    case submitTapped
}

enum RegistrationAction: BaseAction {
    case countrySelected(String)
    case stepChanged(RegistrationStep)
    case submitRequested
    case submissionSucceeded(reference: String)
    case submissionFailed(RegistrationFailure)
}

struct RegistrationState: BaseState {
    var step: RegistrationStep = .personalDetails
    var selectedCountryID: String = "US"
    var isSubmitting = false
    var submissionReference: String?
    var error: RegistrationFailure?
}

enum RegistrationEffect: BaseEffect {
    case submit(RegistrationPayload)
}
```

## 25. IntentHandler

``` swift
struct RegistrationIntentHandler {
    func handle(_ intent: RegistrationIntent) -> [RegistrationAction] {
        switch intent {
        case .continueTapped:
            return [.stepChanged(.address)]

        case .backTapped:
            return [.stepChanged(.personalDetails)]

        case .countrySelected(let countryID):
            return [.countrySelected(countryID)]

        case .submitTapped:
            return [.submitRequested]
        }
    }
}
```

This is illustrative. A real multi-step wizard should calculate the
next/previous step based on current state rather than hard-code every
transition in the IntentHandler. The IntentHandler can receive a context
or the Store can map a state-aware intent into an action, provided the
reducer remains the state-transition authority.

## 26. Reducer

``` swift
struct RegistrationReducer {
    func reduce(
        state: inout RegistrationState,
        action: RegistrationAction
    ) -> [RegistrationEffect] {
        switch action {
        case .countrySelected(let countryID):
            state.selectedCountryID = countryID
            return []

        case .stepChanged(let step):
            state.step = step
            return []

        case .submitRequested:
            state.isSubmitting = true
            state.error = nil
            return [.submit(/* build payload from state */)]

        case .submissionSucceeded(let reference):
            state.isSubmitting = false
            state.submissionReference = reference
            state.step = .confirmation
            return []

        case .submissionFailed(let failure):
            state.isSubmitting = false
            state.error = failure
            return []
        }
    }
}
```

The payload construction should be a deliberate mapping, not an untyped
global lookup. In a concrete implementation, the reducer can return an
effect with the required payload or the effect handler can receive a
well-defined request value.

## 27. EffectHandler and repository

``` swift
protocol RegistrationRepository {
    func submit(_ payload: RegistrationPayload) async
        -> Result<String, RegistrationFailure>
}

struct RegistrationEffectHandler {
    let repository: any RegistrationRepository

    func handle(
        _ effect: RegistrationEffect
    ) async -> [RegistrationAction] {
        switch effect {
        case .submit(let payload):
            switch await repository.submit(payload) {
            case .success(let reference):
                return [.submissionSucceeded(reference: reference)]
            case .failure(let failure):
                return [.submissionFailed(failure)]
            }
        }
    }
}
```

The runtime dispatches returned result actions through the same reducer
pipeline.

## 28. What this example proves

-   Feature-specific enums and State remain outside CoreMVI.
-   The reusable Store handles orchestration.
-   The IntentHandler maps UI events.
-   The reducer owns state transitions.
-   The EffectHandler performs asynchronous work.
-   The repository owns API/data access.
-   The package can support other features without knowing their
    business logic.

------------------------------------------------------------------------

# Part VII --- Feature component/template design

## 29. Reusable feature assembly

We discussed a reusable feature template that provides common
capabilities by default while allowing consumers to extend or replace
feature-specific behavior.

A feature should provide: - Intent and Action types. - State and Effect
types. - IntentHandler. - Reducer. - EffectHandler. - Repository/service
protocols. - Feature composition/factory. - UI and state projection.

CoreMVI provides: - Generic Store/runtime. - Dispatch and reducer
orchestration. - Effect scheduling. - Cancellation. -
Interception/observation extension points. -
Logging/analytics/performance integration points. - Test support. -
Lifecycle handling.

The consumer should be able to: - Use default analytics/metrics
behavior. - Add feature-specific analytics. - Override selected policies
through explicit composition. - Inject a mock repository for tests. -
Use custom effect policies where required.

Avoid a massive inheritance-based `BaseFeature` class that forces every
feature into the same shape. Prefer protocols, composition, generic
runtime types, and explicit configuration.

## 30. Feature composition tree

Tree composition became an explicit choice in the later architecture
discussion.

A feature may compose child features or subflows:

``` text
App
└── MainFeature
    ├── HomeFeature
    ├── SearchFeature
    └── AccountFeature
```

A complex feature can contain child components:

``` text
RegistrationFeature
├── PersonalDetailsStep
├── AddressStep
├── VerificationStep
└── ConfirmationStep
```

Composition should define: - Parent/child ownership. - State visibility
boundaries. - Action forwarding rules. - Effect/service scope. - Child
lifecycle and cancellation. - Introspection traversal. - Navigation
ownership.

Do not create shared mutable State across unrelated features just to
simplify communication. Prefer explicit parent-child messages/actions or
shared repository/service abstractions.

------------------------------------------------------------------------

# Part VIII --- CoreMVI 2.0 and Shopify Reflex-inspired evolution

## 31. Why evolve the original CoreMVI?

The original CoreMVI focused on reusable MVI execution. The later
discussions introduced additional requirements: - Run feature logic
without UI. - Inspect a running feature programmatically. - Invoke
supported actions from a test harness or CLI. - Compose features and
components as a tree. - Make the action/state boundary explicit. -
Integrate with server-driven UI. - Add action/effect interception as
cross-cutting extension points.

These additions extend the original reusable MVI foundation rather than
replacing its unidirectional flow.

The user-selected architecture decisions for this evolution were: -
Separate IntentHandler. - Tree composition. - Both unit and integration
testing.

The name **CoreMVI 2.0** was used for the reusable Swift package
request. A later possible name such as CoreMVI 3.0 was discussed as an
option, but it was not approved as a settled rename. Keep CoreMVI 2.0
unless explicitly versioned otherwise.

## 32. Shopify Reflex: concepts adopted as inspiration

Reference: [Shopify Engineering ---
Reflex](https://shopify.engineering/reflex)

The Reflex discussion focused on these ideas: 1. **UI-free Core:** core
logic has no UI framework dependency and can run headlessly. 2.
**Action-in/state-out contract:** views read state and send actions
through a narrow interface. 3. **Introspection:** runtime nodes expose
stable names, state snapshots, and supported actions. 4.
**CLI/tooling:** an external driver can inspect the tree and invoke
supported actions. 5. **Tree composition:** application/feature
structure can be traversed. 6. **Injected services:** dependencies are
supplied rather than hard-wired to UI/platform code. 7. **Fixture-driven
tests:** deterministic behavior can be tested without a running UI or
live backend. 8. **Share tests, not code:** iOS and Android can
implement native runtimes while validating the same behavioral
contracts.

These are the Reflex-inspired elements we want to incorporate. This
document does not claim that our implementation is Shopify's code or
that every proposed capability is a direct Reflex feature.

## 33. Headless Core

For iOS: - CoreMVI is a pure Swift package. - No SwiftUI/UIKit imports
in the core target. - Platform services are injected. - Core can run in
tests or a command-line executable.

For Android: - Core is a pure Kotlin/JVM module. - No Android UI
dependencies in the core module. - Android platform services are
injected through adapters.

Headless does not mean "no UI app." It means the business/runtime core
does not require UI rendering to execute and test its behavior.

## 34. Strict view contract

``` text
View reads State
View sends Intent/Action through public dispatch
Core processes
View observes updated State
```

The UI should not: - Mutate State directly. - Invoke the reducer. - Call
repositories to bypass the feature. - Implement feature business rules.

The public API can accept Intent at the presentation boundary and
convert it into Action internally. The runtime may also expose carefully
controlled action dispatch for tests/tools. Both routes must converge on
the same reducer pipeline.

## 35. Introspection

An introspection node can expose: - Stable node name. - JSON state
snapshot. - Supported action descriptors. - Optional input schemas. -
Child nodes.

Conceptual protocol:

``` swift
protocol IntrospectionNode {
    var name: String { get }
    func stateSnapshot() throws -> JSONValue
    func availableActions() -> [ActionDescriptor]
    var children: [any IntrospectionNode] { get }
}
```

Introspection should not expose arbitrary internal objects, secrets,
credentials, or unstable implementation details. Snapshots should use
explicit serializers/projections rather than assuming every State is
automatically JSON encodable.

## 36. CLI / external driver

A future CoreCLI could provide commands such as:

``` text
list
inspect home
actions home
execute home.selectCategory {"categoryID":"drama"}
snapshot home
reset home
```

The CLI is a client of the public Core runtime. It must invoke the same
action dispatch path as the UI and tests. It must not call the reducer
or repository directly to simulate behavior.

## 37. Interception: our extension

We explicitly discussed interception as a capability to add to our
CoreMVI design. It should be described as **our architectural
extension**, not as one of Shopify Reflex's documented pillars.

### Action interception

Runs before reducer processing and can support: - Logging. -
Validation. - Authorization. - Analytics metadata. - Correlation IDs. -
Diagnostics. - Explicit rejection.

Conceptual contract:

``` swift
protocol ActionInterceptor {
    func intercept(
        _ action: AnyAction,
        context: ActionContext,
        next: ActionNext
    ) async -> ActionResult
}
```

### Effect interception

Wraps effect execution and can support: - Retry. - Caching. -
Timing/metrics. - Tracing. - Deduplication. - Cancellation policy. -
Error normalization.

### Rules

-   Interceptors cannot mutate State directly.
-   Accepted actions still go to the reducer.
-   Effect results still return as actions.
-   Interceptor order is deterministic.
-   Rejection/error semantics are explicit.
-   Feature-specific business rules stay in the feature, not global
    interceptors.

## 38. CoreMVI vs CoreSDUI

Keep the two responsibilities separate:

**CoreMVI** - Feature state machine. - Actions and reducers. - Effects
and execution. - Intent mapping. - Interception. - Introspection. -
Headless runtime.

**CoreSDUI** - JSON schema and decoding. - Component tree. - Component
registry. - Binding resolution. - Dependency evaluation. - Declarative
action interpretation. - Native renderer integration.

CoreSDUI translates SDUI component events into feature intents/actions.
It does not become a second business state manager.

------------------------------------------------------------------------

# Part IX --- Testing strategy

## 39. Unit tests

Test independently: - IntentHandler mapping. - Reducer transitions. -
Effect generation. - EffectHandler/repository mapping. - Interceptor
ordering and rejection. - Cancellation behavior. - State snapshot
serialization. - Action descriptor generation.

## 40. Runtime integration tests

Use a test runtime and fake services to verify: - Intent produces the
expected action. - Action updates State. - Reducer emits Effect. -
EffectHandler calls the expected repository method. - Result Action
re-enters the reducer. - Final State is correct. - Cancellation prevents
stale results from winning.

## 41. Headless journey tests

Example:

``` text
Given the Registration feature is at the Address step
When the user selects Canada
And submits the registration
And the fake repository returns success
Then the feature reaches Confirmation
And the submission reference is present
```

The same style of journey can be expressed in a shared contract format
and run against iOS and Android implementations.

## 42. UI tests

Headless tests do not replace UI tests. Keep native UI tests for: -
Layout and rendering. - Accessibility. - Focus and keyboard behavior. -
Navigation integration. - Device-specific behavior. - Visual
regressions.

The goal is to test business behavior primarily at the Core level and
reserve UI automation for behavior that genuinely depends on
rendering/platform integration.

------------------------------------------------------------------------

# Part X --- Evolution timeline and decision record

## 43. Timeline

### 4 November 2026 --- Reusable CoreMVI foundation

The initial discussions established: - A reusable CoreMVI Swift
Package. - Generic base contracts. - A reusable BaseFeatureStore. - The
pipeline from View through Intent, Action, Reducer, State/Effect,
EffectHandler, Repository, and result Action. - Registration Wizard as
the example feature. - Repository ownership of catalogues/cached data
rather than bloating State. - Shared analytics, performance, logging,
networking integration points. - Cancellation and testing as production
requirements.

The discussion explored both a direct Intent-to-Action mapping in the
Store and a separate IntentHandler. The separate IntentHandler was
selected to preserve separation of concerns.

### 4--6 November 2026 --- Responsibilities and extension points

The design was refined around: - IntentHandler vs EffectHandler. -
Reducer purity. - Store orchestration. - Optional middleware/plugin
hooks. - ExecutionPlugin lifecycle callbacks for
analytics/performance/logging. - Cancellation management and effect
execution. - Naming and responsibility boundaries for middleware.

### 17--19 November 2026 --- CoreMVI 2.0 and SDUI

The work expanded into: - Server-driven screen definitions. - Component
registry and recursive rendering. - Bindings and component
dependencies. - Tree composition. - Shared behavior contracts across iOS
and Android. - Mock JSON backend. - StreamFlix Home vertical slice.

### 19--20 November 2026 --- Shopify Reflex research and integration

The discussion incorporated: - Headless Core. - Strict action/state
contract. - Introspection. - CLI/headless driver. - Fixture-based
tests. - Tree inspection. - Cross-platform behavior parity. - Action and
effect interception as our extension.

A standalone CoreMVI 2.0 package artifact was also requested in the
prior work. The architecture should continue to be treated as evolving
until the package is built, compiled, and its tests are run in the
target toolchain.

## 44. Settled decisions vs open implementation choices

  ---------------------------------------------------------------------
  Topic                              Current position
  ---------------------------------- ----------------------------------
  Reusable CoreMVI package           Core objective

  Separate IntentHandler             Selected

  Separate EffectHandler             Selected

  Reducer owns state transitions     Required invariant

  Repository owns data access/cache  Selected responsibility boundary

  Feature State should stay          Strong preference; use judgment
  lightweight                        for renderable snapshots

  Generic reusable Store/runtime     Core requirement

  Tree composition                   Selected

  Unit + integration tests           Selected

  iOS first, Android port            Selected implementation sequence
                                     for StreamFlix

  Headless core                      Adopt Reflex-inspired capability

  Introspection                      Adopt Reflex-inspired capability

  CLI                                Planned developer/test tooling

  Action/effect interception         Proposed CoreMVI extension

  Actor implementation               Production strategy to finalize;
                                     serialize state safely without
                                     blocking on long effects

  Exact base protocol constraints    Finalize against deployment
                                     targets and concurrency
                                     requirements

  Exact effect identity/cancellation Finalize during implementation
  API                                

  Middleware vs plugin naming        Prefer explicit interceptor vs
                                     observer/plugin roles

  CoreMVI 3.0 rename                 Not approved; CoreMVI 2.0 remains
                                     the working name
  ---------------------------------------------------------------------

------------------------------------------------------------------------

# Part XI --- Recommended implementation order

## 45. Build the reusable core before the SDUI framework

1.  Freeze CoreMVI contracts and public API.
2.  Implement generic runtime/Store.
3.  Implement IntentHandler and Reducer orchestration.
4.  Implement EffectHandler/EffectExecutor.
5.  Implement state observation.
6.  Add effect identity and cancellation.
7.  Add logging/analytics/performance observer hooks.
8.  Add action/effect interceptor pipelines.
9.  Add introspection.
10. Add test runtime and fixture services.
11. Validate with Registration Wizard.
12. Add headless CLI only after the runtime API is stable.

## 46. Then integrate CoreSDUI and StreamFlix

1.  Freeze versioned SDUI JSON schema.
2.  Implement CoreSDUI decoding and validation.
3.  Implement component registry and recursive component tree.
4.  Implement bindings.
5.  Implement dependency evaluator.
6.  Connect component events to CoreMVI intents/actions.
7.  Build the iOS Home screen vertical slice.
8.  Test the category-selector-to-carousel dependency headlessly.
9.  Port the same behavior contracts to Android Compose.

See the companion [StreamFlix SDUI + CoreMVI / Reflex-Inspired
Architecture](StreamFlix_Sdui_CoreMVI_Architecture_Updated.md) for the
detailed screen schema, category/carousel example, component registry,
cross-platform layout, and SDUI roadmap.

------------------------------------------------------------------------

# Part XII --- Final architecture principles

## 47. Non-negotiable principles

1.  **One-way flow:** all state changes are caused by actions processed
    by the reducer.
2.  **Pure reducer:** no networking, persistence, or UI framework calls
    inside reducers.
3.  **Separate translation and execution:** IntentHandler translates;
    EffectHandler executes.
4.  **Generic core, specific features:** CoreMVI provides mechanics, not
    business rules.
5.  **Repository owns data access:** caching and remote/local data
    policy do not belong in views.
6.  **Lightweight State:** hold what the feature needs to represent and
    render, not arbitrary duplicated data.
7.  **Explicit concurrency:** task lifetime, cancellation, and state
    isolation are designed, not accidental.
8.  **Headless capability:** core behavior runs without a UI.
9.  **Observable and introspectable:** state and supported actions have
    stable, safe representations.
10. **Controlled interception:** cross-cutting policies cannot bypass
    the reducer.
11. **Tree composition:** ownership, action forwarding, and lifecycle
    are explicit.
12. **Native presentation:** SwiftUI and Compose render; they do not own
    feature business logic.
13. **Share contracts and tests:** cross-platform parity does not
    require identical implementation code.
14. **Versioned SDUI:** server-provided definitions are validated,
    allowlisted, and safely downgraded.

## 48. One-sentence summary

**CoreMVI is a reusable, headless feature runtime that translates
intents into actions, applies deterministic reducer transitions,
executes effects through injected services, and exposes
observable/introspectable state---while CoreSDUI describes and binds
server-defined native screens to that runtime.**

------------------------------------------------------------------------

## References

-   [Shopify Engineering --- Reflex](https://shopify.engineering/reflex)
-   Companion document: [StreamFlix SDUI + CoreMVI / Reflex-Inspired
    Architecture](StreamFlix_Sdui_CoreMVI_Architecture_Updated.md)

*End of master architecture document.*
