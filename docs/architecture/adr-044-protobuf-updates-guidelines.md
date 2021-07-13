# ADR 044: Guidelines for Updating Protobuf Definitions

## Changelog

- 28.06.2021: Initial Draft

## Status

Draft

## Abstract

This ADR provides guidelines and recommended practices when updating Protobuf definitions. These guidelines are targeted at module developers.

## Context

The SDK maintains a set of [Protobuf definitions](https://github.com/cosmos/cosmos-sdk/tree/master/proto/cosmos). When making changes to these Protobuf definitions, the SDK currently only follows [Buf's](https://docs.buf.build/) recommendations. We noticed however that Buf's recommendations might still result in breaking changes in the SDK in some cases. For example:

- Adding fields to `Msg`s. Adding fields is a not a Protobuf spec-breaking operation. However, when adding new fields to `Msg`s, the unknown field rejection will throw an error when sending the new `Msg` to an older node.
- Marking fields as `reserved`. Protobuf proposes the `reserved` keyword for removing fields without the need to bump the package version. However, by doing so, client backwards compatibility is broken as Protobuf doesn't generate anything for `reserved` fields. See [#9446](https://github.com/cosmos/cosmos-sdk/issues/9446) for more details on this issue.

Moreover, module developers often face other questions around Protobuf definitions such as "Can I rename a field?" or "Can I deprecate a field?" This ADR aims to answer all these questions by providing clear guidelines on allowed updates for Protobuf definitions.

## Decision

We decide to keep [Buf's](https://docs.buf.build/) recommendations with the following exceptions:

- `UNARY_RPC`: the SDK currently does not support streaming RPCs.
- `COMMENT_FIELD`: the SDK allows fields with no comments.
- `SERVICE_SUFFIX`: we use the `Query` and `Msg` service naming convention, which doesn't use the `-Service` suffix.
- `PACKAGE_VERSION_SUFFIX`: some packages, such as `cosmos.crypto.ed25519`, don't use a version suffix.
- `RPC_REQUEST_STANDARD_NAME`: Requests for the `Msg` service don't have the `-Request` suffix to keep backwards compatibility.

On top of Buf's recommendations we add the following guidelines that are specific to the SDK.

### Updating Protobuf Definition Withouth Bumping Version

#### 1. `Msg`s SHALL NOT have new fields.

When processing `Msg`s, the SDK's antehandlers are strict and don't allow unknown fields in `Msg`s. This is checked by the unknown field rejection in the [`codec/unknownproto` package](https://github.com/cosmos/cosmos-sdk/blob/master/codec/unknownproto).

Now imagine a v0.43 node accepting a `MsgExample` transaction, and in v0.44 the chain developer decides to add a field to `MsgExample`. A client developer, which only manipulates Protobuf definitions, would see that `MsgExample` has a new field, and will populate it. However, sending the new `MsgExample` to an old v0.43 node would cause the v0.43 node to reject the `Msg` because of the unknown field. The expectation that the same Protobuf version can be used across multiple node versions should be guaranteed.

For this reason, module developers SHALL NOT add new fields to existing `Msg`s.

It is worth mentioning that this does not limit adding fields to a `Msg`, but also to all nested structs and `Any`s inside a `Msg`.

On the other hand, module developers MAY add new fields to Protobuf definitions related to the `Query` service, as the unknown field rejection does not apply to queries.

#### 2. Fields MAY be marked as `deprecated`, and nodes MAY implement a protocol-breaking change for handling these fields.

Protobuf supports the [`deprecated` field option](https://developers.google.com/protocol-buffers/docs/proto#options), and this option MAY be used on any field, including `Msg` fields. If a node handles a Protobuf message with a non-empty deprecated field, the node MAY change its behavior upon processing it, even in a protocol-breaking way. When possible, the node SHOULD handle backwards compatibility.

As an example, the SDK v0.42 to v0.43 update contained two Protobuf-breaking changes, listed below. Instead of bumping the package versions from `v1beta1` to `v1`, the SDK team decided to follow this guideline, by reverting the breaking changes, marking those changes as deprecated, and modifying the node implementation when processing messages with deprecated fields. More specifically:

- The SDK recently removed support for [time-based software upgrades](https://github.com/cosmos/cosmos-sdk/pull/8849). As such, the `time` field has been marked as deprecated in `cosmos.upgrade.v1beta1.Plan`. Moreover, the node will reject any proposal containing an upgrade Plan whose `time` field is non-empty.
- The SDK now supports [governance split votes](./adr-037-gov-split-vote.md). When querying for votes, the returned `cosmos.gov.v1beta1.Vote` message has its `option` field (used for 1 vote option) deprecated in favor of its `options` field (allowing multiple vote options). Whenever possible, the SDK still populates the deprecated `option` field, that is, if and only if the `len(options) == 1` and `options[0].Weight == 1.0`.

#### 3. Fields SHALL NOT be renamed.

Whereas the official Protobuf recommendations do not prohibit renaming fields, as it does not break the Protobuf binary representation, the SDK explicitly forbids renaming fields in Protobuf structs. The main reason for this choice is to avoid introducing breaking changes for clients, which often rely on hard-coded fields from generated types. Moreover, renaming fields will lead to client-breaking JSON representations of Protobuf definitions, used in REST endpoints and in the CLI.

### Bumping Protobuf Package Version

TODO, needs architecture review. Some topics:

- Bumping versions frequency
- When bumping versions, should the SDK support both versions?
  - i.e. v1beta1 -> v1, should we have two folders in the SDK, and handlers for both versions?
- mention ADR-023 Protobuf naming

## Consequences

> This section describes the resulting context, after applying the decision. All consequences should be listed here, not just the "positive" ones. A particular decision may have positive, negative, and neutral consequences, but all of them affect the team and project in the future.

### Backwards Compatibility

> All ADRs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The ADR must explain how the author proposes to deal with these incompatibilities. ADR submissions without a sufficient backwards compatibility treatise may be rejected outright.

### Positive

{positive consequences}

### Negative

{negative consequences}

### Neutral

{neutral consequences}

## Further Discussions

This ADR is still in the DRAFT stage, and the "Bumping Protobuf Package Version" will be filled in once we make a decision on how to correctly bump Protobuf package versions.

## Test Cases [optional]

Test cases for an implementation are mandatory for ADRs that are affecting consensus changes. Other ADRs can choose to include links to test cases if applicable.

## References

- [#9445](https://github.com/cosmos/cosmos-sdk/issues/9445) Release proto definitions v1
- [#9446](https://github.com/cosmos/cosmos-sdk/issues/9446) Address v1beta1 proto breaking changes