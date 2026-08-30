# CLAUDE.md

Guidance for Claude Code and other agents working in this repository.

## The skill

This repository is built to the conventions of the
[swift-grpc-microservices-skill](https://github.com/the-braveknight/swift-grpc-microservices-skill).
Install it (`git clone` into `~/.claude/skills/swift-grpc-microservices-skill`) and read its
`SKILL.md` and `references/grpc-and-protos.md` (*Canonical proto package*, *Contract design*)
before changing anything here. The skill is the source of truth; this file records only what is
specific to this repository.

## What this is

`emberfilm-protos` is a contract package, not a service. It holds every `.proto` in the EmberFilm
system, one target and product per service, and nothing else — no Swift beyond the empty marker
file each target needs, no helpers, no shared messages across services. It is consumed by tag
from `https://github.com/EmberFilm/emberfilm-protos.git`; every service and the gateway pin it
with `from:`.

Sibling repositories live next to this directory under `emberfilm-microservices/`. The identity
package's `ServiceIdentity` product is the one library outside the services that links a contract
here (`AuthenticationProtos`, for `IssueServiceToken`).

## Layout

```
Sources/
  AuthenticationProtos/  emberfilm/authentication/v1/authentication.proto
  BillingProtos/         emberfilm/billing/v1/billing.proto
  EntitlementsProtos/    emberfilm/entitlements/v1/entitlements.proto
  NewsletterProtos/      emberfilm/newsletter/v1/newsletter.proto
  UsersProtos/           emberfilm/users/v1/users.proto
```

Each target carries the same `grpc-swift-proto-generator-config.json` (clients, servers,
messages; `public` access) and a `<Service>Protos.swift` marker.

## Commands

```sh
swift build                         # generates and compiles every product
swift build --product UsersProtos
git tag <version> && git push --tags
```

There are no tests. A contract change is verified by building the producer and every consumer
against it — point a consumer at this working copy with `swift package edit emberfilm-protos
--path ../emberfilm-protos` (see the skill's *The shared identity package* for the edit-mode
pitfalls), then unedit, tag, and bump.

## Specialization

**Releasing is the delivery.** A source edit here changes nothing anywhere until it is tagged.
Tag additive changes as a minor release so `from:` consumers pick them up on their next resolve;
bump the requirement in each consumer's `Package.swift` deliberately so the pinned floor names
what the code imports. The current consumers pin different floors (`0.2.0` … `0.15.0`); that is
fine as long as each floor contains what that consumer uses.

**Worker-facing services are separate services.** `RegistrationService` and
`ReconciliationService` exist so a Temporal worker reaches its own service's data over gRPC as a
`service` instead of holding a database. Add worker RPCs there, never to the caller-facing
service, and give a new service with a worker its own `<X>Service` beside the user-facing one.

**Create requests carry no id and, where retried, an `idempotency_key`.** `CreateUserRequest` is
the model. Do not add an `id` field to a create request to make a caller's retry convenient.

**Session-issuing RPCs have their own response types.** `IssueServiceTokenResponse` has no
refresh token because a process gets none; a session type with an empty field would make the
contract lie.

**`Role` on the wire is the users service's enum, not the token's.** `emberfilm-identity`
defines the token's `Role` (`user`, `admin`, `service`); this package's `Role` is what the
users service persists (`ROLE_USER`, `ROLE_ADMIN`). They are mapped at the transport
boundary and `USER_ROLE_UNSPECIFIED` is refused. Adding a role touches identity first, then here.

**The billing contract models the monolith's package.** Products and offers are the gateway's
views of it; `GetPackageByPriceID` is what a Stripe webhook resolves against. Keep that shape
unless the billing service changes it first.

## Deviations from the skill

None. The package has no `.err` file because it has no executable.

## Conventions

- Swift 6 language mode, tools 6.3, macOS 15+.
- `package emberfilm.<service>.v1;` matches the directory. Snake-case field names; `_id` suffixes
  for identifiers; noun-based date names; `google.protobuf.Empty` for RPCs that return nothing.
- Every RPC name is a business capability, not a table operation.
