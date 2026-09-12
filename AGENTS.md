# AGENTS.md

`emberfilm-protos` is the contract package of the EmberFilm system: every `.proto`, one target and
product per service, and nothing else. The swift-microservices skills are the source of truth for
how contracts are shaped, generated, and consumed; this file drives them and records EmberFilm's
choices and what is specific to this package. When the two disagree, it is either a deviation
listed below or a bug in one of them; say which rather than papering over it.

## Skills

Install once: `/plugin marketplace add swift-microservices/skills` then
`/plugin install swift-microservices@swift-microservices` (Codex: symlink `skills/<skill>` into
`~/.codex/skills`). Invoke by task:

| Task | Skill |
| --- | --- |
| A new service, RPC, message, or field; generation; a tag | `building-swift-services` (its grpc-and-protos reference: *Canonical proto package*, *Contract design*) |
| Which audience an RPC belongs to, a boundary or consistency change | `designing-swift-systems` |
| A review on request, read-only | `reviewing-swift-services` |

## EmberFilm's choices

The skills leave these open; EmberFilm decided them once, and every repository follows the same answers.

- **Shape:** microservices over gRPC (grpc-swift-2), fronted by one HTTP gateway (`emberfilm-api`, Hummingbird). Contracts live in `emberfilm-protos`, split by audience; the org layer is `emberfilm-core` (`EmberFilmAuthentication`, `EmberFilmPersistence`, `EmberFilmTesting`) over the swift-microservices packages. Both are consumed by tagged URL; a change there is invisible until it is tagged and `Package.resolved` moves.
- **Data:** one Postgres 18 instance per environment with a database and its own owner per service; roles `<service>_service`, `<service>_internal`, and `<service>_worker` where a worker exists. Row-level security is tenant isolation on `app.caller_user_id` in users, entitlements, and billing; newsletter and authentication have no tenant tables. Identifiers are `uuidv7()`; migrations run at boot behind `--migrate-database`.
- **Identity:** `UserIdentity` is the token (`sub`, `role`, `iss`, `iat`, `exp`), EdDSA-signed by the authentication service alone; every other process verifies with the public key and forwards the original token with `BearerPropagationInterceptor`. A process is its certificate, `spiffe://emberfilm/<process>`, over mTLS from the stack's own CA. Roles are `user` and `admin`; authorization is decided in use cases only.
- **Delivery:** develop deploys to staging and main to production, two environments of one Dokploy project on one host; images are published per commit to ghcr.io with a short-SHA tag and a branch tag; logs ship in-process to Loki; the local stack is the suite's `compose.yml`.

## This repository

One target per service at `Sources/<Service>Protos/emberfilm/<service>/v1/<service>.proto`, with
the same `grpc-swift-proto-generator-config.json` (clients, servers, messages; `public` access) and
an empty `<Service>Protos.swift` marker. Generated types are prefixed `Emberfilm_<Service>_V1_`.
No Swift beyond the markers, no helpers, no messages shared across services.

| Product | Services, by audience |
| --- | --- |
| `AuthenticationProtos` | `AuthenticationPublicService` (no credential: register, verify, sign in, refresh, logout, reset), `AuthenticationService` (a signed-in user's own passkeys, auth codes, password) |
| `BillingProtos` | `PackagePublicService` (the catalogue), `PackageService` (administrators write it), `BillingPublicService` (the Stripe webhook, authenticated by its signature), `BillingService` (checkout, portal, and purchases for the caller) |
| `EntitlementsProtos` | `EntitlementService` (a user's own account; the last three RPCs are an administrator's), `EntitlementInternalService` (the billing process, by certificate) |
| `NewsletterProtos` | `SubscriberPublicService`, `SubscriberService` |
| `UsersProtos` | `UserService`, `UserInternalService` (the authentication process, by certificate) |

**The audience is the service, never the method.** The comment above each `service` says who may
call it, and the producer applies its interceptor per service. An RPC whose caller changes moves
to another service; an internal service exists for another process's certificate and is never
opened to a person. `GetUserByEmail` stays internal because it would confirm which addresses have
accounts.

**Releasing is the delivery.** A source edit here changes nothing anywhere until it is tagged.
Tag additive changes as a minor release so `from:` consumers pick them up on their next resolve,
and bump each consumer's floor deliberately so it names what that consumer imports. The six
consumers currently pin `from: "0.23.0"`.

**Create requests carry no id; a retried one carries an `idempotency_key`.** `CreateUserRequest`
is the model. Never add an `id` field to make a caller's retry convenient; the owning database
generates it and the response returns it.

**`UserRole` on the wire is the users service's enum, not the token's.** This package's `UserRole`
is `USER_ROLE_USER` and `USER_ROLE_ADMIN`, what the users service persists; the token's `UserRole`
in `emberfilm-core` is an open string. They are mapped at the transport boundary and
`USER_ROLE_UNSPECIFIED` is refused. Adding a role touches `emberfilm-core` first, then here, then
the authentication service that mints it.

**The billing contract models the monolith's package.** A package is a Stripe price, a title, what
it grants, and how many devices it may enroll; products and offers are the gateway's views of it.
Keep that shape unless the billing service changes it first.

## Commands

```sh
swift build                         # generates and compiles every product
swift build --product UsersProtos
git tag <version> && git push --tags
```

There are no tests. A contract change is verified by building the producer and every consumer
against it: point a consumer at this working copy with
`swift package edit emberfilm-protos --path ../emberfilm-protos`, then unedit, tag, and bump.

## Deviations from the skills

- **`Package.resolved` is tracked.** The delivering skill has a library never commit it; this
  package does, from before the rule. Untrack it with the next contract change, not as a drive-by.
- **Releases are bare tags.** The skills release a library as a GitHub Release from semver labels;
  this package has no workflows and no labels yet, so `git tag` is the release until it does.

## Conventions

- Swift 6 language mode, tools 6.3, macOS 15+.
- `package emberfilm.<service>.v1;` matches the directory. Snake-case field names, `_id` suffixes
  for identifiers, noun-based date names, `google.protobuf.Empty` for RPCs that return nothing.
- Every RPC name is a business capability, not a table operation.
- The comment above each service names its audience; a change that invalidates it rewrites it.
