---
name: rfc_process
started: 2024-07-04
pr: pact-foundation/roadmap#0000
---
## Summary

The Request for Comments (RFC) process is a way to propose and discuss changes to the Pact specification, ecosystem, and community. The RFC process ensures that changes are well-considered, documented, and communicated to all stakeholders. This RFC proposes the process itself, including how to submit an RFC, how to review and discuss it, and how to accept or reject it.

## Motivation

While the evolution of Pact and its ecosystem is open, the process for proposing and discussing changes is not well-defined. By introducing an RFC process, we can ensure that changes are well-considered, documented, and communicated to all stakeholders. This will help to ensure that changes are made in a transparent and collaborative way, and that the community has a voice in the direction of the Pact ecosystem.

## Guide-level explanation

### Scope of RFCs

The Pact ecosystem is broad and includes, broadly speaking, the following areas:

- The Pact specification itself ([pact-foundation/pact-specification](https://github.com/pact-foundation/pact-specification))
- The Pact broker ([pact-foundation/pact-broker](pact-foundation/pact_broker))
- The Pact CLI ([pact-foundation/pact-ruby-cli](https://github.com/pact-foundation/pact-ruby-cli))
- The Pact FFI ([pact-foundation/pact-reference](https://github.com/pact-foundation/pact-reference/tree/master/rust/pact_ffi)).

This RFC process is intended to cover any change which will impact the broader Pact ecosystem.

While specific changes to the Pact implementation in various languages are generally out of scope[^1], we do wish to consider how changes in the areas above will impact the implementation in various languages. Furthermore, we may wish to introduce or formalize common conventions or guidelines that are applied across the various languages to ensure consistency, interoperability, and familiarity for users.

[^1]: Changes to the Pact implementation in various languages are out of scope because they are managed by the maintainers of the individual language implementations (which may even have their own RFC process).

### The Process

The RFC process is intended to remain flexible while still ensuring that changes are well-considered, documented, and communicated to all stakeholders. The process is as follows:

1. Fork the Pact Foundation Roadmap repository: [`https://github.com/pact-foundation/roadmap`](https://github.com/pact-foundation/roadmap)
2. Copy the `rfc/0000-template.md` file to `rfc/0000-my-feature.md`. Use a simple, descriptive name for the file, but do not assign an RFC number yet.
3. Fill in the RFC.
4. Submit a pull request. The pull request is where the RFC will be discussed, and reviewed.
5. Incorporate feedback, and iterate on the RFC.

Eventually, a member of the Pact Foundation will either reject the RFC by closing the PR, or accept it by merging the PR. If the RFC is accepted, the following steps will also be taken:

1. Creating a tracking issue in the Pact Foundation Roadmap repository for the implementation of the RFC.
2. Assign an RFC number based on the PR number, and update the filename and metadata (including link to the tracking issue to implementation).
3. Where applicable, other issues may be created in other repositories, with the tracking issue linking to these issues.

## Drawbacks

The RFC process may slow down the pace of change in the Pact ecosystem, since it introduces a formal process for proposing and discussing changes. This may be seen as a drawback by some, especially those who are used to a more informal process.

## Rationale and alternatives

The current process for changes has been ad-hoc and mostly informal. While this has allowed for rapid iteration and experimentation, it has made it difficult to track the reasoning behind changes as they are spread across various issues, pull requests, and (archived) Slack conversations. This has made it difficult for new contributors to understand the rationale behind changes, and for existing contributors to keep track of the changes.

By introducing an RFC process, we can ensure that changes are well-considered, documented, and communicated to all stakeholders. This will help to ensure that changes are made in a transparent and collaborative way, and that the community has a voice in the direction of the Pact ecosystem.

## Unresolved questions

- Does the proposed RFC process balance ease with which changes can be proposed and discussed with the need for a formal process?
- Does the proposed RFC process cover all the areas that it needs to, or are there any areas that are missing?
- What should happen if an RFC is rejected? Is it sufficient to simply close the PR?
