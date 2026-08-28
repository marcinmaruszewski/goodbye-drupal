# NestJS, Fastify, and GraphQL integration boundaries

Status: research note  
Checked: 2026-08-29  
Question: Which combinations are officially supported, and what constraints must later architecture decisions respect?

## Executive findings

- A Nest application using the Fastify HTTP adapter has two documented, first-party GraphQL choices: `ApolloDriver` from `@nestjs/apollo` and `MercuriusDriver` from `@nestjs/mercurius`.
- `ApolloDriver` explicitly supports Nest's Express and Fastify adapters. With Fastify it requires `@as-integrations/fastify`. `MercuriusDriver` is Fastify-only and rejects other Nest HTTP adapters.
- Both drivers support Nest's code-first and schema-first approaches. Schema ownership is therefore independent of the driver decision.
- As published when this note was checked, the shared compatibility envelope for keeping both drivers viable is Nest 12, Fastify 5, and GraphQL 16. GraphQL 17 is accepted by the Apollo line but not by the current Nest Mercurius line.
- HTTP authentication can use Nest guards with a GraphQL-aware execution-context adapter. WebSocket authentication is a separate lifecycle and must be configured explicitly.
- GraphQL subscriptions can distribute server-side domain events, but GraphQL specifies neither their wire transport nor delivery guarantees. They do not by themselves solve collaborative editing, ordering, conflict resolution, presence, reconnection, or state resynchronization.

No final driver, schema approach, authentication mechanism, or realtime topology is selected here.

## 1. Supported integration combinations

### Documented facts

Nest's [GraphQL quick start](https://docs.nestjs.com/graphql/quick-start) documents two official integrations and installation combinations:

| Nest HTTP platform | GraphQL integration | Support boundary | Additional runtime packages |
| --- | --- | --- | --- |
| Fastify | `ApolloDriver` | Officially supported | `@nestjs/apollo`, `@apollo/server`, `@as-integrations/fastify`, `graphql` |
| Fastify | `MercuriusDriver` | Officially supported and Fastify-native | `@nestjs/mercurius`, `mercurius`, `graphql` |
| Express | `ApolloDriver` | Officially supported, but outside this repository's fixed Fastify constraint | Express integration package |
| Express | `MercuriusDriver` | Not supported | The driver throws for any adapter whose type is not `fastify` |

