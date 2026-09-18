# What control plane already exists in the TOON ecosystem?

> **Research note, not a decision record.** Answers
> [#4](https://github.com/toon-protocol/omarchy/issues/4), a child of wayfinder map
> [#1](https://github.com/toon-protocol/omarchy/issues/1). Facts and quotes against
> primary sources only — repository source at HEAD on 2026-09-18, read through `gh api`, plus
> this repository's own tree. Nothing here is a recommendation.

`CONTEXT.md:118` defines the organ this note is about:

> **Controller**:
> Whatever decides the connector's leased routes and peering. Outside the connector by
> definition — the connector never learns, announces, or discovers. The line is about
> **deciding**, not about fetching: a connector told to reach a counterparty will read that
> counterparty's self-description to learn how, exactly as it dials a handler's URL. What it
> never does is choose whom to peer with, or find one it was not pointed at.

`CONTEXT.md:132` and `:137` draw the two verbs either side of that line:

> **Announcing**:
> Pushing facts about yourself into a network unprompted. A connector never does this: deciding
> to participate in a discovery network is the controller's business.
> _Distinct from_: answering, which a connector does do.

> **Answering**:
> Telling whoever asks what your own configuration already says — your identity, and what a
> route of yours costs. Mechanism, not policy: it decides nothing, and reaches nobody who did
> not ask. […] _Avoid_: discovery (for this — a connector answers, it does not discover)

---

## 0. The socket the controller would plug into is finished

Before asking who occupies the ground, it is worth recording that the connector's side is
**complete and was completed recently**. `docs/protocol/operator-spec.md:316-322`:

> **Reads** — bearer token: `GET /peers` · `/routes` · `/routes/leased` · `/routes/peers` ·
> `/channels` · `/claims` · `/rates` · `/identity` · `/audit-log` · `/metrics`
>
> **Writes** — RFC 9421 HTTP Message Signature from a key on an operator allowlist, with RFC
> 9530 Content-Digest binding the signature to the body: `POST /packets` · `POST|DELETE /peers`
> · `POST /routes/leased` · `POST|DELETE /routes/peers` · `POST /channels` ·
> `/channels/:id/fund` · …

The leased-route write is `crates/connector-operator/src/lib.rs:444`, and its doc comment names
the missing party outright:

> `POST /routes/leased`: a controller outside this connector pushes a route to a peer with a
> time limit.

Its body (`crates/connector-operator/src/lib.rs:438-443`) is three fields:

```rust
struct CreateLeasedRouteRequest {
    prefix: String,
    peer_id: String,
    ttl_seconds: i64,
}
```

ADR 0049's `## Update (issue #1160)` closes the last hole in that surface:

> Consequences above says _"The operator surface must be able to express a cap, and today it
> cannot. `Connector::upsert_runtime_peer` takes an `id` and nothing else."_ That was true when
> it was written and is not true now: `POST /peers` carries `max_packet_amount` beside `fee`,
> and the durable runtime peering persists both.

**Nothing in the TOON organisation calls any of it.** `grep -rn "routes/leased"` across this
repository returns only the implementation, its tests, `README.md:1042,1084` and
`docs/protocol/operator-spec.md`. An org-wide code search for `"routes/leased"` returned no
results outside this repo. None of `rig`, `gateway`, `tuzzy`, `provider` or `toon-client`
contains a `POST /routes/leased`, a `POST /peers`, an RFC 9421 `Signature-Input`, or an
`[operator] write_keys` caller (see each section below).

---

## 1. `rig` — "control plane" means the git event graph, not a connector

`rig`'s own root `README.md`:

> The **Rig** is TOON Protocol's git-to-TOON write path and its decentralized control-plane
> frontend. It interprets Nostr events (NIP-34 git is its first surface — not a GitHub clone)
> and gives them a browsable, forge-like UI, while the core library builds the git objects and
> NIP-34 events that make repos publishable to TOON.

`rig`'s own `CONTEXT.md`, opening line:

> Git with a TOON remote: repo state lives in NIP-34 events on a relay, objects live on Arweave,
> writes are paid, reads are free.

`packages/rig-web/README.md:3`:

> The Rig — a browser-only SPA that renders TOON Protocol events (NIP-34 git vocabulary first:
> repos, refs, issues, PRs) from a relay over WebSocket, with git objects fetched from Arweave
> gateways. **No backend, no accounts, no servers.**

**What it controls: repository state.** The "plane" is the Nostr event graph — NIP-34 kinds
`30617`, `30618`, `1617`, `1621`, `1630`–`1633`, `1111`; NIP-C1 CI kinds declared at
`packages/rig/src/ci/nip-c1-events.ts:33-47`; NIP-90 store/factory job kinds `5094`–`5097`.
Nothing administers a connector.

**No directory of nodes.** One URL is the entire network configuration.
`packages/rig/src/cli/entry.ts:1-9`:

> `rig entry` — name the TOON connector rig pays.
> One URL is the whole network configuration since 4.0: a connector describes itself on
> `GET /ilp` (addresses, prices, chains, sealing key — connector ADR 0050), so there is nothing
> else to point rig at.

`packages/rig/README.md:11-15`:

> **One node URL is the whole network configuration** — `rig` embeds its own payment client
> (`@toon-protocol/client` 3.x) built from your seed phrase and pays the TOON connector you name
> with `rig entry <url>`. The node describes itself on `GET /ilp`; there is nothing to discover

`rig`'s `CONTEXT.md` explicitly disowns the one directory that does exist in the ecosystem:

> Terms for leases, workloads, providers, listings, and the Provider Directory are defined in
> TOON Network's `CONTEXT.md` (toon-protocol/TOON_Network).

**No leased-route push.** A search of the whole repo (lockfile included) for `POST /routes`,
`POST /peers`, `POST /channels`, `write_keys`, `RFC 9421`, `Signature-Input` and `httpbis`
returned **zero hits**. The only connector interaction is _paying_ it —
`packages/rig/src/standalone/connector-publisher.ts:1-12`:

> The paid write path on `@toon-protocol/client` 2.x — a {@link Publisher} that pays a TOON
> connector per request. Everything a paid command needs comes from one connector URL: the
> client reads the node's own `GET /ilp` (addresses, prices, settlement chains, sealing key —
> connector ADR 0050), opens or adopts a payment channel, and pays each request with a signed
> claim. There is no relay to discover peers on (kind:10032 was removed by ADR 0046) and no
> topology to negotiate

`packages/rig/src/routes.ts` is a false friend: it is the `toon-clientd` daemon's `/git/*`
control API, not a connector's. `rig channel open|close|settle` are on-chain wallet operations
(`packages/rig/src/cli/channel.ts:12-14`), not operator writes.

**Deployment:** the GitHub Pages workflow that both READMEs still cite was deleted —
`a851ad77`, 2026-08-09, _"ci: drop the GitHub Pages deploy — rig-web publishes to Arweave now
(#75)"_. The live deployment is an Arweave manifest pinned at
`packages/rig/src/rig-pointer.ts:38-46` (`manifestTx: 'Iy8sHYVnoevSpwDC2lqUoIg8BWdes9NmMbgmPd81UpU'`);
Pages survives only as a no-JS fallback link (`:129-130`), whose "tracks main" comment is now
false. The last `github-pages` deployment was 2026-08-07.

**Packages:** exactly two — `@toon-protocol/rig` 4.7.0 (published, `bin: rig`) and
`@toon-protocol/rig-web` 0.2.46 (`"private": true`, never published).

---

## 2. `TOON_Network` (private) — a directory exists, and it is for compute providers

**Access confirmed.** `TOON_Network` and `provider` are private; I could read both through
`gh api`. Everything quoted here is read from them at HEAD, 2026-09-18.

`TOON_Network/CONTEXT.md` defines a Directory block, verbatim:

> ### Directory
>
> **Provider Directory**:
> The set of published provider profiles, listings and liveness that tenants search to find a
> provider.
> _Avoid_: Registry, marketplace
>
> **Provider Profile**:
> A provider's published statement of who it is and how it is reached and paid, independent of
> anything it sells.
> _Avoid_: Offer, provider ad
>
> **Listing**:
> One sellable tier published by a provider: the resources a lease gets, the capabilities it
> grants, and its prices.
>
> **Liveness**:
> A provider's short-lived published statement that it is up, which expires unless renewed.
>
> **Relay Set**:
> The relays a provider publishes its profile, listings and liveness to.

and a **Registries** block for `Image Registry`, `Blob Record` and `Template`.

`docs/spec/toon-network-v1.md` §4 makes it concrete. Kinds: `10432` Provider Profile
(replaceable), `30432` Listing (addressable), `10433` Liveness (replaceable). §4 opening:

> A provider publishes the events in this section to **every relay in its Relay Set**. Each
> event MUST carry the tag `["L","toon.network"]` so directory queries can select them.

The Provider Profile's content table (§4.1) **carries the connector's facts**:

| Field                | Meaning                                                                                        |
| -------------------- | ---------------------------------------------------------------------------------------------- |
| `ilp_address`        | The provider's ILP address, chosen by the provider, e.g. `g.acme`                              |
| `connector_url`      | The connector's self-description URL, e.g. `https://c.acme.example/ilp`. A location hint only. |
| `connector_seal_key` | The connector's sealing public key (§3)                                                        |
| `relays`             | The Relay Set                                                                                  |
| `settlement`         | `{ "chain": …, "token": …, "decimals": 6 }`                                                    |
| `hidden`             | Hidden Provider declaration (§10)                                                              |
| `host`               | Public host tenants connect to. MUST be absent when `hidden` is true.                          |

**This is the "different object" ADR 0046 anticipated, and it closes the hole ND-14 worries
about.** ADR 0046:

> **A third party's announce is a different object, and this record does not define one.** If a
> controller publishes facts about a node it did not sign, the event is that controller's claim
> about the node rather than the node's claim about itself — a materially different security
> property from what kind:10032 has meant. Anyone building that is defining a new thing and
> should say so.

`TOON_Network/docs/adr/0011-a-profile-pins-its-connectors-sealing-key.md` says so, in full:

> A provider profile publishes its connector's sealing public key alongside the connector's URL.
> A tenant seals requests only to that key, and refuses to proceed if the self-description at the
> URL reports a different key.
>
> Spawn requests carry environment variables that may hold secrets. The profile is signed by the
> provider's Nostr key, so the sealing key inside it cannot be forged, while a URL can be spoofed
> or intercepted. Pinning the key also covers the case where a request reaches the provider
> through hops and the tenant never contacts the provider's URL at all.

That is the inverse of `ND-14`/ADR 0054's rule and it is safe for the same reason: the key is
signed by the **provider's own** Nostr key and is cross-checked against the node's own
self-description, so no hop can substitute one. It is `CONTEXT.md:132`'s _announcing_ done by
the operator, from a second config file, as `CONTEXT.md:125` ("Some verbs belong to the operator
and not to the process: announcing is one.") requires.

**It is built.** `toon-protocol/provider` (private, Rust) has `src/directory.rs` (34 KB), whose
header reads:

> The Directory port: the second of the provider's two I/O ports (the first is `ComputeBackend`).
> Everything the provider says about itself in public — its Provider Profile, its Listings, its
> Liveness, an Eviction Notice — leaves through `publish` […]
> Why a port at all: a relay write on the TOON Network is PAID (ADR 0007), so publishing is a
> payment, a network round trip and a per-relay outcome

`src/nostr/directory_events.rs:43-53` is the wire struct, with the connector fields carried
verbatim:

```rust
pub struct ProfileContent {
    pub ilp_address: String,
    /// The connector's self-description URL. A LOCATION HINT ONLY: the key
    /// below is what a tenant seals to, and it refuses if the URL reports a
    /// different one (ADR 0011).
    pub connector_url: String,
    /// The connector's sealing public key, hex.
    pub connector_seal_key: String,
    …
}
```

They come from the **provider app's own TOML**, copied by hand — `provider.example.toml:43-46`:

```toml
# the key verbatim from `GET <connector_url>/identity` so the comparison is …
connector_url = "https://c.acme.example/ilp"
connector_seal_key = "0x04325b…"
```

and it is exercised end to end. `toon-protocol/infra`'s `sandbox/scripts/smoke-directory.mjs:1-3`:

> Provider Directory smoke test (TOON_Network Milestone 1, ticket #6): the provider is
> DISCOVERABLE.

— proving the Profile is on the relay carrying "the connector's SEALING KEY (byte-for-byte what
the connector's own `/ilp` identity reports — ADR 0011)", that Listings are **relay-filterable**
(`#l isolation:shared-kernel`, `#l arch:amd64`), that Liveness expires and replaces, and that the
writes were **paid**.

**What it does not define.** I searched the whole 88 KB spec for `controller`, `10032`,
`IlpPeerInfo`, `leased route` and `discovery`: **zero hits for every one of them**. There is no
controller role, no connector directory, no route leasing and no peering decision anywhere in
TOON Network. Its directory finds a **provider selling compute**, not a **connector to peer
with**. A tenant still reaches the provider's connector by the URL the Profile hands it, which
is the same "ask direct, pay through" the connector already assumes.

Two further facts from the spec worth recording here:

- §11 open item 4, verbatim: _"**Runtime route writes:** the connector has none for terminated
  routes, so every listing change restarts it."_ That is correct against this tree —
  `CreateLeasedRouteRequest` (above) takes `peer_id`, never a `handler_url`, so only a
  **forwarded** route can be written at runtime.
- §4.3 Liveness is a paid, expiring, replaceable event with `["expiration", now + 5 ×
liveness_cadence_s]` (ADR 0007). That is the ecosystem's only built answer to "is this node
  still up", and it is a provider's answer, not a connector's.

---

## 3. `gateway` — naming and resolution, keyed by a grant; not a directory

`gateway`'s `README.md`:

> A stable HTTPS name for a TOON Network workload, keyed by its **workload id**. […] the
> canonical label is the **lowercase, unpadded base32 (RFC 4648) of the 32-byte workload id** —
> 52 characters, where the 64 hex characters of the same id would not fit a 63-character DNS
> label.

> **It holds no lease, pays for nothing and calls no paid route.** Its whole authority is the
> **Gateway Grant** (spec §3.1.3) — a tenant's signed, published delegation naming one gateway,
> one workload, the workload's HTTP port and its Standby Set, with an expiry.

**It is two resolutions stacked.**

1. **name → workload id**, purely derivational. `README.md`: _"It is derived, never assigned: two
   gateways holding the same grant serve the same label, and a Takeover changes nothing about the
   name."_
2. **workload id → a live `host:port`**, `src/resolve.mjs` header:

> RESOLUTION FOLLOWS THE GRANT AND NOTHING ELSE (spec §12). The grant names the Standby Set; each
> member's Provider Profile says where its connector is; every member is asked for `status` with
> a request this gateway signed and the grant inside it; the member answering `running` with
> `access` is the one running the workload.

**Where the candidate set comes from: Nostr, not configuration.** Four sources, none of them DNS,
on-chain or a provider API:

- the grant itself, kind `30438`, found with one filter — `src/relays.mjs`:
  `return { kinds: [kind], '#p': [gatewayPubkey] };`, commented _"`#p` is the gateway's own key,
  which is why no tenant has to make contact: publishing the grant IS telling the gateway"_;
- each named member's **Provider Profile** (kind `10432`), `src/profiles.mjs`:
  `filters: [{ kinds: [K_PROFILE], authors: [...watched] }]`, whose header says _"a gateway never
  guesses a connector from a host, and never asks a member the grant does not name"_;
- the members' live free `status` answers;
- Takeover events (kind `30433`) on the primary's Relay Set.

Its only config is `GATEWAY_RELAYS`, `GATEWAY_SECRET_KEY`, `GATEWAY_DOMAIN`, TLS,
`TOON_SOCKS_PROXY` and `GATEWAY_DIAL_REWRITE` (`src/config.mjs`: _"Environment only […] there is
no config file to drift from what a deployment actually runs."_).

**Is it discovery by another name? Partly — but it cannot be enumerated or searched.** A host the
gateway did not previously know does arrive, off a relay, without anyone contacting the gateway.
But the query key is a workload id already delegated to that specific gateway by a signed grant.
`src/serve.mjs`: _"A hostname that is none of those is answered 503 by the gateway itself and
NOTHING is dialled: no provider is asked about a workload nobody granted."_ `src/grants.mjs`
refuses the word on purpose:

> Deliberately not called a registry: CONTEXT.md keeps "Registry" for the Image Registry and warns
> it off the Provider Directory, and this is neither. It is the set of grants held, which is the
> words the spec uses.

**The five named files.**

- `src/dial.mjs` — the single chokepoint for every outbound TCP connection and the `.anyone`
  policy. Header: _"Every outbound connection a gateway makes […] goes through ONE seam […] Both
  legs share it on purpose, because both legs can land on a Hidden Provider […] and a gateway that
  got one leg right and the other wrong would leak exactly what hiding is for."_ Exports
  `isAnyoneHost`, `NoProxyError`, `refusalFor`, `connectionOptions`, `secured`, `createDialer`.
- `src/main.mjs` — 40 lines: `readConfig(process.env)` in a try/catch that exits 1,
  `startGateway({ config, log })`, signal handlers.
- `src/profiles.mjs` — reads, verifies and holds kind `10432` Provider Profiles for Standby Set
  members; splits `.anyone` relays into a `hiddenRelays` list; grows its relay set transitively
  from what a Profile names.
- `src/reasons.mjs` — one 503 vocabulary of exactly six codes: `no_grant`, `grant_expired`,
  `not_resolved`, `no_running_member`, `member_unreachable`, `no_proxy`, rendered as the
  `{ "error", "message" }` shape _"every provider route answers a refusal in (spec §5)"_.
- `src/rewrite.mjs` — `GATEWAY_DIAL_REWRITE`, development only, a `host[:port] → host[:port]` map
  applied at the dial seam. _"What it does NOT do: it rewrites no URL, no `Host` header and
  nothing a member or a workload sees."_

**`.anyone`, and no `.onion` at all.** `src/dial.mjs`:

```js
/** Whether a host is an `.anyone` address, which is never dialled directly. */
export const isAnyoneHost = (host) =>
  typeof host === 'string' && /\.anyone\.?$/i.test(host.replace(/^\[|\]$/g, ''));
```

```js
connect(host, port) {
  if (!isAnyoneHost(host)) return undefined;
  if (proxy === undefined) throw new NoProxyError(host, port);
  // `socks` sends a name as a DOMAINNAME (RFC 1928 ATYP 3) and resolves
  // nothing itself; the proxy does, at the far end of the circuit.
  return SocksClient.createConnection({ … });
}
```

with `validateSocks5hUrl` imported from `@toon-protocol/client`. **`onion` has zero occurrences
in the repository** (searched; not found). The connector's own rule (ADR 0070, amended by #1284)
accepts **both** spellings; the gateway accepts `.anyone` only.

**Payment-oblivious, and it says so.** `src/status.mjs`:

> `status` is a FREE route (spec §5), so there is no payment to make and no claim to attach, and
> this gateway holds no channel, no mnemonic and no wallet: it must never call a paid route.

`README.md`: _"It holds no payment channel, no mnemonic and no lease, and it never calls a paid
route — so there is no connector configuration here to get wrong."_ Searched for `payer` —
zero hits. It forms no ILP packet; its own stated limit is that it reaches _"not a member
reachable only through a sealed client edge"_.

**Operator API: untouched.** The only connector-directed traffic is one plain `POST` to
`<connector_url minus /ilp>/status` carrying a signed kind-`4432` Lease Request. No `/channels`,
no `/packets`, no RFC 9421, no bearer token.

---

## 4. `tuzzy` — a buyer, not a control plane

`tuzzy/README.md`, first lines:

> Buys [Anyone Protocol][anyone] circuit credentials over [TOON][toon], in USDC, for a crawler
> that never handles payment.
> It is the **buyer** to [anytoon][anytoon]'s seller. anytoon showed that an operator can sell
> blind-signed credentials for tokens over ILP; this is the other half — a continuous,
> non-speculative, machine-scale customer for them.

> It buys bundles, verifies them under the issuer's epoch key, and pools them. On request it hands
> one out. **It does not present them.** Nothing relay-side enforces credentials yet […] so
> "done" here means _bought, verified, pooled and takeable_.

Six TypeScript source files, a CLI with three verbs (`buy`, `take`, `status`) on a cron timer,
`@toon-protocol/client` ^3.0.0 as its payment client. Created 2026-09-10, six commits, last
pushed 2026-09-12. Its only CI workflow publishes an image and _"runs no test and proves nothing
about the loop"_ (`docs/handoff.md`).

**Nothing to do with directories or a control plane.** Searched for `discovery`, `index`, `10032`
and `onion` — **zero hits each**; `directory` and `registry` hit only filesystem paths and
`ghcr.io`; `search` hits only prose about Wuzzy, the sibling search product.

**Why it shows up in an Anyone Protocol search:** it is _for_ Anyone Protocol by name.
`package.json` `description`: _"Buys Anyone Protocol circuit credentials over TOON, in USDC, for
a crawler that never handles payment."_ Its `src/config.ts` carries the same host rule the
connector has:

```ts
if (!proxy.startsWith('socks5h://')) {
  throw new Error(
    `TUZZY_SOCKS_PROXY must be socks5h:// (the proxy must resolve the name): ${proxy}`
  );
}
```

and its `CONTEXT.md` states the rule this repository's ADR 0070 states:

> ## Hidden-service endpoint
>
> Fixed by anytoon's glossary and not redefined here. The address is a **host**: it is not a
> scheme, not a carriage and not a transport.
> The TOON client learned this the hard way — a `.anyone`/SOCKS5h _transport overlay_ was built
> and then removed, and hidden-service support returned as a host that demands a `socksProxy`.
> Do not reintroduce the overlay reading.

`docs/handoff.md` is candid about its own reach: _"That logic is unit-tested and has never carried
a purchase. No automated check in any of these repositories has tuzzy buying a bundle over a
circuit."_ and _"Presentation is unproven, because it is impossible today […] `take` hands out a
credential. **Nothing can present one.**"_

---

## 5. ADR 0046's fallout list, item by item

ADR 0046's `## Update (issue #1074)`:

> **Downstream consumers were not touched, and this is the sequencing ADR 0046 already recorded
> as a separate operational task.** `toon-client`'s `discovery-subscription.ts`,
> `@toon-protocol/core`'s `parseIlpPeerInfo`, `rig` and the genesis peer seeds all still read
> kind:10032.

Taken one at a time, against HEAD on 2026-09-18:

### `toon-client`'s `discovery-subscription.ts` — **deleted; the file no longer exists**

I walked the full recursive git tree, not just `packages/*/src/`. `toon-client` has exactly one
package, `packages/client`, and `find -iname '*discover*' -o -iname '*subscri*' -o -iname
'*genesis*' -o -iname '*seed*' -o -iname '*bootstrap*'` over the whole tree returns **zero
files**. There is no `DiscoverySubscription` symbol anywhere in source, and `nostr-tools` is not
a dependency of `@toon-protocol/client` (3.0.0) or of the root `package.json`.

It went in the 1.0 rewrite. `packages/client/CHANGELOG.md`, the `## 2.0.0` entry (_"**1.0 — a
pure TOON client.**"_, commit `efd6397`):

> **Removed:** all Nostr and relay logic (event signing, relay subscriptions, kind:10032 peer
> discovery, NIP-59 unwrapping, the render trust gradient, kind:5094 blob storage) […] Also
> removed: the `./render` subpath export, and the `@toon-protocol/core` and `@toon-protocol/sdk`
> dependencies.
>
> **Bootstrapping is one `GET`.** A connector's addresses, endpoints, sealing key, settlement
> chains and route prices now come from its own self-description, which this client reads from
> the `connector` URL you configure. There is no relay to subscribe to and no peer list to seed.

What replaced it is `packages/client/src/connector/self-description.ts:1-12`:

> This replaces peer discovery entirely. Earlier versions of this client learned a node's
> addresses, endpoints, sealing key and settlement facts by subscribing to a relay and reading
> announcements it published about itself; the Rust connector publishes nothing (ADR 0046 removed
> the announce, ADR 0022 says a connector _answers_, it never announces). One free,
> unauthenticated `GET` now carries every fact that mechanism used to scatter

The four `10032` hits left in `toon-client` source are all comments saying the mechanism is gone.
`packages/client/src/client/types.ts:51-56`:

> This is the whole of bootstrapping. There is no discovery, no relay and no peer list: one free
> `GET` on this URL returns every fact needed to transact with the node (`GET /ilp`, connector
> ADR 0050).

`packages/client/src/presets.ts:1-17` does carry four hardcoded devnet node URLs, and disclaims
itself: _"**These are conveniences, never authorities.** […] Two declarations of one fact is how a
mainnet node comes to be described as devnet, so nothing here is ever consulted in preference to
the document a node answers with."_ Nothing iterates it as a seed list.

Note in passing for ADR 0070/#1284: `toon-client` accepts `.anyone` **only** —
`packages/client/src/transport/hs-hostname.ts:28` is `export const HS_HOSTNAME_REGEX =
/^[a-z2-7]+\.anyone$/;` and `:68` rejects `.onion` by name, _"naming the network mismatch: that
is Tor"_.

### `@toon-protocol/core`'s `parseIlpPeerInfo` — **still there, and it is the buried half of a whole control plane**

This is the largest finding in this note, and ADR 0046's fallout list understates it: what
survives in `toon-protocol/toon` is not one orphaned parser but a **complete, wired discovery and
announce stack** — reader _and_ producer.

`packages/core/src/events/parsers.ts:78` — `export function parseIlpPeerInfo(event: NostrEvent):
IlpPeerInfo`, keyed off `packages/core/src/constants.ts:11`, `export const ILP_PEER_INFO_KIND =
10032;`. It still destructures the retired `blsHttpEndpoint`.

Its non-test call sites are all live:

| File                                                  | Where                              |
| ----------------------------------------------------- | ---------------------------------- |
| `packages/core/src/discovery/NostrPeerDiscovery.ts`   | import :8, calls :137, :208        |
| `packages/core/src/bootstrap/discovery-tracker.ts`    | import :12, call :163              |
| `packages/core/src/bootstrap/BootstrapService.ts`     | import :21, calls :744, :797, :897 |
| `packages/core/src/discovery/seed-relay-discovery.ts` | import :25, call :278              |

and they feed upward into `packages/core/src/compose.ts:25-29,313,338` (wiring `BootstrapService`

- `createDiscoveryTracker` into a `ToonNode`) and `packages/sdk/src/create-node.ts` (`:874`,
  `:889`, `:1060`, `:1082`, `:1157` → `bootstrapServiceInstance.bootstrap()`).

`packages/core/src/discovery/index.ts` exports a directory layer with four independent sources:
`NostrPeerDiscovery`, `GenesisPeerLoader`/`GenesisPeer`, `ArDrivePeerRegistry`,
`SocialPeerDiscovery` and `SeedRelayDiscovery`/`publishSeedRelayEntry`.
`packages/core/src/discovery/ArDrivePeerRegistry.ts:176` is a permanent-storage registry:

> ArDrive-based peer registry for permanent, decentralized storage of ILP peer info on Arweave.
> Read path: queries Arweave GraphQL gateway (free, no wallet needed).

with `const DEFAULT_GATEWAY_URL = 'https://arweave.net/graphql';`, called from
`BootstrapService.ts:214` (`const ardriveMap = await ArDrivePeerRegistry.fetchPeers();`).

**The genesis peer seeds ADR 0046's list means are here, as committed values** —
`packages/core/src/discovery/genesis-peers.json`, loaded at `GenesisPeerLoader.ts:8`, in full:

```json
[
  {
    "pubkey": "30fdd01d55c3efeb4c19c2cbeda8247cbc40ae9b15c026e9a301a263001fa7a9",
    "relayUrl": "wss://relay-ws.devnet.toonprotocol.dev",
    "ilpAddress": "g.toon.relay",
    "btpEndpoint": "wss://proxy.relay.devnet.toonprotocol.dev/ilp/btp"
  },
  {
    "pubkey": "499cdd71c7c3eab8d9b35f88ec9cde29018461e4bef86389004abcd7cfa1108a",
    "relayUrl": "wss://relay-ws.devnet.toonprotocol.dev",
    "ilpAddress": "g.toon.ario",
    "btpEndpoint": "wss://proxy.ario.devnet.toonprotocol.dev/ilp/btp"
  }
]
```

**And it publishes.** `packages/core/src/events/builders.ts:42`, `export function
buildIlpPeerInfoEvent(`, header at `:25-26` — _"Builds and signs a kind:10032 Nostr event from
IlpPeerInfo data"_ — using `import { finalizeEvent } from 'nostr-tools/pure';`. Two internal call
sites, both in `BootstrapService.ts`:

- `:846-858`, `private async publishOurInfo(relayUrl)`, _"Publish our own kind:10032 ILP Peer Info
  to a relay"_ → `await this.pool.publish([relayUrl], event);`, called at `:470`, `:957`, `:985`;
- `:512-522`, `private async announceViaIlp(result)`, _"Announce own kind:10032 as paid ILP
  PREPARE (Phase 2)"_, called at `:304` and `:936`.

`packages/core/README.md:5` still advertises it: _"protocol primitives: the TOON binary codec,
**Nostr peer discovery (kind:10032)**, bootstrap, ILP address derivation, settlement chain config,
and the structural `ConnectorNode` interface (`EmbeddableConnectorLike`)."_

`@toon-protocol/core` is at **3.5.0**; the repo was last pushed **2026-08-28**. `toon-client`
3.0.0 dropped the dependency on it, and the Rust connector never had one. So this stack is a
_complete_ peer-discovery-and-announce control plane that nothing in the current ecosystem
depends on — and one that only ever drove the **retired TypeScript connector**
(`EmbeddableConnectorLike`, `ToonNode`), never the Rust one's operator surface.

### `rig`'s genesis peer seeds — gone as values, alive as an inert seam

No `IlpPeerInfo` type, no parser, and **no code anywhere in `rig` subscribes to, filters on or
parses kind 10032**. The only `kinds:[10032]` literal in the tree is a fabricated stderr string
inside `packages/rig/src/cli/strict-json.test.ts:192,196,412`, which asserts that such chatter
lands on stderr rather than stdout; nothing emits it.

The genesis seed itself is a live dependency seam whose default is a hard-coded `undefined` —
`packages/rig/src/cli/fund.ts:267-281`:

```ts
/**
 * Default {@link FundDeps.loadGenesisSeed}: core's committed genesis peer,
 * loaded lazily …
 */
async function loadGenesisSeedDefault(): Promise<
  { relayUrl?: string; btpEndpoint?: string } | undefined
> {
  // There is no built-in network seed since 4.0: a node is named (TOON_CONNECTOR
  // / `rig entry <url>`), never discovered. Nothing configured means no seed.
  return undefined;
}
```

The seam around it is still wired as precedence step 3 of `resolveEffectiveNetwork()`
(`fund.ts:332`), still has a `seededOrigin: boolean` field (`:312`), and still produces
user-facing strings such as `"Inferred network 'devnet' from the built-in genesis seed"`
(`:566`) and `"(genesis seed)"` (`:634`) — which the tests exercise by injecting a seed
(`fund.test.ts:519-524`). `rig fund`'s shipped help text (`fund.ts:123`) still promises _"A
completely fresh install with NO origin configured anywhere infers devnet from the built-in
genesis seed, so a bare `rig fund` works with zero config"_ — **now false**. `rig init`'s usage
text (`init.ts:97`) carries the same stale promise while its body (`:180-183`) is already
correct.

Four doc comments still describe kind:10032 discovery as live and contradict two others in the
same package: `remote-state.ts:227`, `standalone-context.ts:70` and `:82-83`, and
`name.ts:1161-1165` against `standalone-mode.ts:9` and `connector-publisher.ts:9`.

### Is anything still producing the corpus?

**No live producer. Three dormant ones, in three repositories.**

The producer that actually fed the corpus was a shell loop on each devnet box, and ADR 0046's
`## Update (issue #1074)` is what removed it (_"each box's scheduled announce compose overlay and
the relay box's second `connector-rust.swap-announce.toml`"_). `toon-protocol/relay`'s
`packages/relay/src/launcher/relay.ts:130-142` still describes it from the outside:

> On the TOON devnet the only events carrying an `expiration` tag are kind:10032 node announces,
> published with a 600s TTL and refreshed by a shell loop every 240s — 2.5 refresh periods of
> margin, and the loop's failure backoff (5s doubling, capped at 240s) needs SEVEN consecutive
> failed publishes before an announce goes past its expiry. […] the store and swap announce loops
> PAY for each republish out of a payment channel

and `packages/relay/docs/retention.md:18-31` records what is left of it: three **stale** devnet
announces observed live (`pk=915d2990 g.toon.relay`, `pk=31e6a28e g.drew.relay`, `pk=b23599a6
g.toon.swap.sol` at `ws://127.0.0.1:3401`, with no expiry and no surviving key).

The relay itself still stores and serves them and publishes none of its own —
`packages/relay/docs/retention.md:27-41`:

> **That discovery mechanism is retired**, and this relay no longer publishes an announce of its
> own: a node is reached at its URL, and everything about its paid side is served on the
> connector's own `GET /ilp` (connector ADR 0046 / 0050). None of that makes this page obsolete,
> because the relay still _stores and serves_ whatever its clients write — kind:10032 events among
> them

There is no kind allowlist on the write path (`packages/relay/src/launcher/handlers/write-handler.ts`
branches only on the NIP-16 ephemeral range), and the store treats 10032 as
parameterized-replaceable (`packages/relay/src/storage/SqliteEventStore.ts:145-157`, with `:452-455`
— _"The `d` value is the empty string for events that carry no `d` tag — which is every kind:10032
announce on the network today"_). So the door stays open.

The three dormant producers:

1. **`@toon-protocol/core`'s `BootstrapService`** — `publishOurInfo` and `announceViaIlp`, above.
   Complete and wired; nothing current depends on the package.
2. **`packages/announcer` in this repository** — below.
3. **the retired box-side shell loops** — deleted by #1074.

**The one in this repository is maintained and unit-tested.**

`packages/announcer/README.md`:

> # @toon-protocol/announcer
>
> A standalone kind:10032 announcer sidecar for the Rust connector's client edge (connector#681).
>
> ## Why this is a separate service
>
> ADR 0022 ("a connector answers, it does not announce") and ADR 0006 ("mechanism not policy")
> forbid the Rust connector from pushing a kind:10032 self-announce itself. **This sidecar is the
> component ADR 0006 anticipated living "somewhere else"**: it never links against connector
> crates, never touches connector config, and never runs inside the connector process. It only
> **asks** the edge's already-public answers — […] `GET /ilp/identity` […] the x402
> payment-required greeting […] — and republishes them as a signed kind:10032 event on a timer
> (default 300s). No push-announce loop enters the connector binary; ADR 0022 stands.

It signs with its **own dedicated announce identity**, takes `ANNOUNCER_RELAY_URLS`,
`ANNOUNCER_RUST_EDGE_URL`, `ANNOUNCER_REFRESH_INTERVAL_SECS` and a NIP-40 `ANNOUNCER_TTL_SECS`,
and is described in `docs/architecture/source-tree.md:180`:

> `announcer` — A standalone `kind:10032` announcer sidecar. It is not the connector and never
> was: it never links against connector crates, never reads connector config and never runs in
> the connector process. It only asks the client edge's already-public answers and republishes
> them (ADR 0022, ADR 0006).

It is a **live npm workspace** run by `make test` (`source-tree.md:207-210`), and it was last
touched on 2026-09-03 (`fe996af2`, _"The execution condition leaves the wire (issue #1269, ADR
0069)"_) — i.e. kept current with protocol changes two weeks before this note.

**Nothing deploys it.** No compose file in this repository references it. The relay box's deploy
bundle (`toon-protocol/relay`, `deploy/docker-compose.yml`) has services for Caddy, the connector
(pinned `ghcr.io/toon-protocol/connector:rust-2026.08.28.1`) and the relay app — and no
announcer. The two places that still mention it are archaeology:
`infra/linode-relay/nginx/conf.d/node.conf:149-176` — _"`ANNOUNCER_RELAY_URLS` […] **used to**
reach this same payment-oblivious write ingress over the docker container network"_, keeping an
IP-restricted `location = /relay-write/write` alive for a caller that is no longer there — and
`infra/linode-relay/connector-rust.toml:90` — _"`identity_key_file` carried **the retired apex
announcer's** Nostr pubkey forward so a pinned genesis seed would not go stale (issue #870); with
no event to sign there is no signature to keep stable."_

So the corpus is stale exactly as ADR 0046 predicted (_"the corpus simply stops being refreshed,
and what those readers hold goes stale rather than wrong"_), and the only thing that could refresh
it is an unshipped sidecar in this repo. A generic NIP-01 relay would still _accept_ a kind:10032
write — the relay's paid route (`toon-protocol/relay`, `deploy/connector.toml:66-72`,
`prefix = "g.toon.relay"` → `http://relay:3100/write`) is kind-agnostic — so the door is open and
nobody walks through it.

### The connector itself

Confirmed clean. `grep -rn "10032\|IlpPeerInfo"` over `crates/` returns only doc comments and
the parse-to-reject path: `crates/connector-config/src/error.rs:1092` —

```
"'[node] {field}' was removed with the kind:10032 announce (ADR 0046, issue #1074): a …"
```

---

## 6. The verdict

### The controller organ is **EMPTY**.

Nothing in the TOON ecosystem decides a connector's leased routes or its peering. Every artefact
examined either pays a connector, is paid by one, or names one by URL out of band:

| Candidate                           | What it actually is                                                                                                                                                                                                                                                                       | Touches `POST /routes/leased` or `POST /peers`?                                      |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `rig` / `rig-web`                   | A git forge over NIP-34 events; "control plane" names the **repository** event graph. Pays one connector named by `rig entry <url>`.                                                                                                                                                      | No — zero hits for `POST /routes`, `POST /peers`, `Signature-Input`, `write_keys`    |
| `TOON_Network` + `provider`         | A **Provider Directory** for compute sellers: Profile / Listing / Liveness on relays, filterable by label. Defines no controller, no connector directory, no route leasing.                                                                                                               | No — `controller`, `10032`, `leased route`, `discovery`: zero hits in the 88 KB spec |
| `gateway`                           | A **naming and resolution** layer keyed by workload id under a signed Gateway Grant. Payment-oblivious; one plain `POST …/status`.                                                                                                                                                        | No                                                                                   |
| `tuzzy`                             | A credential **buyer** on a cron timer.                                                                                                                                                                                                                                                   | No                                                                                   |
| `toon-client` 3.0.0                 | The payer. One configured URL is the whole of bootstrapping; `discovery-subscription.ts` is deleted.                                                                                                                                                                                      | No                                                                                   |
| `@toon-protocol/core` 3.5.0 / `sdk` | A **complete** kind:10032 discovery-and-announce stack — four directory sources, committed genesis seeds, an Arweave peer registry, reader and publisher — driving the **retired TypeScript connector**. Orphaned: `toon-client` dropped the dependency, the Rust connector never had it. | No — it drove `EmbeddableConnectorLike`, not an operator surface                     |
| `packages/announcer` (this repo)    | A kind:10032 **producer** for the Rust edge. Maintained, tested, deployed nowhere.                                                                                                                                                                                                        | No — it only reads `GET /ilp/identity` and the x402 greeting                         |
| `hub` (townhouse)                   | "Operator product — apex orchestrator" — **no longer supported** per map #1.                                                                                                                                                                                                           | Not examined further                                                                 |

The connector's controller socket (`§0`) is finished and unused. The four things the map might
have hoped were a controller are each something else:

1. **`rig`'s "control plane" is a homonym.** It controls repositories, not connectors. It is a
   _client_ of a connector, and `rig entry` is the explicit statement that the network has no
   directory to consult.
2. **`TOON_Network`'s Provider Directory is real, built, paid for and tested — and it is a
   directory of compute sellers, not of connectors.** It publishes a connector's `connector_url`
   and `connector_seal_key` as a side effect of a provider advertising a lease, by hand, out of
   the provider app's own TOML. It is the closest thing in the ecosystem to _announcing_, it is
   exactly the "different object" ADR 0046 refused to define, and ADR 0011 gives it the safety
   property ND-14 demands (sign with the provider's key, pin, cross-check against the node's own
   self-description). But nothing in it decides a **peering**, and nothing in it leases a
   **route**.
3. **`gateway` is resolution, not discovery.** It finds a host nobody told it about, but only for
   a workload already delegated to it by a signed grant naming its own key, and it can be neither
   enumerated nor searched.
4. **`@toon-protocol/core`'s discovery stack is a corpse, not an occupant.** It is the only thing
   in the ecosystem that ever looked like a full controller — four directory sources, committed
   genesis seeds, an Arweave peer registry, kind:10032 read _and_ write — and it is complete and
   still compiles. But it drove the **retired TypeScript connector** through
   `EmbeddableConnectorLike`, never the Rust connector's operator surface; `toon-client` 3.0.0
   dropped the dependency; the repo has not been pushed since 2026-08-28. It occupies nothing
   because nothing runs it.

### Half-built, precisely and only in this sense

If the organ is called **half-built** rather than empty, the half that exists is the _publishing_
half of _announcing_, and it exists three times over, none of them for a connector that sells
carriage:

- **the pattern** — a signed, replaceable, expiring, relay-borne profile carrying a node's ILP
  address, connector URL, sealing key, settlement legs and liveness, with relay-side label
  filters — exists, is paid for, and is smoke-tested (`TOON_Network` §4,
  `provider/src/directory.rs`, `smoke-directory.mjs`). It advertises a **compute seller**;
- **a producer for connectors specifically** exists in this repository, tested and current
  (`packages/announcer`), and is deployed nowhere;
- **a whole discovery-and-announce layer** exists in `@toon-protocol/core` 3.5.0, orphaned;
- **the deciding half — whom to peer with, which routes to lease, at what cap — does not exist
  anywhere**, in any repository, public or private, running or abandoned. Not even the
  `@toon-protocol/core` stack decided a peering: it gathered peer info and handed it to an
  in-process TypeScript connector, which is fetching, not deciding.

`CONTEXT.md:118` locates the organ in _deciding_, not in fetching or publishing. By that
definition the answer is unambiguous: **EMPTY**.

### Two records that are less alive than the ticket assumes

- **ADR 0054 is accepted and not built, and its implementation issue was closed as
  `NOT_PLANNED`.** #1083 (_"An unsealed termination reject carries the terminating connector's
  URL (ADR 0054 / #1071) — closes #1026"_) was closed 2026-08-28 with: _"Closing for a clean slate
  (tracker sweep) — **not** because it stopped being true. I verified this one against the tree
  before closing it, and the defect it describes is present on `main` today."_ #1026 is closed
  too. ADR 0054's own falsifier still holds against this tree —
  `crates/connector-runtime/src/connector.rs:60` is `fn unsealed_termination_reject(message: &str)`,
  a message and nothing else.
- **`self-description-spec.md:196` and `:207-208` still stand**: _"**Not built:** the unsealed
  reject's URL (#1083, ND-15)"_ and _"Until it is built, a forwarded route is reachable only by a
  client that already knows the terminating node's URL out of band."_ Every discovery mechanism
  found above supplies exactly that out-of-band URL for the one kind of node it covers: `rig entry`
  by hand, the Provider Profile's `connector_url` for a compute provider. Neither reaches a
  connector that sells nothing but carriage.

---

## 7. Discrepancies found while answering, recorded as facts

None of these is this note's to fix; they are named so they are not re-derived.

1. **ADR 0046's fallout list is now wrong in two of its four items.** `toon-client`'s
   `discovery-subscription.ts` is deleted, and `rig` holds no kind:10032 reader and no seed
   values. The two that remain are `@toon-protocol/core`'s `parseIlpPeerInfo` — understated,
   since what is there is a whole stack, not a parser — and the genesis peer seeds, which are in
   `toon`'s `packages/core/src/discovery/genesis-peers.json`, not in `rig`.
2. **`rig` ships help text that is false.** `packages/rig/src/cli/fund.ts:123` and
   `packages/rig/src/cli/init.ts:97` both promise genesis-seed behaviour that
   `loadGenesisSeedDefault()` (`fund.ts:267-281`) removed by returning `undefined`
   unconditionally — the JSDoc directly above that function still says _"core's committed genesis
   peer"_. Four further doc comments (`remote-state.ts:227`, `standalone-context.ts:70,82-83`,
   `name.ts:1161-1165`) describe kind:10032 discovery as live and contradict two others in the
   same package.
3. **`rig`'s READMEs cite a deleted workflow.** Both the root `README.md` and
   `packages/rig-web/README.md` say rig-web is _"deployed to GitHub Pages
   (`.github/workflows/deploy-rig-web.yml`)"_; that file was deleted 2026-08-09 (`a851ad77`) and
   the live deployment is the Arweave manifest at `packages/rig/src/rig-pointer.ts:38-46`. The
   root README also pins `packages/rig` at `3.0.0` where `package.json` says `4.7.0`.
4. **`.onion` support is this repository's alone.** ADR 0070, amended by #1284, makes the
   connector accept a host ending in `.onion` **or** `.anyone`. Every downstream implementation
   examined accepts `.anyone` only: `gateway` has zero occurrences of `onion`
   (`src/dial.mjs`'s `isAnyoneHost` is `/\.anyone\.?$/i`), `tuzzy` has zero, and `toon-client`
   **rejects** `.onion` by name (`packages/client/src/transport/hs-hostname.ts:28,68`). A peer
   reachable only at a `.onion` host is therefore reachable by this connector and by no TOON
   client in the organisation.
5. **`infra/linode-relay/nginx/conf.d/node.conf:149-176` keeps an IP-restricted
   `location = /relay-write/write` alive for the announcer sidecar**, which nothing deploys. The
   nginx comment itself is written in the past tense (_"used to reach"_).
6. **TOON_Network §11 open item 4 is a live constraint on this connector.** _"Runtime route
   writes: the connector has none for terminated routes, so every listing change restarts it."_
   Correct against this tree: `CreateLeasedRouteRequest` and `POST /routes/peers` both name a
   `peer_id`, so only a **forwarded** route can be written at runtime; a terminated route with a
   `handler_url` is config-file-and-restart only.

---

## Sources

Read at HEAD on 2026-09-18 through `gh api` (contents/readme endpoints; no clones), except this
repository, read from the working tree.

| Repo                         | Visibility                     | What was read                                                                                                                                                                                                                                                                                  |
| ---------------------------- | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `toon-protocol/connector`    | public                         | `CONTEXT.md`, `CLAUDE.md`, `docs/adr/0046`, `0054`, `0049`, `docs/protocol/self-description-spec.md`, `operator-spec.md`, `crates/connector-operator/src/lib.rs`, `crates/connector-runtime/src/connector.rs`, `packages/announcer/`, `infra/linode-relay/`, issues #1026, #1083, #1, #4 |
| `toon-protocol/rig`          | public                         | root `README.md`, `CONTEXT.md`, `packages/rig/**`, `packages/rig-web/**`, `.github/workflows/`, `docs/adr/`                                                                                                                                                                                    |
| `toon-protocol/TOON_Network` | **private — access confirmed** | `CONTEXT.md`, `CLAUDE.md`, `docs/spec/toon-network-v1.md` (88 KB, read in full for §1–§12), `docs/adr/0011`, `docs/research/akash-toon-integration.md`                                                                                                                                         |
| `toon-protocol/provider`     | **private — access confirmed** | `src/directory.rs`, `src/nostr/{kinds,directory_events}.rs`, `provider.example.toml`, tree listing                                                                                                                                                                                             |
| `toon-protocol/gateway`      | public                         | `README.md`, `src/{dial,main,profiles,reasons,rewrite,resolve,serve,status,grant,grants,relays,config,follow,forward,gateway,kinds}.mjs`                                                                                                                                                       |
| `toon-protocol/tuzzy`        | public                         | `README.md`, `CONTEXT.md`, `package.json`, `src/{config,buy}.ts`, `docs/handoff.md`                                                                                                                                                                                                            |
| `toon-protocol/toon-client`  | public                         | full recursive git tree, `packages/client/CHANGELOG.md`, `src/connector/{self-description,ConnectorEdgeClient}.ts`, `src/client/types.ts`, `src/presets.ts`, `src/transport/hs-hostname.ts`, `README.md`                                                                                       |
| `toon-protocol/toon`         | public                         | `packages/core/src/{constants,types}.ts`, `events/{parsers,builders,nip40}.ts`, `discovery/**` (incl. `genesis-peers.json`), `bootstrap/BootstrapService.ts`, `compose.ts`, `packages/sdk/src/create-node.ts`, `packages/core/README.md`                                                       |
| `toon-protocol/infra`        | public                         | `README.md`, `sandbox/` tree, `sandbox/scripts/smoke-directory.mjs`                                                                                                                                                                                                                            |
| `toon-protocol/relay`        | public                         | `deploy/docker-compose.yml`, `deploy/connector.toml`, `packages/relay/src/launcher/relay.ts`, `src/storage/SqliteEventStore.ts`, `docs/retention.md`, `README.md`                                                                                                                              |
