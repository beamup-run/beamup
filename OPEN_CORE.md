# BeamUp open-core boundary

BeamUp intends to publish the components that define what runs on a developer's
computer, what authority an application requests, and how portable artifacts
can be inspected and verified.

BeamUp will operate the hosted coordination, governance, and managed-continuity
service.

## Planned public surface

### Artifact contracts

- manifest schema and capability model
- bundle layout and content identity
- canonical parsing, normalization, and hashing
- compatibility fixtures and golden vectors

These contracts make a BeamUp artifact independently inspectable.

### Local trust boundary

- CLI
- local runner
- host capability enforcement
- protected local runner identity
- local application state adapter
- approved local-source boundary
- TypeScript starter and host adapter

These components run on the developer's machine. Publishing them is intended to
make local execution and local-data access reviewable.

### Portability contracts

- portable KV snapshot format
- approved-source snapshot format
- export and verification codecs
- placement compatibility reporting

Portable formats should be usable without treating a proprietary binary as the
only way to recover application data.

### Interoperability surface

- bounded runner protocol types
- protocol versioning rules
- conformance vectors
- fail-closed decoding and policy tests

The public protocol surface is not a promise that BeamUp's hosted control plane
will be self-hostable.

## Hosted commercial surface

BeamUp plans to keep the following service implementation private:

- owner and viewer authentication services
- stable application namespace and routing decisions
- policy and approval graph
- placement and promotion coordination
- managed runner provisioning and workload operations
- managed state, backup, restore, and recovery operations
- entitlements and billing integration
- event history and organizational application inventory
- managed connectors and organization-specific policy

Public client contracts may describe the requests and receipts needed to use
the service. They will not include the service implementation or private
operational systems.

## Release stages

### Stage 0: boundary published

This is the current stage. The repository documents intent and governance. No
BeamUp implementation is published.

### Stage 1: artifact contracts

Publish the manifest and protocol crates, specifications, starter, conformance
vectors, and focused tests.

### Stage 2: local runtime

Publish the CLI and local-only runner after managed adapters are separated,
release packaging is ready, and the distributed binaries are tied to the exact
public source commit.

### Stage 3: portability and ecosystem

Publish snapshot codecs, compatibility tooling, and additional integration
surfaces as their stability and support contracts become clear.

## Release requirements

The first implementation release should include:

- a reviewed public source commit
- a semantic version and signed tag
- locked dependencies and build instructions
- checksums for distributed artifacts
- a software bill of materials
- exact source-to-binary provenance
- a documented vulnerability-reporting path
- a public compatibility and support policy

The stages have no promised dates. BeamUp will publish source when the boundary
and release artifacts are ready to support responsibly.
