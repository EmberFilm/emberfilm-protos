# emberfilm-protos

The canonical protobuf contracts for the EmberFilm microservices. Every `.proto` in the system
lives here and nowhere else; producers and consumers depend on this package by tag and never copy
a schema.

## Products

One library product per service, each generating messages, a client, and a server with the
`GRPCProtobufGenerator` plugin:

| Product | Package | Services |
| --- | --- | --- |
| `AuthenticationProtos` | `emberfilm.authentication.v1` | `AuthenticationService`, `RegistrationService` |
| `BillingProtos` | `emberfilm.billing.v1` | `PackageService`, `BillingService`, `ReconciliationService` |
| `EntitlementsProtos` | `emberfilm.entitlements.v1` | `EntitlementService` |
| `NewsletterProtos` | `emberfilm.newsletter.v1` | `SubscriberService` |
| `UsersProtos` | `emberfilm.users.v1` | `UserService` |

Files are nested by organization, service, and version —
`Sources/<Service>Protos/emberfilm/<service>/v1/<service>.proto` — and the `package` declaration
matches the path. Generated Swift types are prefixed `Emberfilm_<Service>_V1_`.

## Requirements

Swift 6.3, Swift 6 language mode, macOS 15 or later.

## Installation

```swift
.package(url: "https://github.com/EmberFilm/emberfilm-protos.git", from: "0.15.0")
```

```swift
.target(
    name: "MyServiceGRPC",
    dependencies: [
        .product(name: "UsersProtos", package: "emberfilm-protos"),
        .product(name: "GRPCCore", package: "grpc-swift-2"),
        .product(name: "GRPCProtobuf", package: "grpc-swift-protobuf"),
        .product(name: "SwiftProtobuf", package: "swift-protobuf"),
    ]
)
```

A service links only the contracts it speaks: its own to serve, and each upstream's to call.

## Conventions

- `v1` stays `v1`; additions are compatible and land as minor tags. A breaking change is a `v2`
  package beside the old one, never an edit to a released message.
- Removed field numbers and names are reserved.
- Identifiers are lowercase UUID strings on the wire; the owning service's database generates
  them, so a create request never carries the entity's id. A retryable create carries an
  `idempotency_key` instead.
- Dates are `google.protobuf.Timestamp` fields named as nouns: `creation_date`, `update_date`,
  `expiration_date`.
- Enums start with an `_UNSPECIFIED` zero value that consumers refuse rather than default.
- Where a service has two audiences, it has two gRPC services in the same file — the caller-facing
  one and a worker-facing one such as `RegistrationService` or `ReconciliationService`.

## Releasing

```sh
swift build
git tag 0.16.0 && git push --tags
```

A change here is invisible to every service until it is tagged and the service's `Package.swift`
requirement and `Package.resolved` move to it.
