# How BeamUp works

People ask what BeamUp actually does after a coding agent finishes writing an
app. The short answer is that BeamUp gives the app somewhere governed to exist.
The longer answer involves WebAssembly, capability manifests, stable identity,
encrypted runner connections, content-addressed bundles, and a carefully timed
change of authority when a laptop-hosted app moves online.

Let’s take the longer route. We’ll build the system from the bottom up, which is
roughly how we built it, minus the parts where a clean diagram was preceded by
several impolite diagrams.

> **Open-core status**
>
> BeamUp is currently at Stage 0. The
> [public repository](https://github.com/beamup-run/beamup) contains
> documentation and project governance, but no implementation source yet.
> BeamUp plans to publish the artifact contracts, local trust boundary, and
> portability tooling in stages. The hosted control plane will remain
> proprietary.

With enough time, this article should give you what you need to build something
BeamUp-shaped yourself. BeamUp plans to publish the bundle and manifest
contracts, manifest tools, CLI, local runner, starter, bounded runner protocol
types, and portable-state formats in stages. The hosted control plane is not
part of that plan, and the public protocol is not a self-hosting promise. If
your response to that is “fine, I’ll write a control plane,” you may be exactly
the kind of person who enjoys reading the rest of this.

## The app is no longer the expensive part

Suppose you ask a coding agent to make a launch-review tool. Ten minutes later
you have a form, a list of decisions, and maybe even tests. It runs on your
laptop.

What do you need before somebody else can use it?

You need a build, a process that stays alive, a URL, TLS, authentication,
permissions, a database, secrets, logs, updates, rollback, and some answer to
the question “who owns this thing?” If the app reads a file on your laptop, you
also need to decide whether that file may be uploaded, mounted, proxied, copied,
or never moved at all.

The traditional answer is to create a miniature production system for every
app.

<!-- Artwork brief: An app at left followed by an absurdly tall stack of
repository, container, registry, IAM, load balancer, TLS, database, secrets,
monitoring, and on-call boxes. Greyscale with one BeamUp yellow app tile. -->

*Figure 1. One modest app with its traditional industrial-size infrastructure
stack.*

That approach works. It is also why useful internal software ends up living in
terminal tabs, shared as a screen recording, or rebuilt later as a “real
project.” The code was cheap. The operating identity around the code was not.

A second answer is to tunnel a port from the developer’s laptop. That is faster,
and it is useful, but the tunnel mostly answers a networking question. It does
not tell you what the generated code can access, give the app a durable identity,
separate owners from viewers, preserve state across versions, or explain what
happens when the laptop closes.

BeamUp starts with a different primitive.

## An application is not a process

In BeamUp, an application has a stable identity. A version is one immutable
piece of code assigned to that application. A runner is the place currently
executing it. These are three separate things.

That separation sounds fussy until the first update.

If identity and process are the same thing, replacing the process tends to
replace its URL, state, credentials, or deployment record too. If identity and
placement are the same thing, moving from a laptop to managed compute creates a
second app. Links break, permissions drift, and someone eventually asks which
copy is authoritative.

BeamUp keeps the application ID, slug, URL, owner, viewer access, state
namespace, capability decisions, and history independent of both version and
placement.

```text
Application
  stable ID
  stable URL
  owner and viewers
  capability history
  state namespace
  placement epoch

      points to one active version
      points to one active runner
```

<!-- Artwork brief: Three horizontal layers. Stable app identity at top,
version cards moving beneath it, and local/managed/company runners at bottom.
Arrows change while the top app card remains fixed. -->

*Figure 2. Versions and runners can change without asking the app to become a
different app.*

We sometimes describe this as immutable code plus mutable governed state. The
code can be replaced frequently. The state, ownership, permissions, and history
should not be accidentally discarded with it.

## The bundle is the executable contract

The first concrete thing BeamUp creates is a bundle. It contains the compiled
WebAssembly component, static browser assets, a normalized capability manifest,
runtime metadata, and build provenance. Entries have canonical names, canonical
ordering, and explicit size limits. The complete bundle has a cryptographic
content hash.

This is not just reproducible-build enthusiasm, although we do enjoy that sort
of thing. The hash is the name of the exact reviewed bytes.

In the private alpha, `beamup publish` builds locally with pinned inputs,
normalizes the manifest, and shows a plain-language capability review. A
current app might ask for this:

```text
launch-review requests:
  Allow  read snapshots from review-queue
  Allow  read and write this app's private BeamUp KV
  Deny   outbound network
  Deny   all other files and directories
  Deny   secrets and environment variables
```

The owner approves that contract, not a vague category called “safe.” BeamUp
records the decision against the application, version, manifest hash, and
bundle hash. If a later version asks for more authority, it requires a new
approval. If it asks for the same or less, BeamUp can say exactly why no
authority increased.

The bundle is still only a candidate. A content hash proves which bytes you
have. It does not prove those bytes are allowed to run on a particular runner.
Installation authority also binds the owner, application, version, runner,
runner kind, assignment, placement epoch, and capability-decision receipt.

The runner then distrusts everyone one more time, which is usually a healthy
instinct for a computer that is about to execute code. It independently parses
the bundle, recomputes the hashes, reproduces the canonical manifest, checks the
runtime manifest against the approved host policy, writes into private staging,
rereads and reverifies the result, and only then atomically installs an
immutable candidate.

A partial upload is not an application. A correct bundle delivered for the
wrong assignment is not an application. A previously valid approval attached
to different bytes is not an application. These all fail closed.

## WebAssembly is the floor, not the whole security model

BeamUp’s default runtime is Spin on Wasmtime. The compiled app runs as a
WebAssembly component on local and managed runners.

WebAssembly gives us a portable executable format and strong memory isolation.
It does not automatically know that a launch-review app may read one data source
but not your SSH directory. That part belongs to the BeamUp host.

Generated code receives no ambient filesystem, network, secret, database, or
cross-application state access. The manifest names capabilities. The host
constructs only those capabilities. The guest cannot grant itself another one,
and generated code never gets to implement its own sandbox. That would be like
letting a houseguest write the lease after finding the spare key.

The current alpha is deliberately narrow. Its example data capability is one
owner-approved, read-only JSON review queue. The runner opens one regular file
through a retained directory boundary, rejects symlinks and aliases, bounds it
to 64 KiB and 100 records, validates a closed schema, and publishes an immutable
snapshot to the guest. The guest receives selected fields and content
revisions. It never receives the path, a directory, a file handle, or a general
filesystem API.

The same pattern extends beyond files. A future connector to a warehouse,
GitHub repository, or Slack channel is not an environment variable containing
a powerful credential. It is a host operation scoped to a named resource and a
small set of actions. Credentials can rotate outside the app. Access can be
revoked outside the app. Denials can be recorded outside the app.

There are limits to say out loud here. WebAssembly does not protect a runner
from its own privileged operator. It does not make denial of service
impossible. Managed placement therefore adds operating-system isolation,
workload identities, quotas, bounded restart behavior, protected state,
patching, and incident controls around the Wasm guest.

## The control plane is centralized. Execution is not.

We now have a reviewed bundle and a runner willing to execute it. How does a
browser find that runner?

BeamUp has a centralized control plane. It stores application identity, owner
identity, viewer access, versions, capability decisions, runner registrations,
placement assignments, promotion receipts, entitlements, and bounded event
history.

The application’s payload state does not live in the ordinary control-plane
database. Local app state belongs to the local runner. Managed app state belongs
to the managed state plane. The control plane stores the facts needed to decide
which placement is authoritative, not a convenient miscellaneous drawer for
customer data.

Each local runner has a cryptographic iroh endpoint identity. Its secret is
created on the runner and stored in the operating system’s protected credential
store. It is never silently regenerated. The runner proves possession when it
registers, and both sides authenticate the exact endpoint when opening a
connection. Managed runners add an assignment-bound workload identity for the
exact application and placement epoch.

For a local runner, iroh can connect through NAT and changing network locations
without asking the developer to open an inbound port. It connects directly when
possible and can use an encrypted relay path when necessary. A relay can forward
the runner traffic, but it is not a BeamUp application principal and cannot
decrypt the gateway-to-runner connection.

<!-- Artwork brief: Browser at left, trusted HTTPS gateway in center, control
plane above it, and three runners at right. Metadata arrows go between gateway
and control plane. App request arrows go from browser through gateway to exactly
one selected runner. A faint relay sits under the local-runner path. -->

*Figure 3. Centralized identity and policy select one distributed execution
location.*

This is not a peer-to-peer browser connection. The browser uses ordinary HTTPS
to the BeamUp gateway. The gateway terminates TLS and may observe the HTTP
request and response. That is an explicit trust boundary, not a detail hidden
behind the word “local.”

Why keep the gateway in the path? It gives the application one normal,
placement-independent URL. It also gives BeamUp one place to authenticate the
viewer, select the current placement, apply request limits, and enforce a
browser response policy that the guest cannot weaken.

So the system is centralized where people want consistency and distributed
where placement matters. Control is centralized. Execution and state follow
the selected runner.

## Owners, runners, and viewers are different principals

We skipped over a question. Who is allowed to register a runner, publish a
version, or approve a move?

The current owner login uses GitHub’s device flow as the external identity
proof. BeamUp asks for no GitHub scopes. The provider token exists only for the
bounded exchange. BeamUp then issues its own short-lived access credential and
rotating refresh session. The CLI stores those in macOS Keychain or Linux
Secret Service. The control plane stores one-way hashes and lifecycle metadata,
not raw reusable credentials.

The owner credential can authorize a runner registration, but it is not the
runner credential. The runner proves possession of its own endpoint key and is
registered to the exact owner and BeamUp origin. A public endpoint ID is not
enough to preclaim a runner. Losing a local runner key does not cause BeamUp to
quietly invent a replacement with the old identity.

The owner’s browser is another principal. BeamUp does not copy the CLI bearer
into browser storage. The authenticated CLI creates a short-lived,
exact-origin handoff. The owner types a one-time code in the browser, compares
a four-word phrase with the CLI, and confirms the match. Only then does the
browser receive a separate, revocable owner session.

A link viewer is separate again. Viewer authority can open one app. It cannot
register runners, inspect owner events, approve capabilities, publish versions,
or move placement.

<!-- Artwork brief: Four identity cards labeled external identity, BeamUp
owner, runner, and viewer. Narrow arrows show which proof creates which
authority. No arrow from viewer to owner. -->

*Figure 4. Identity proof, owner authority, runner authority, and viewer access
do not share a credential.*

## One browser request, all the way down

Let’s follow a request to `https://beamup.run/apps/launch-review`.

First, the viewer proves access. BeamUp currently supports a revocable link
capability. The raw capability is exchanged for an opaque, application-bound
viewer session. The control plane stores only a domain-separated hash of the
raw capability. Rotating or revoking the link invalidates the corresponding
generation without turning it into an owner credential.

Owner and viewer authority are different types in the system. A viewer cannot
become an owner by changing a header, reusing a cookie, or finding an owner
route. This sounds obvious. It also sounds like the kind of obvious thing that
should be represented in code rather than optimism.

The gateway resolves the viewer session, application, active version, exact
runner assignment, runner identity, runner kind, and placement epoch in one
transactional view. An offline runner, revoked link, stale assignment, disabled
runner, or inactive version does not produce a half-valid routing decision.

The gateway then creates one bounded `beamup-http/1` request. Browser-supplied
cookies, authorization, forwarding headers, and `x-beamup-*` identity claims
are removed. The current protocol bounds each request and response body to
1 MiB and gives the complete gateway exchange a five-second deadline. There is
no automatic retry of an ambiguous POST.

The gateway opens one authenticated bidirectional stream to the registered
runner. The runner checks the gateway identity, then independently checks the
application, version, assignment, runner kind, runner ID, and placement epoch
against its own trusted assignment. Only then can the request reach the guest.

The response makes the return trip, but the guest does not get final say over
the browser.

BeamUp discards guest-supplied cookies and security headers and installs a fixed
content security policy, sandbox policy, referrer policy, framing policy, and
permissions policy. Cross-origin redirects fail. Same-app redirects are
rewritten beneath the stable app route. Refresh, popups, workers, frames,
downloads, third-party resources, and top-level navigation are denied by the
current source-backed policy.

For the alpha’s reviewed source-backed interface, BeamUp goes one step further.
Browser asset routes must return the exact reviewed content type and canonical
asset hash. This prevents generated server code from replacing a reviewed,
text-only interface with a new browser program that uses a disclosure path the
owner never approved.

<!-- Artwork brief: Seven numbered steps around a loop. Viewer session, route
snapshot, bounded protocol, authenticated stream, assignment check, Wasm guest,
gateway response policy. -->

*Figure 5. A request is authorized, pinned, bounded, authenticated, checked
again, executed, and filtered on the way back.*

The gateway is trusted and authorized viewers can receive approved source
fields. Browser extensions, an authorized person copying data, and browser
defects remain disclosure paths. “Runs on your laptop” does not mean “data can
never leave your laptop.” It means the source binding and state begin there,
and movement or disclosure happens through named, reviewable boundaries.

## State belongs to the app

Each BeamUp application has one private logical key-value namespace. The
namespace is derived from the stable application ID, not the version ID.

Locally, the current implementation uses SQLite behind a closed KV interface.
The guest can operate on bounded keys and values in its own namespace. It
cannot choose another app ID, open the database, run SQL, or provide a
filesystem path. Quotas are enforced in the storage layer, including for guest
writes.

This gives updates an important property. Version two can replace version one
without acquiring a fresh empty database merely because its code hash changed.
State continues with the app.

It also creates a harder question. How do you move the app without copying a
live SQLite file and hoping no one wrote to it halfway through?

BeamUp exports logical state, not database pages. The runner acquires a bounded
writer freeze, reads canonical key-value entries in sorted order, encodes them
in a versioned format, and computes a content hash. A separate receipt binds
the app ID, schema, hash, size, and counts without putting keys or values into
control-plane metadata.

Host data follows the same rule. The current validated review-queue snapshot can
be represented as canonical, path-free bytes. The live local file binding
cannot. One is portable. The other is location-bound.

This distinction matters enough to have names.

- **Portable** means the exact reviewed item can move through a canonical,
  verified format.
- **Replaceable** means the owner may choose a different approved binding at
  the destination.
- **Location-bound** means it cannot follow the app automatically.

Code portability is not environment portability. A WebAssembly component may
run anywhere while the USB device, office database, or file it depends on
remains stubbornly located where physics left it.

## Updates are a pointer change, after readiness

When you publish a new version, BeamUp does not overwrite the old executable in
place.

The new bundle is built, reviewed, transferred, verified, installed as an
immutable candidate, and started beside the current version. The runner reports
readiness for that exact assignment. Only then does the control plane
compare-and-swap the application’s active version from the expected old
assignment to the new one.

Requests admitted before the switch carry the complete old assignment and can
finish on the old process within the bounded drain window. Requests admitted
after the switch carry the new assignment. There is no request made of half of
version one and half of version two, an innovation we are happy to leave to
other platforms.

The immediately previous verified version is retained for rollback. A rollback
performs the same kind of reviewed pointer change in the other direction. It
does not rewind state. If version two wrote `approved=true`, activating version
one does not make that write disappear. Code rollback and data restoration are
different operations, and pretending otherwise tends to produce exciting
incident reports.

<!-- Artwork brief: v1 active, v2 staged and ready, a single transaction flips
the pointer, v1 drains and becomes previous. A second small panel reverses the
pointer while the state cylinder remains unchanged. -->

*Figure 6. Versions move around the state namespace. The state namespace does
not move around the versions.*

## Keep online is a change of authority

A local app is available while its runner is reachable. Eventually somebody
closes the laptop because laptops are portable computers, not unusually thin
datacenters.

BeamUp’s **Keep online** operation moves the same application to a managed
runner. It is not a second deploy.

Before anything moves, BeamUp presents the owner with the exact target
placement, price, bundle and capability identity, portable state, approved
source revision, and location-bound behavior that will stop. The live local
file refresh in the current alpha is one such behavior. The immutable approved
snapshot can move. The path and future refreshes cannot.

After approval, the promotion sequence is:

```text
local active
  -> freeze local writes
  -> export canonical state and source snapshots
  -> verify exact hashes
  -> install the reviewed bundle on the managed runner
  -> import state into the managed state plane
  -> start the managed candidate
  -> verify readiness and denial parity
  -> atomically advance the placement epoch
  -> stop the stale local assignment
```

The important step is the placement compare-and-swap.

Until that transaction commits, local epoch N is the only writer. A managed
candidate can be installed, imported, started, and ready without receiving
viewer traffic or write authority. If something fails before the switch,
BeamUp must prove that the target was cleaned and authenticate an exact resume
before it reports a completed failure. If it cannot prove both, the operation
stays recovery-pending. It does not switch authority and it does not guess.

At the switch, the application’s active assignment changes to managed epoch
N+1. The URL, owner, viewers, version, capabilities, state identity, and history
do not change. The old local epoch becomes stale and its writes fail closed.

If final cleanup is interrupted after the switch, managed remains
authoritative. BeamUp retries forward finalization. It does not guess that local
should become the writer again. This asymmetry is intentional. On either side
of the transaction, there is one answer to “where may the next write commit?”

<!-- Artwork brief: Two runner islands with a state package traveling between
them. A large gate labeled placement epoch sits in the middle. Before the gate,
local glows. After the gate, managed glows. Never both. -->

*Figure 7. Data can be copied before cutover. Authority cannot.*

After a successful move, the laptop may shut down. The gateway resolves the
same URL to the managed runner. Existing viewer sessions continue. The local
file is no longer being proxied in the background, because “always online as
long as your closed laptop is secretly online” would be a fairly bad product
promise.

## What BeamUp plans to open and what BeamUp operates

BeamUp uses excellent open-source foundations. Spin supplies the application
model. Wasmtime supplies the WebAssembly runtime. Iroh supplies authenticated
endpoint connectivity, direct paths, relay fallback, and content-addressed
transport primitives.

BeamUp is currently at Stage 0. The public repository contains documentation
and project governance, but no BeamUp implementation source.

The planned public release stages are:

- **Stage 1, artifact contracts.** Publish the manifest and bundle
  specifications, canonical parsing and hashing, bounded protocol types,
  TypeScript starter, conformance vectors, and focused compatibility tests.
- **Stage 2, local runtime.** Publish the CLI and local-only runner after
  managed adapters are separated and distributed binaries can be tied to the
  exact public source commit.
- **Stage 3, portability and ecosystem.** Publish portable KV and approved
  source snapshot codecs, verification tooling, and additional integration
  surfaces as their compatibility contracts stabilize.

When released, these components will let you inspect the local trust boundary,
reproduce bundle identities, verify capability enforcement, export portable
state, and run the local components without treating a proprietary binary as a
small metal box full of good intentions.

The hosted BeamUp control plane will remain a commercial service. BeamUp will
operate the stable application namespace, owner and viewer authority, policy
and approval graph, runner inventory, placement and promotion coordination,
managed runner operations, managed state, backups, recovery, entitlements,
billing integration, event history, managed connectors, organization-specific
policy, and the organizational application record.

That boundary is deliberate. Our advantage is not a secret compression trick
inside the Wasm runner. It is the governed lifecycle that keeps identity,
authority, state, placement, and history coherent while apps and infrastructure
change. Publishing the artifact and local-enforcement layers makes that
lifecycle easier to trust without giving away the complete service that makes
it useful across an organization.

See the
[public open-core boundary](https://github.com/beamup-run/beamup/blob/main/OPEN_CORE.md)
for the release stages, licensing, and source-release requirements.

## Start with one app

The BeamUp model does not require a company-wide platform migration.

In the private alpha, start with an app your agent already wrote. Build it
locally. Review what it can access. Publish it from the machine already running
it. Give one person a revocable URL. Update it without changing that URL. Keep
it online only when the app becomes worth operating continuously.

Later, the same model can place runners inside a company network near private
systems. The control plane still knows the owner, application, version,
capabilities, and history. The host still enforces explicit authority. The
browser still uses the same application identity. Placement changes because the
data and availability requirements changed, not because the app was forced to
start its life over.

Coding agents made the executable part of software cheap enough to throw away
and rebuild. BeamUp is for the parts you should not have to throw away with it.