The implementation confirms the boundary. The current [`ApolloDriver`](https://github.com/nestjs/graphql/blob/v14.0.0/packages/apollo/lib/drivers/apollo.driver.ts) dispatches to either an Express or Fastify registration path. Its Fastify path uses `@as-integrations/fastify`. The current [`MercuriusDriver`](https://github.com/nestjs/graphql/blob/v14.0.0/packages/mercurius/lib/drivers/mercurius.driver.ts) registers Mercurius as a Fastify plugin and rejects non-Fastify adapters.

Nest also permits a [custom GraphQL driver](https://docs.nestjs.com/graphql/quick-start), but that moves integration ownership and compatibility testing into the application.

Both official packages export regular, federation, and gateway drivers: Apollo exports `ApolloDriver`, `ApolloFederationDriver`, and `ApolloGatewayDriver`; Mercurius exports the corresponding `Mercurius*` variants. Nest's [federation guide](https://docs.nestjs.com/graphql/federation) treats these as distributed-schema topologies, not as prerequisites for a normal GraphQL server.

When Fastify is the Nest HTTP provider, Express-oriented recipes and middleware are not portable. Nest's [Fastify guidance](https://docs.nestjs.com/techniques/performance) explicitly requires Fastify equivalents. Fastify plugins also have their own supported Fastify version ranges, as described by Fastify's [plugin documentation](https://fastify.dev/docs/latest/Reference/Plugins/).

The embedded IDE also differs. The Nest quick start says that Mercurius does not ship the old GraphQL Playground integration and should use GraphiQL. Nest has deprecated the Apollo Playground path as well and directs current applications toward GraphiQL.

### Inference for this repository

- The fixed choice of Fastify does not force Mercurius; both regular drivers remain valid candidates.
- “Apollo versus Mercurius” is a server-runtime and ecosystem decision, not a decision between GraphQL and Fastify.
- Federation and gateway drivers should remain outside the initial modular-monolith decision. Selecting either would introduce a distributed topology, not merely change a library.
- Adapter-neutral business modules should not depend on Fastify request/reply types or on Apollo/Mercurius hooks. Those types belong at the transport boundary.
- Performance should be measured using representative resolvers and authorization paths. The presence of Fastify alone does not establish which GraphQL driver will be faster for this application.

## 2. Current package compatibility envelope

### Documented facts

The npm registry identified [`@nestjs/core` 12.0.1](https://www.npmjs.com/package/@nestjs/core/v/12.0.1) and the Nest GraphQL packages 14.0.0 as current stable releases when this note was checked. The important peer ranges published for that line are visible in the registry metadata for [`@nestjs/apollo`](https://www.npmjs.com/package/@nestjs/apollo/v/14.0.0), [`@nestjs/mercurius`](https://www.npmjs.com/package/@nestjs/mercurius/v/14.0.0), and [`@nestjs/graphql`](https://www.npmjs.com/package/@nestjs/graphql/v/14.0.0):

| Package line | Relevant peer constraints |
| --- | --- |
| `@nestjs/graphql` 14 | Nest 12; GraphQL `^16.11.0` or `^17.0.0` |
| `@nestjs/apollo` 14 | Nest 12; Apollo Server 5; GraphQL 16 or 17; `@as-integrations/fastify` 2 or 3 |
| `@nestjs/mercurius` 14 | Nest 12; Fastify `^5.2.1`; Mercurius `^16.0.1`; GraphQL 16 |
| `@as-integrations/fastify` 3 | Fastify `^5.3.0`; Apollo Server 4 or 5 |

Consequently, GraphQL 16 is the present intersection if both official Fastify drivers must stay installable. GraphQL 17 is not in the current `@nestjs/mercurius` peer range. Fastify 5 requires Node 20 or newer according to its [v5 migration guide](https://fastify.dev/docs/latest/Guides/Migration-Guide-V5/). Nest 12 has a more detailed runtime and CLI split: its [migration guide](https://docs.nestjs.com/migration-guide) says an application requires Node 20.19+ (or 22.12+ on the 22 line), while current CLI schematics require newer Node releases.

Nest 12 packages are ESM, although the Nest migration guide states that a CommonJS application can still consume them on supported Node versions. The same guide records a current GraphQL breaking change: `@nestjs/graphql` 14 removed `subscriptions-transport-ws`; Apollo subscriptions now use `graphql-ws`.

### Inference for this repository

- Package majors must be selected as a compatible family and locked together; “latest” independently for each package is not a reproducible policy.
- If driver choice is intentionally deferred, GraphQL 16 is the safe shared boundary today. This is a compatibility observation, not a recommendation to preserve driver-swappability indefinitely.
- The development-container and GitHub Actions Node version must satisfy the Nest CLI, not merely the lower application-runtime requirement.
- A future move to GraphQL 17 would need a fresh Mercurius compatibility check and cannot currently be assumed.

## 3. Driver and schema choices are separate axes

### Documented facts

Nest supports both schema construction approaches with either regular driver:

- **Code first:** TypeScript classes and Nest decorators generate the GraphQL schema. `autoSchemaFile` may write an SDL artifact or keep the schema in memory. Nest can also generate SDL without running the application through `GraphQLSchemaBuilderModule`; see [Generating SDL](https://docs.nestjs.com/graphql/generating-sdl).
- **Schema first:** SDL files selected by `typePaths` are the source of truth. Nest can generate corresponding TypeScript definitions from their AST. See the [GraphQL quick start](https://docs.nestjs.com/graphql/quick-start).

The optional [Nest GraphQL CLI plugin](https://docs.nestjs.com/graphql/cli-plugin) reduces code-first decorator boilerplate, but exists because runtime TypeScript metadata cannot express all required schema information. It changes the compilation path and has explicit test-tooling considerations.

In either approach the GraphQL schema can be obtained from `GraphQLSchemaHost` after the application has initialized. Nest documents this specifically for end-to-end testing.

### Inference for this repository

- Code first favors a TypeScript-centered workflow; schema first favors an explicit language-neutral API contract. Neither choice inherently provides typed client operations for Next.js; client code generation remains a separate concern.
- Whichever approach is chosen, CI should materialize and validate a deterministic SDL artifact. That makes schema review and breaking-change checks possible without tying the decision to one driver.
- Domain types should not simply be reused as persistence records, GraphQL inputs, and GraphQL outputs. Those boundaries evolve for different reasons even when code first is used.

## 4. Authentication and authorization implications

### Documented facts

Nest guards, interceptors, filters, pipes, and custom decorators work with GraphQL, but GraphQL resolvers receive a different execution context. Nest's [GraphQL features guide](https://docs.nestjs.com/graphql/other-features) requires converting `ExecutionContext` with `GqlExecutionContext`. The [Passport recipe](https://docs.nestjs.com/recipes/passport) demonstrates a GraphQL auth guard overriding `getRequest()` to return `GqlExecutionContext.create(context).getContext().req`.

Both regular driver implementations add the inbound request as `context.req` by default and preserve it when extending a custom context. This gives HTTP queries and mutations a common Nest-facing request location despite different underlying server integrations.

Nest does not run guards/interceptors/filters for `@ResolveField()` methods by default. `fieldResolverEnhancers` can enable them, but the GraphQL features guide warns about the cost when a field resolver executes many times.

WebSocket subscriptions have a separate connection lifecycle:

- For Apollo, Nest's [subscriptions guide](https://docs.nestjs.com/graphql/subscriptions) enables `graphql-ws` explicitly and performs connection authentication in `onConnect`, using `connectionParams` and the connection context's `extra` field.
- For Mercurius, `subscription` may be `true` or an object. Its official [options reference](https://github.com/mercurius-js/mercurius/blob/master/docs/api/options.md) exposes `verifyClient`, `onConnect`, a subscription-specific `context`, disconnect hooks, and emitter/pubsub configuration.

### Inference for this repository

- The OIDC provider decision is mostly orthogonal to the GraphQL driver: a bearer access token can be validated before domain services are called. Provider SDKs that assume Express middleware are not automatically Fastify-compatible and must be checked separately.
- Authentication should establish a driver-neutral principal; workspace membership and object authorization should be enforced in application/domain services, not only in transport hooks.
- An HTTP guard does not constitute a WebSocket connection policy. Subscription authentication, expired-token behavior, revocation, reconnect, and context refresh need explicit tests and decisions.
- Authorization must also scope event topics and payloads. A client must not receive an event first and rely on UI filtering afterward.
- Enabling field-resolver guards globally is not a free replacement for service-layer authorization because of both coverage defaults and N+1-style execution cost.

## 5. Testing implications

### Documented facts

Nest's [testing guide](https://docs.nestjs.com/fundamentals/testing) supports isolated tests and `TestingModule` tests independently of the HTTP platform. For a Fastify end-to-end application, the documented setup creates a `NestFastifyApplication` with `FastifyAdapter`, calls `app.init()`, waits for the underlying Fastify instance's `ready()`, and sends requests with `app.inject()`.

Fastify's own [testing guide](https://fastify.dev/docs/latest/Guides/Testing/) explains that `inject()` performs fake HTTP injection using `light-my-request` and boots registered plugins before testing. Nest also exposes the initialized GraphQL schema through `GraphQLSchemaHost`, allowing GraphQL operations to execute without an HTTP listener.

Mercurius publishes an official [`mercurius-integration-testing`](https://github.com/mercurius-js/mercurius-integration-testing) utility that covers queries, mutations, and subscriptions. Its subscription client establishes a real connection and accepts initialization payloads, headers, and cookies.

### Inference for this repository

A proportionate test stack has four distinct seams:

1. **Domain and application tests:** call services/use cases directly; no GraphQL driver or Fastify types.
2. **Resolver/module tests:** use `TestingModule` and test argument mapping, Nest guards, and dependency wiring.
3. **Schema contract tests:** obtain or generate SDL and validate intended public types and operations. Direct schema execution is useful but does not prove Fastify routing, headers, cookies, OIDC middleware, or WebSocket handshakes.
4. **Transport end-to-end tests:** POST operations through Fastify `inject()` with real auth headers. Subscription tests additionally require a listening socket/client and must exercise connection authentication, unauthorized rejection, reconnect, unsubscribe, and shutdown.

If the eventual deployment can run multiple API instances, a later integration test must publish on one instance and receive on a connection owned by another. A single-process subscription test cannot demonstrate a production fan-out design.

## 6. Boundaries for subscriptions and live collaboration

### Documented facts

The September 2025 [GraphQL specification](https://spec.graphql.org/September2025/#sec-Subscription) defines a subscription as a stateful response stream driven by a source event stream. Each subscription operation has exactly one root field. The specification deliberately does not choose serialization, transport, acknowledgements, buffering, retries, or other quality-of-service behavior. It also notes that subscription state and execution may need separate services at scale.

For Apollo on Nest 14, the supported WebSocket implementation is `graphql-ws`; the removed `subscriptions-transport-ws` protocol is wire-incompatible. The [GraphQL over WebSocket protocol](https://github.com/enisdenjo/graphql-ws/blob/master/PROTOCOL.md) has an explicit connection-initialization handshake and multiplexes operations by ID.

Mercurius enables subscriptions through its Fastify plugin. Its [WebSocket documentation](https://github.com/mercurius-js/mercurius/blob/master/docs/graphql-over-websocket.md) defaults to the modern `graphql-transport-ws` subprotocol and also supports the deprecated legacy subprotocol for compatibility. Its [subscription guide](https://github.com/mercurius-js/mercurius/blob/master/docs/subscriptions.md) documents an external emitter such as Redis and custom pubsub implementations. The Nest subscriptions guide likewise warns that Apollo's example in-memory `PubSub` is not suitable for production and points to externally backed implementations.

### Inference for this repository

- GraphQL subscriptions are a candidate **delivery interface for domain change notifications**, not a complete shared-canvas synchronization model.
- Client edits remain mutations (or a separate write protocol); subscriptions are server-to-client result streams. The bidirectional WebSocket transport does not change GraphQL's operation semantics.
- Multiple Cloud Run instances will require shared event distribution or another realtime service. An in-memory emitter only reaches connections owned by the same process.
- Reconnect must include an authoritative state refresh or replay strategy because GraphQL promises no acknowledgement, replay, ordering, or exactly-once delivery.
- Collaborative editing still needs explicit decisions about operation identity, optimistic concurrency, conflict handling (for example OT or CRDT), persistence, presence, cursor traffic, and authorization per workspace/document.
- High-frequency ephemeral signals such as cursors may eventually justify a different realtime channel from durable domain events. The application model should not expose driver-specific PubSub APIs so that this can evolve independently.

## 7. Decision boundaries handed to later tickets

This research makes the following questions ready to decide without answering them here:

1. Choose `ApolloDriver` or `MercuriusDriver` for the first regular GraphQL server.
2. Choose code first or schema first as the schema ownership model.
3. Define a single driver-neutral request principal and workspace-authorization boundary.
4. Define the initial test pyramid, including which transport guarantees must be covered in end-to-end tests.
5. Keep subscriptions out of the first increment or select a minimal notification use case; separately decide the later collaborative-state protocol.
6. Pin a compatible Node/Nest/Fastify/GraphQL package family and automate dependency compatibility checks.

## Primary sources

- [NestJS GraphQL quick start](https://docs.nestjs.com/graphql/quick-start)
- [NestJS GraphQL subscriptions](https://docs.nestjs.com/graphql/subscriptions)
- [NestJS GraphQL other features](https://docs.nestjs.com/graphql/other-features)
- [NestJS testing](https://docs.nestjs.com/fundamentals/testing)
- [NestJS Fastify guidance](https://docs.nestjs.com/techniques/performance)
- [NestJS v12 migration guide](https://docs.nestjs.com/migration-guide)
- [NestJS GraphQL v14 source](https://github.com/nestjs/graphql/tree/v14.0.0)
- [Fastify v5 testing](https://fastify.dev/docs/latest/Guides/Testing/)
- [Fastify plugin reference](https://fastify.dev/docs/latest/Reference/Plugins/)
- [Mercurius documentation and source](https://github.com/mercurius-js/mercurius)
- [GraphQL specification, September 2025](https://spec.graphql.org/September2025/)
- [`graphql-ws` protocol](https://github.com/enisdenjo/graphql-ws/blob/master/PROTOCOL.md)
