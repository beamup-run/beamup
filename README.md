# BeamUp

> **Status: coming soon**

This repository is the future public home of the parts of BeamUp that run on a
developer's computer and define a portable BeamUp application.

The implementation is not published yet. This repository is not currently a
software distribution, a self-hosted BeamUp control plane, or evidence of a
generally available product.

[BeamUp](https://beamup.run) is the governed runtime for agent-built software.
It gives an application a persistent identity, explicit capabilities, governed
access, and a path between supported runners.

## Why this repository exists now

We want the open-source boundary to be visible before the source arrives.
Developers should be able to see what BeamUp intends to publish, what the hosted
service will continue to operate, and how the boundary will be released.

Opening the repository early also gives future links a stable destination
without pretending that unfinished code is ready for use.

## What we plan to publish here

- BeamUp manifest and bundle specifications
- canonical parsing, normalization, and content hashing
- bounded gateway-runner protocol types and conformance vectors
- the BeamUp CLI and local runner
- the TypeScript starter and host adapter
- portable KV and approved-source snapshot formats
- focused tests for local enforcement and artifact compatibility

The first source release will be made from a reviewed, tagged commit with build
instructions, checksums, and release provenance. No release date is promised
yet.

## What BeamUp will operate

The hosted BeamUp service will continue to operate:

- stable application identity and URLs
- owner and viewer authority
- policy and approval history
- runner inventory and placement coordination
- managed state, backups, and recovery
- entitlements, events, and the organizational application record

That service boundary is the commercial product. Publishing the local runtime
and portable contracts is intended to make the customer-side trust boundary
inspectable without turning this repository into a clone of the hosted
platform.

See [OPEN_CORE.md](OPEN_CORE.md) for the planned boundary and release stages.

## Current phase

The repository currently contains documentation and project governance only.
Implementation pull requests are not being accepted yet.

You can:

- follow this repository for future releases
- open an issue about the documented boundary
- [request private alpha access](https://beamup.run/request-access?source=github)
- read [How BeamUp works](https://beamup.run) as the public technical material
  is published

## License

Code and documentation in this repository are available under either the
[Apache License 2.0](LICENSE-APACHE) or the [MIT License](LICENSE-MIT), at your
option.

The licenses do not grant rights to the BeamUp name or marks. See
[TRADEMARKS.md](TRADEMARKS.md).

## Security

Do not report security issues or include secrets in public issues. See
[SECURITY.md](SECURITY.md) for the private reporting path.
