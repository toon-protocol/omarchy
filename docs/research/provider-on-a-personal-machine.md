# What does `provider` already do, and could it run on a personal machine?

> **Research note, not a decision record.** Answers
> [#2](https://github.com/toon-protocol/omarchy/issues/2), a child of the wayfinder map
> [#1](https://github.com/toon-protocol/omarchy/issues/1) (_TOON as an OS facility on
> Omarchy_). Facts and quotes only; no recommendations. Written 2026-09-18.

**Sources read.** All primary, all first-party:

| Source                                                      | Access                        | Revision read                         |
| ----------------------------------------------------------- | ----------------------------- | ------------------------------------- |
| `toon-protocol/provider` (private, Rust)                    | readable with this `gh` token | `main` @ `0df7e57`, pushed 2026-09-17 |
| `toon-protocol/TOON_Network` (private, spec)                | readable with this `gh` token | `main` @ `303d549`, pushed 2026-09-17 |
| `toon-protocol/infra` (public)                              | readable                      | `main` @ `8c7dab5`, pushed 2026-09-17 |
| Omarchy 4.0.0.alpha on the machine this note was written on | `/usr/share/omarchy`          | pacman-installed                      |

Nothing here is guessed. Both private repos opened; no section of this note rests on inference
where a file could be read instead. Line numbers are from the tarballs of the revisions above.

---

## 0. The short version

`provider` is **not a prototype**. TOON_Network Milestones 1 through 5 are all closed
(`gh issue list --repo toon-protocol/TOON_Network`: #1, #10, #11, #12, #46 all `CLOSED`; only
Milestone 6 — a Continuation Token replacing the tenant's signature — is open). It sells leases
on OCI containers, resolves and verifies image bytes three different ways, holds warm standbys
and settles takeovers, publishes a signed directory, and — this is the part that matters here —
**already runs entirely without a public IP, without an inbound port, and without a DNS name**,
because a Hidden Provider's every reachable surface is a `.anyone` address minted over the `anon`
control port.

The assumptions a laptop breaks are therefore **not** the network ones. They are: a Docker
daemon socket the process may use, a lease table and blob cache that must outlive a restart, a
liveness publication that costs money on a cadence, and a **self-stop rule that treats going
quiet as a partition** — the closest thing in this codebase to a laptop lid.

---

## 1. What exactly does `provider` sell, and how does a buyer reach it?

### What it sells

A **Lease Interval** on a **workload** — an OCI container — priced in integer µUSDC.

> "**v1 workloads are OCI containers** (ADR 0001 scope; Paygress Docker backend)."
> — `TOON_Network:docs/spec/toon-network-v1.md` §9

> "**Prices:** every price is **per lease interval** (ADR 0003). No price is per second."
> — `TOON_Network:docs/spec/toon-network-v1.md` §2

The unit on sale is a `[[listings]]` entry — a tier at a version:

```toml
[[listings]]
name = "basic"            # stable across versions; an ILP address segment
version = 1
arch = "amd64"            # amd64 | arm64
lease_interval_s = 3600   # one payment buys this many seconds
price = 1000              # µUSDC per Lease Interval, on spawn and on extend
capabilities = []         # granted to every workload of this tier (spec §4.4)
capacity = 4              # leases of this tier that may run at once, all versions
[listings.resources]
cpu_millicores = 500
memory_mb = 512
storage_gb = 10
```

— `provider:provider.example.toml`

A listing may also carry `standby_price`, which sells **held capacity with nothing running** — a
Warm Standby reservation (spec §7). A price or resource change is a _new listing version_ with
its own routes, never a repricing of the old one (ADR 0009).

Two capabilities are spec'd; one is built:

| Capability | Here                                                                                                                                                           |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `docker`   | **Granted**: a per-lease `dind` sidecar — `docker:28-dind` pinned by digest, `--privileged`, the one privileged container of the lease, running no tenant code |
| `nesting`  | Not built: refused at config load                                                                                                                              |

— `provider:README.md` §Capabilities

### How a buyer reaches it

Through the provider's **own TOON connector**, on ILP routes the connector terminates and
forwards to the app as plain HTTP. The app is payment-oblivious:

> "The app is an ordinary HTTP app that runs behind the provider's own TOON connector. The
> connector terminates payment, seals and unseals the payload, and forwards a plain HTTP request
> — so this app reads no `X-TOON-*` header, and a tenant paying through hops is served
> identically to one paying directly."
> — `provider:README.md`

> "**Payment headers:** the provider app MUST NOT read `X-TOON-Payer`, `X-TOON-Amount` or
> `X-TOON-Chain`. They are absent whenever the packet came through a hop."
> — `TOON_Network:docs/spec/toon-network-v1.md` §2

The seven routes (spec §5, `provider:README.md` §Routes and the connector):

| ILP route                              | HTTP path                                      | Price           |
| -------------------------------------- | ---------------------------------------------- | --------------- |
| `<addr>.<listing>.v<n>.spawn`          | `POST /listings/<listing>/v<n>/spawn`          | listing price   |
| `<addr>.<listing>.v<n>.extend`         | `POST /listings/<listing>/v<n>/extend`         | listing price   |
| `<addr>.<listing>.v<n>.standby`        | `POST /listings/<listing>/v<n>/standby`        | `standby_price` |
| `<addr>.<listing>.v<n>.standby.extend` | `POST /listings/<listing>/v<n>/standby/extend` | `standby_price` |
| `<addr>.availability`                  | `POST /availability`                           | 0               |
| `<addr>.status`                        | `POST /status`                                 | 0               |
| `<addr>.terminate`                     | `POST /terminate`                              | 0               |

`toon-provider routes --config provider.toml` prints these as connector `[[routes]]` rows ready
to paste into the connector's config (`provider:src/main.rs`, `Command::Routes`).

**Discovery** is Nostr, not the connector. The provider publishes a **Provider Profile** (kind
`10432`), one **Listing** per tier (`30432`) and **Liveness** (`10433`) to its Relay Set, all
signed with its own Nostr key, all tagged `["L","toon.network"]`. A tenant finds a listing on a
relay, reads the Profile's `connector_url` and `connector_seal_key`, seals a **Lease Request**
(kind `4432`, never published) to that key, and pays the spawn route. This is directly relevant
to the map's note that "discovery is an open wound": TOON Network **does not** use the connector
for discovery — ADR 0046's "nothing, inside the connector" is satisfied by putting the directory
on relays, outside it.

> "**Pinned sealing key:** a Provider Profile MUST publish its connector's sealing public key. A
> tenant seals to that key and refuses if the URL's self-description reports another key
> (ADR 0011)."
> — `TOON_Network:docs/spec/toon-network-v1.md` §3

Once a lease is running, the buyer reaches the **workload** at the access details `status`
returns — `{ host, ssh_port, ports[] }`. SSH is the tenant's public key only, handed to the
container as `SSH_PUBLIC_KEY`; no password is ever issued. `host` is the provider's `public_ip`,
or, on a Hidden Provider, the lease's own `.anyone` address. A stable HTTPS name is **not** the
provider's job — that is a separate **Workload Gateway** app keyed by `workload_id`, which holds
a tenant-signed **Gateway Grant** and is the only thing that ever terminates TLS (spec §12,
ADR 0013).

> "A provider never owns a domain, runs ACME or terminates TLS, and never holds a tenant's
> certificate key."
> — `TOON_Network:docs/spec/toon-network-v1.md` §9

### What money actually moves

Three separate flows, and only one of them is the sale:

1. **The tenant pays the provider** — a TOON payment channel terminating at the provider's
   connector, per spawn and per extension.
2. **The provider pays to be discoverable.** Every relay write is a paid packet on the paid
   relay route (ADR 0007). The provider app does not hold money; a **directory publisher**
   sidecar (Node, `provider:tools/publisher/`) holds the channel and pays: "the provider's Nostr
   secret key never reaches this process. It holds money, not identity — it cannot forge a
   directory event, only decline to pay for one."
3. **`publish_url` unset means the provider publishes nothing, which is legal** — "it simply
   does not appear in the directory." A provider with no relay set still _reads_ the directory
   for free.

The publishing cost is small and quotable: "On the devnet relay a 60-second cadence costs about
$0.0014 a day." (`TOON_Network:docs/adr/0007-liveness-is-a-paid-replaceable-event.md`)

---

## 2. How does it drive `anon`?

### Yes — one `.anyone` address per lease, minted over the control port, never a HiddenServiceDir

`grep -rn "HiddenServiceDir" toon-protocol/provider/src/` returns **no hits**. There is no
on-disk hidden-service directory anywhere in this codebase. Addresses are created through the
daemon's control protocol with an ephemeral key:

`provider:src/anon_control.rs:11-23` (module header):

```
//     PROTOCOLINFO 1
//     250-AUTH METHODS=COOKIE,SAFECOOKIE COOKIEFILE="/var/lib/anon/control_auth_cookie"
//     AUTHENTICATE <hex of the cookie file>          (or AUTHENTICATE "<password>")
//     ADD_ONION NEW:ED25519-V3 Flags=Detach Port=22,127.0.0.1:40000 Port=443,…
```

Exactly four verbs are ever sent — `PROTOCOLINFO 1` (`:206`), `AUTHENTICATE` (`:240`/`:243`),
`ADD_ONION` (`:286`), `DEL_ONION` (`:422`). No `GETINFO`, no `SETCONF`, no `SAFECOOKIE`, no
`AUTHCHALLENGE`.

- **Transport:** one plain `TcpStream` to `[anon.control].addr`, opened per call
  (`provider:src/anon_control.rs:65`, `:192-199`). No unix socket. `:32-39` — "A CONNECTION PER
  CALL, rather than one held open for the process's life."
- **Key type:** `const KEY_TYPE: &str = "ED25519-V3";` (`:77`), used as
  `add_onion(workload_id, &format!("NEW:{}", KEY_TYPE), ports)` (`:354-356`). The daemon mints
  the key in memory; the provider stores the returned `PrivateKey` (`:340-343`) **on the lease
  record** (`provider:src/hidden_service.rs:88-101`, serde-serialised into the lease state file),
  and re-adds the same key on restart (`restore_address`, `:371-404`).
- **`Flags=Detach`** is what makes the address outlive the control connection (`:27-30`, `:286`).
- **Ports:** the lease's SSH forward plus each published port, same number both sides —
  `provider:src/provider/lease_address.rs:39-43`.
- **Teardown:** `DEL_ONION <service id>` on every ending; a `552` reply (no such service) is
  treated as success (`:422`, `:428`).

The exact bytes are asserted against an in-process stub control port
(`provider:tests/anon_control.rs:303-315`):

```rust
vec![
    "PROTOCOLINFO 1".to_string(),
    format!("AUTHENTICATE {}", COOKIE_HEX),
    "ADD_ONION NEW:ED25519-V3 Flags=Detach Port=40000,127.0.0.1:40000 \
     Port=41000,127.0.0.1:41000"
```

### What that requires of the host

| Requirement                                                                                                                                | Where it is enforced                                                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A running `anon` daemon with a **`ControlPort`** (it defaults to `0`) reachable over TCP at `host:port`                                    | `provider:src/provider/config.rs:865-870` — "`anon.control.addr {:?} is not host:port — the anon daemon's ControlPort`"                                                                                   |
| **`CookieAuthentication 1` with a readable `CookieAuthFile`, or `HashedControlPassword`** — exactly one of the two                         | `provider:src/anon_control.rs:220-221` — "Set CookieAuthentication 1 (and CookieAuthFile) or HashedControlPassword in the daemon's anonrc"; exclusivity at `:150-161` and `config.rs:871-888`             |
| The provider process must be able to **`read(2)` the cookie file**, re-read fresh on every connection                                      | `provider:src/anon_control.rs:232-240` — "cannot read the anon control cookie … it must be readable by this process"; `:90-92` — "read fresh on every connection: the daemon rewrites it on each restart" |
| A **SOCKS port** at `socks5h://host:port` for the provider's own outbound                                                                  | `provider:src/provider/config.rs:825-830`, `:890-899`; `src/outbound_proxy.rs:72-81`                                                                                                                      |
| `anon.forward_host` — where the daemon can reach this provider's published host ports. Default `127.0.0.1`                                 | `provider:src/provider/config.rs:159-165`                                                                                                                                                                 |
| An **internal Docker network** with the daemon attached at a fixed gateway address, running a transparent egress (`TransPort` + `DNSPort`) | `[anon.egress]`, `config.rs:831-843`                                                                                                                                                                      |
| A **private settlement RPC** — loopback, RFC 1918/ULA/link-local, or a name resolving only to those                                        | `config.rs:845-854`                                                                                                                                                                                       |

**No root and no privileged capability is required by the anon path itself.** There is no
`cap_add`, `NET_ADMIN` or `--privileged` anywhere in `src/anon_control.rs`,
`src/hidden_service.rs`, `src/provider/lease_address.rs` or `src/outbound_proxy.rs`. `NET_ADMIN`
appears only in the _Docker backend_, on the per-lease egress namespace owner
(`provider:src/docker.rs:107-111`, `:178`, `:558`, `:635`).

Plain `COOKIE` is used rather than `SAFECOOKIE` deliberately: "an attacker who can read it can
also speak SAFECOOKIE" (`provider:src/anon_control.rs:41-47`). The cookie **path comes from
config, never from the daemon's `PROTOCOLINFO` reply** — `:597-601`: "taking the daemon's word
for a path to read would let the thing being authenticated to choose the file."

### The startup gate

A provider that declares `hidden = true` **refuses to start** unless it can authenticate to the
control port:

```rust
pub async fn refuse_unreachable_control(config: &ProviderConfig) -> Result<()> {
    if !config.hidden { return Ok(()); }
    AnonControlService::from_config(config)?.preflight().await
```

— `provider:src/anon_control.rs:465-470`, called once from `ProviderService::run`

> "a provider that cannot create addresses must not advertise itself as one that can, and finding
> out at the first paid spawn would mean refusing a tenant who has already paid."
> — `provider:README.md`

### The provider's own outbound

`hidden = true` also routes **everything the process itself dials** through the SOCKS proxy —
relay websockets (every relay, not only `.anyone` ones), the TOON store gateway, upstream OCI
registries, the anonymous pull-token exchange, and the publish request. `socks5h` is enforced
twice and a `socks5://` config is a named startup refusal:

```rust
"anon.socks_proxy {:?} must be socks5h://<host>:<port>: only a socks5h proxy \
 resolves names on the far side, so no lookup leaves this provider"
```

— `provider:src/outbound_proxy.rs:72-78`

One exception: a **near publisher** (loopback, RFC 1918/ULA/link-local, or a name resolving only
to those) is dialled directly — `provider:src/outbound_proxy.rs:150-192`.

### One drift worth flagging

`provider:tests/anon_control.rs:14-30` says: _"The sandbox's `hs` profile daemon has
`ControlSocket 0` and no `ControlPort` today, so nothing there answers this yet"_ — while
`provider:README.md` says the sandbox's `anon-hs` "answers it at `172.30.1.2:9051` with the
cookie the sandbox mounts at `/var/lib/anon/control/control_auth_cookie`". **The README is the
current one**: `infra:sandbox/conf/provider-hs.toml:122,149` sets
`addr = "172.30.1.2:9051"` and `cookie_file = "/var/lib/anon/control/control_auth_cookie"`, and
`sandbox/docker-compose.yml:1890-1932` mounts `anon-hs-control:/var/lib/anon/control:ro` into the
provider. The test comment is stale. The real-daemon test is `#[ignore]`d and env-gated
(`TOON_ANON_CONTROL` / `TOON_ANON_COOKIE`), so nothing in CI settles it — which is how the comment
got to be stale.

---

## 3. What does it assume about its host?

Enumerated, with the laptop verdict on each.

| #   | Assumption                                                                                                                                                                                                                                  | Enforced / stated at                                                                                                              | Does a laptop break it?                                                                                                                                              |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **A Docker daemon socket this process may use.** The app shells out to the `docker` CLI; its container mounts `/var/run/docker.sock`                                                                                                        | `provider:src/docker.rs:1` ("Shells out to the `docker` CLI"); `provider:Dockerfile` ("it needs the CLI and a socket it may use") | **Yes, on Omarchy specifically** — see §4                                                                                                                            |
| 2   | **A public IP with inbound ports** — `public_ip:host_port`, no hostnames, no TLS                                                                                                                                                            | `provider:provider.example.toml`; `config.rs:786-790` ("`public_ip` is required")                                                 | **Yes — and it is already optional.** `hidden = true` _refuses_ `public_ip` (`config.rs:796-799`) and replaces it with per-lease `.anyone` addresses                 |
| 3   | **A contiguous host port range.** ids `1000..1999`, SSH forwards from `ssh_port_start = 40000`, 16 published ports per id from `workload_port_start = 42000`                                                                                | `provider:provider.example.toml`; both checked at load                                                                            | **No** when hidden: the daemon forwards to `forward_host` (default `127.0.0.1`), so the ports need only be reachable on loopback. No firewall hole, no NAT traversal |
| 4   | **Root-ish networking for the hidden egress** — an `--internal` Docker network the operator creates, an `anon` container holding `NET_ADMIN` writing `nat PREROUTING` redirects, and a per-lease namespace-owner container with `NET_ADMIN` | `provider:README.md` §Workload egress; `src/docker.rs:107-111`                                                                    | **Yes** — the _creation_ of these needs privileged Docker, though not root for the provider process                                                                  |
| 5   | **`br_netfilter` handling.** A host with `bridge-nf-call-iptables = 1` needs `-i <egress bridge> -o <egress bridge> -j ACCEPT` in `DOCKER-USER`                                                                                             | `provider:README.md` — "the reference host has no `br_netfilter`"                                                                 | **No** on the Omarchy machine checked: `/proc/sys/net/bridge/bridge-nf-call-iptables` does not exist; the module is not loaded                                       |
| 6   | **Durable state that outlives a restart.** `lease_state_path` is "the only record that a lease exists, so a restart without it strands every paid workload"                                                                                 | `provider:provider.example.toml`                                                                                                  | **No** — it is a file path, not a server facility                                                                                                                    |
| 7   | **A blob cache with room and no eviction.** "There is no eviction yet: give it room for every image this provider will run, or cap it" — a blob crossing `blob_cache_max_bytes` makes the spawn `no_capacity`                               | `provider:provider.example.toml`                                                                                                  | **Partly.** A laptop SSD is finite; the cap turns that into a refusal rather than a failure, but capacity is genuinely smaller                                       |
| 8   | **Always-on, and publishing on a cadence.** Liveness every `liveness_cadence_s` (default 60), each expiring five cadences out (ADR 0007)                                                                                                    | `provider:provider.example.toml`; ADR 0007                                                                                        | **Yes** — see the self-stop rule below                                                                                                                               |
| 9   | **The self-stop rule.** Five cadences reaching less than a strict majority of the Relay Set **stop the workload of every primary lease**                                                                                                    | `provider:README.md` §Stopping itself; spec §7.1                                                                                  | **Yes — this is the laptop hazard**                                                                                                                                  |
| 10  | **Startup takeover check.** On every boot, every live primary lease with a Standby Set asks its relays whether somebody took it over; a Takeover found marks the lease `taken_over` and it **stays stopped for the rest of the lease**      | `provider:README.md` §Stopping itself                                                                                             | **Yes**, for a machine that restarts often                                                                                                                           |
| 11  | **A loopback-only operator port.** `operator_bind_addr` default `127.0.0.1:8090`, validated to be loopback, "MUST NEVER be exposed off this host"                                                                                           | `provider:provider.example.toml`; `Dockerfile`                                                                                    | **No** — this is laptop-shaped already                                                                                                                               |
| 12  | **Money on a cadence.** A directory publisher sidecar holding a payment channel, funded, with `TOON_CHANNEL_STORE` that "must outlive a restart and die with the chain"                                                                     | `provider:tools/publisher/README.md`                                                                                              | **Partly** — and see §6 of the map's out-of-scope list, which rules the wallet out of the spec                                                                       |
| 13  | **A private settlement RPC** when hidden — a chain node on loopback or a private range, _refused_ if public                                                                                                                                 | `provider:src/provider/config.rs:845-854`; README §The settlement RPC rule                                                        | **Yes.** Running a Solana or EVM node on a laptop is the single heaviest assumption in the list                                                                      |

### The two that actually bite, quoted in full

**The self-stop rule** (`provider:README.md` §Stopping itself):

> "**Five cadences** in a row that reached less than a **strict majority** of its own Relay Set
> stop the workload of every **primary** lease it holds. […] A **standalone** lease is never
> stopped by this rule: it is in no Standby Set, so nobody is waiting to take it over. A provider
> with **no Relay Set** is exempt entirely — nothing it publishes reaches anyone, so nothing can
> take its workloads over either."

> "**At startup**, before anything is served, every live primary lease with a Standby Set asks
> the same question: a provider whose PROCESS was down is the loudest partition there is — its
> containers keep running on the host daemon beside it, its standbys see no Liveness and take
> over, and it comes back to a lease table that says `running`."

At a 60 s cadence, five cadences is **five minutes**. A closed lid, a suspended Wi-Fi link or a
`systemctl restart` longer than that is, to this code, a partition. Note the two exemptions
already written into it: a **standalone** lease (no Standby Set) and a provider with **no Relay
Set** are both untouched by the rule. A personal machine selling standalone leases is inside
those exemptions today, by configuration, with no code change.

Note this is _symmetric_ with — and distinct from — the map's quoted ADR 0030 hazard. ADR 0030 is
about two connector processes sharing one `state_dir` and silently ceasing to be paid. The
provider's hazard is the opposite shape: the machine goes quiet and its **workloads are stopped
on purpose**, loudly, with `status` answering `"stopped"`.

**The private settlement RPC.** This one has no exemption:

> "The URL's host must be loopback, a private range (RFC 1918 `10/8`, `172.16/12`, `192.168/16`;
> link-local; IPv6 `::1`, ULA `fc00::/7`, `fe80::/10`), or a hostname whose DNS resolution yields
> **only** such addresses […] A public address anywhere in the answer — `https://api.mainnet-beta.solana.com`
> resolved, or `8.8.8.8` written down — is refused, because an unproxied read of a public RPC
> links the operator's network location to its on-chain identity."
> — `provider:README.md` §Hidden Provider

It is gated on `hidden = true` only: "Without `hidden = true` none of the `[anon]` keys is
required and the RPC is not gated." So the choice a personal machine faces is a real one —
_hidden and running your own chain node_, or _not hidden and publishing a public IP_. The spec
offers no third position, and ADR 0008 explains why:

> "A hidden connector alone looks sufficient, and anytoon runs that way. But tenants may run any
> image (ADR 0004), and a tenant's code can simply look up its own public IP. Without the other
> conditions a 'hidden' provider is a clearnet provider with an onion front door."

---

## 4. Could an Omarchy machine run it as-is, or as-is minus specific assumptions?

**Omarchy ships Docker and enables it — and then deliberately withholds access to it.**

`/usr/share/omarchy/install/omarchy-base.packages` lines 23-25, 67, 132: `docker`,
`docker-buildx`, `docker-compose`, `lazydocker`, `ufw-docker`.
`/usr/share/omarchy/install/config/enable-services.sh`: `systemctl enable docker.socket`.

And then, in full, `/usr/share/omarchy/install/config/docker.sh`:

> "The Docker daemon runs as root and its socket is root-owned, so membership in the docker group
> is equivalent to passwordless root: any process in it can `docker run -v /:/host` and rewrite
> the host as root. We therefore do NOT add the install user to the docker group by default, so a
> single rogue process running as the user cannot silently escalate to root.
>
> The daemon is still enabled (docker.socket, in enable-services.sh) for system use. The Docker
> TUI (Super + Shift + D) and the Windows VM reach it through a polkit prompt, and the plain
> `docker` CLI runs under sudo."

The opt-out exists and is deliberately scary — `omarchy-setup-security-sudoless-docker` prints:

> "⚠️ WARNING: Enabling sudoless Docker adds you to the 'docker' group. […] equivalent to
> passwordless root. Any process running as your user could then run, for example:
> `docker run -v /:/host alpine   # full read/write of the host, as root` […] It is convenient
> for development, but it removes the protection Omarchy keeps by default."

This is the single sharpest collision in this whole ticket. Omarchy's long-lived daemons are
**systemd user units** (map's own finding: `WantedBy=graphical-session.target`,
`ConditionPathExists` toggles under `~/.local/state/omarchy/toggles/`). A user unit running
`toon-provider` cannot reach `/var/run/docker.sock` unless the user has taken an opt-in that
Omarchy's own text calls root-equivalent. Running the provider as a _system_ unit as root sidesteps
the group but is the same grant by another route.

The sandbox does not soften this: `provider`, `provider2` and `provider-hs` all bind-mount
`/var/run/docker.sock` straight in, and **`group_add` appears zero times in the whole compose
file** — i.e. the reference deployment simply assumes the socket is reachable by whoever is running
it. Nothing in either repo has ever had to think about a host that withholds it.

The firewall is the other Omarchy fact: `/usr/share/omarchy/install/config/firewall.sh` opens
with `ufw default deny incoming`, `ufw default allow outgoing`, with holes only for LocalSend
(53317) and Docker DNS. **Nothing listens inbound on a stock Omarchy machine.** For a clearnet
provider that is fatal. For a Hidden Provider it is irrelevant — the `.anyone` addresses forward
to `forward_host` (`127.0.0.1` by default) and carry no inbound port at all.

### What runs as-is, and what does not

| Assumption                                                  | Verdict on a stock Omarchy laptop                                                                                                                                                                                                                                    |
| ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rust binary, one TOML file, no env vars                     | **As-is.** `cargo build`; `cargo test` needs no Docker (the Docker tests are `#[ignore]`d)                                                                                                                                                                           |
| `anon` daemon with `ControlPort` + cookie                   | **As-is-able.** Nothing in `anon_control.rs` needs root; the cookie needs only to be readable by the process. The map records "zero occurrences of tor / onion / anon / ngrok / cloudflared" in Omarchy, so the daemon is a thing to bring, not a thing to configure |
| No public IP, no inbound port, no DNS name                  | **As-is.** This is what `hidden = true` is                                                                                                                                                                                                                           |
| `br_netfilter` / `DOCKER-USER` rules                        | **As-is.** Not loaded on the Omarchy machine checked; matches the "reference host has no `br_netfilter`" case                                                                                                                                                        |
| Loopback-only operator port                                 | **As-is**                                                                                                                                                                                                                                                            |
| Durable `lease_state_path` + blob cache                     | **As-is**, modulo disk size                                                                                                                                                                                                                                          |
| **Docker daemon access**                                    | **Not as-is.** Needs `omarchy-setup-security-sudoless-docker` (root-equivalent, opt-in, requires a reboot), or a root system unit, or a rootless Docker/Podman backend that does not exist in this codebase (`backend = "docker"` is "the only one today")           |
| **`--privileged` `dind` and `NET_ADMIN` egress containers** | **Not as-is.** Both follow from the above: they need a Docker daemon willing to grant them                                                                                                                                                                           |
| **A private settlement RPC on the machine**                 | **Not as-is.** A Solana validator or an EVM node on a laptop, for the hidden case                                                                                                                                                                                    |
| **A funded payment channel + publisher sidecar**            | **Not as-is**, and explicitly out of the map's scope                                                                                                                                                                                                                 |
| **Liveness cadence / self-stop under sleep**                | **Not as-is** for leases in a Standby Set. Exempt for standalone leases, and wholly exempt with no Relay Set                                                                                                                                                         |

**As-is minus specific assumptions:** a stock Omarchy machine can run a `hidden = true` provider
today if, and only if, four things are supplied: (a) Docker socket access, by an opt-in Omarchy
frames as root-equivalent; (b) an `anon` daemon with a control port, which Omarchy has never heard
of; (c) a private settlement RPC; (d) a funded publishing channel, or `publish_url` left unset and
therefore no directory presence at all. Everything else — the network shape, the firewall, the
ports, the state — is already laptop-shaped.

---

## 5. What does `TOON_Network` say a provider is, and is a personal machine contemplated?

The glossary definition, in full:

> **Provider**:
> An operator who sells leases on workloads running on hardware they control.
> _Avoid_: Host, seller, node
> — `TOON_Network:CONTEXT.md`

"Hardware they control" is the whole of it. There is no minimum, no uptime obligation, no stake,
no attestation, and no admission control anywhere in the spec. Note also what the glossary bans:
**"node"** is an avoided word for a provider — which matters for the map's vocabulary rule, since
this repo's `CONTEXT.md` uses "node" for a connector deployment.

**Is a personal machine contemplated anywhere?** **No.** A case-insensitive grep across the whole
`TOON_Network` repo for `laptop|personal|desktop|home machine|spare|idle|consumer|workstation|
residential|Omarchy|always-on|24/7|uptime` returns:

- `CONTEXT.md:15` — "consumer" listed as a word to _avoid_
- `CONTEXT.md:77` — "idle lease" listed as a word to _avoid_
- several hits in `docs/research/paygress-on-toon-architecture.md`, all describing _Paygress's_
  pre-fork design (`uptime_percent` in its offer schema, its heartbeat history, its WireGuard
  tunnel "for providers behind NAT")

That is the entire set. The spec contemplates neither a personal machine nor a server; it simply
never says. **The nearest thing to a statement is an absence**: no uptime field survives the fork
— the Paygress offer carried `uptime_percent` and `total_jobs_completed`, and the TOON Provider
Profile carries neither. What replaced them is **Liveness with an expiry** and the Warm Standby
machinery, i.e. the network's answer to "this provider may go away" is _structural_ (a standby
takes over) rather than _reputational_ (don't buy from flaky providers). Reputation is explicitly
parked: "The reputation math stays in the tree, compiling, with nothing calling it."
(`provider:README.md`)

Two spec facts point the other way, toward small providers being fine:

- `isolation` is a published, filterable label with exactly two values: `shared-kernel` and
  `dedicated-host` (spec §4.4). A laptop truthfully publishes `shared-kernel`, and a tenant that
  needs otherwise filters it out by tag.
- `capacity` is per-listing and provider-chosen. A provider selling `capacity = 1` is well-formed.

And one that points at it hard:

> "A warm standby takes over when the primary's liveness has expired on a majority of the
> primary's relay set. […] A crashed primary cannot announce that it stopped […] We accept that
> two copies may run briefly while a partitioned primary notices."
> — `TOON_Network:docs/adr/0010-takeover-on-liveness-expiry-without-state.md`

> "No workload state moves on takeover: the standby starts from the image. A standby set
> guarantees the service comes back, not its data."
> — same ADR

So the spec's model of an unreliable provider is: **you are replaced, and the replacement starts
from the image.** It is not hostile to a laptop; it is indifferent to one, and it has already
built the mechanism that makes a laptop's unreliability someone else's problem.

---

## 6. The falsifier check: is "an Omarchy machine sells things" already this project?

**Read the falsifier precisely.** It says: _"It is not `anytoon` or `provider` with a wallpaper.
Both already run a connector behind a hidden service. If a ticket's answer is 'compose file, but
prettier', the answer is no."_

### It fires, on the seller half. Plainly.

**"An Omarchy machine sells compute over TOON behind a hidden service" is `provider`.** Not
adjacent to it, not inspired by it — it is the thing. Every mechanism the description names is
built, tested and closed:

- selling: Milestone 1, closed
- image bytes from three sources, verified by digest: Milestone 2, closed
- warm standby and takeover: Milestone 3, closed
- **hidden provider, one `.anyone` address per lease, all egress through `anon`, own outbound
  proxied:** Milestone 4, closed
- a stable HTTPS name for what it sells, without the provider owning a domain: Milestone 5, closed

There is no missing capability between "server sells compute over TOON" and "laptop sells compute
over TOON" that this map could supply as a _protocol_ or _connector_ change. The gap is the four
items in §4: Docker access, an `anon` daemon, a private RPC, and money. Three of those are
**packaging and provisioning**. The fourth (money) the map has already ruled out of scope.

So if the destination is _"a personal machine sells compute"_, the honest answer is: **that is
`provider`, and the only remaining work is packaging.** Say it plainly, because the ticket asked
for plainly.

### What the falsifier does not touch

It is worth being equally precise about what this finding leaves standing, because the map's
**Destination** is not primarily a seller:

> "On day one the machine is a **client** and pays live connectors. As the install base grows, the
> same machinery lets it be a **connector** that other operators peer with."

Three of the four things the Destination names — _a machine identity_, _a spend limit every agent
on it inherits_, _a lifecycle that survives sleep and restart_ — have **no counterpart anywhere in
`provider` or `TOON_Network`**:

- **Identity.** The provider has a Nostr key (`nostr_private_key`, hex or `nsec1…`, in the TOML)
  and a connector signer. Neither is a machine identity an OS could hold on a user's behalf;
  both are file paths an operator writes.
- **A spend limit across agents.** There is no such concept. `provider` is a _seller_; it has no
  budget, no cap and no notion of a caller. The publisher sidecar has `TOON_DEPOSIT` (a channel
  deposit), which is a funding decision, not a policy an OS enforces across programs.
- **Sleep and restart lifecycle.** What exists is the _opposite_ of a solution: the self-stop
  rule and the startup takeover check are both mechanisms that assume going quiet is a failure
  and act on it. Nobody has written the laptop-shaped version.
- **Buying.** `provider` sells. The map's day one is a machine that _pays_. That is
  `toon-client`'s territory, and this ticket touched none of it.

So: the falsifier fires on the seller half, and does not reach the buyer half or the OS-facility
half. **"Omarchy machines sell compute" is `provider` plus packaging.** "Omarchy holds an
identity, a budget every agent inherits, and a lifecycle that survives a lid" is not in either
private repo, and nothing read for this ticket says it is.

### Two things this makes concrete for the map

1. **The provisioning fan-out argument gets a second beneficiary.** The map calls
   `omarchy-provision-user`'s five-agent-home symlink "the strongest 'needs Omarchy' argument
   found". If a machine is also a `provider`, the same fan-out is how _the machine's own agents_
   would find the skill to buy from it. That is an Omarchy-only fact, not a packaging fact.
2. **Nothing found here requires a connector change.** Consistent with the map's premise. The
   provider drives `anon` itself, over its own control connection, and uses the connector only as
   a payment terminator in front of an ordinary HTTP app. ADR 0070's `socks_proxy` and the
   `.anyone` host rule are exactly what it needs and already has.

---

## Appendix: `infra/sandbox` — the worked example, measured

`toon-protocol/infra` is public and `sandbox/` is "a complete TOON Protocol network on your
machine: one `docker compose` project with local chains (Solana, EVM, Arweave-sim), a local AR.IO
permaweb stack (gateway + Turbo bundler + ArNS), the TOON payment layer (four ILP connectors with
real, collateralised payment channels…) and the four first-party TOON apps" (`infra:README.md`).

### How heavy it actually is

`sandbox/docker-compose.yml` defines **45 services**, all under `name: toon-sandbox`, every one
behind a `profiles:` key — a bare `docker compose up` selects nothing. By profile: `full` 35,
`credentials` 15, `payments` 13, **`hs` 8**, `gateway` 2.

The `hs` profile — the Hidden Provider worked example — is those 8: `hs-ingress`, `anon`,
`anon-client`, `hs-provider-ingress`, `anon-hs`, `provider-hs`, `directory-publisher-hs`,
`provider-hs-connector`. It still needs `payments`' chains beneath it (`solana-validator`, `anvil`,
`relay`, seeders).

Weight, quoted:

- `solana-validator` — "agave test validator with AR.IO's five Anchor programs + Metaplex Core +
  the TOON `payment_channel` program preloaded at genesis, 2MB NameRegistry account preloaded"
- `anvil` — "local EVM chain (chain-id 31337) … real mainnet ANYONE + WETH9 bytecode, real
  Uniswap v3, two seeded pools with primed oracles"

Plus, per lease, containers the provider creates on the **host** daemon: a plain lease is one
(`toon-<id>`), a `docker`-capability lease adds a privileged `toon-<id>-dind`, and a **hidden lease
is three** (`sandbox/Makefile:216-217`: "`toon-<id>-egress` … `toon-<id>` … and `toon-<id>-ingress`").

**There is no statement anywhere in `sandbox/` about host RAM, disk or CPU**, and the word
"laptop" does not appear in its README. `docker-compose.yml` sets no `mem_limit`, `cpus`,
`deploy.resources` or `shm_size` on any service. `### Prerequisites` lists only: Docker with the
compose plugin, Node ≥ 20, four sibling checkouts, `*.localhost` resolving to loopback, and **free
host ports** — 17 named ports plus `40000–42599` and `43000–45599` for the two providers' workload
SSH and published ports.

The closest thing to a capacity note is in troubleshooting: "a `lazydocker` or `docker stats` left
open in another terminal is the usual answer … Every smoke that spawns through the hub … fails this
way on a **loaded daemon**, and the failure is the machine rather than the change under test."
Timings: `make up-hs` "Expect a minute or two of bootstrapping, more on a cold volume";
`smoke-m4` "Two to three minutes on a good day"; `smoke-m5` "six to seven minutes".

### How `provider` is wired to Docker and to `anon`

Verbatim, `sandbox/docker-compose.yml:1200-1221`:

```yaml
provider:
  profiles: ['payments', 'full']
  build:
    context: ${PROVIDER_CONTEXT:-../../provider}
  volumes:
    - ./conf/provider.toml:/etc/toon-provider/provider.toml:ro
    - /var/run/docker.sock:/var/run/docker.sock
    - provider-state:/var/lib/toon-provider
  expose:
    - '8080' # PRIVATE, only provider-connector dials this
```

No `ports:`, no `privileged`, no `cap_add`, **no `group_add`** (zero occurrences in the whole
file), no `user:`. The socket is bind-mounted into four services — `localstack`, `provider`,
`provider2`, `provider-hs` — and the rationale is stated at `:1195-1199`: "The app runs INSIDE
compose but its workloads run on the HOST daemon … so every SSH forward and published port lands on
the host and `public_ip` is the host's own loopback."

`provider-hs` (`:1890-1932`) adds two things and nothing else:

```yaml
      - anon-hs-control:/var/lib/anon/control:ro
    networks:
      default: {}
      hs-provider: {}
```

**That read-only named volume is the whole of the cookie-auth host requirement** — the daemon's
control directory shared into the provider's container. It then reaches `anon-hs` at pinned
addresses on the `hs-provider` network (`conf/provider-hs.toml:122,149`):
`socks_proxy = "socks5h://172.30.1.2:9050"`, `addr = "172.30.1.2:9051"`. So the sandbox _does_ have
a `ControlPort` — which is the side of the drift noted in §2 that matches the README, not the test
comment.

`privileged` appears **nowhere** in the compose file. The only capability grant is on `anon-hs`
(`:1826-1827`):

```yaml
cap_add:
  - NET_ADMIN
```

with `:1811-1814`: "they must exist before the daemon drops to `anond` and before any workload runs
— so the container installs them as root and then execs the image's own entrypoint unchanged."
Its command runs `iptables -t nat -A PREROUTING …`. Privileged containers _do_ exist at runtime —
the per-lease `dind` sidecars — but the provider creates them on the host daemon, not compose.

`conf/provider-hs.toml` sets `forward_host = "172.17.0.1"` — "the docker0 address every container
on this daemon reaches it at" — because the provider is itself containerised while its workloads'
ports land on the host. A provider running **directly** on a machine would use the default
`127.0.0.1`.

### What the hidden provider's connector config declares

`conf/connector-provider-hs.toml` (119 lines) is an ordinary connector config with:

```toml
client_edge_addr = "0.0.0.0:3000"
state_dir = "/app/state"

[signer]
key_file = "/app/data/signer.key"

[node]
addresses = ["g.toon.provider-hs"]
http_endpoint = "http://<56-char base32>.anyone/ilp"
btp_endpoint = "ws://<56-char base32>.anyone/ilp/btp"
```

Seven `[[routes]]` — the four paid spawn/extend rows for `basic` and `smoke` at `price = 1000`, and
`availability` / `status` / `terminate` at `price = 0` — each `handler_url` pointing at
`http://provider-hs:8080/…`. Two settlement backends (EVM at `http://anvil:8545`, Solana at
`http://solana-validator:8899`). `[operator]` with `bearer_token_file` and `write_keys_file`.

Two absences are load-bearing for this ticket:

- **No `socks_proxy` key in the connector config.** The SOCKS proxy lives in the _provider's_
  config, not the connector's. The connector is reached _at_ an `.anyone` address; it does not dial
  one.
- **No peering.** The header says: "THERE IS NO PEERING … Hence no `[[peers]]`, no
  `[[peer_channels]]`, and no `peer_expose`." A hidden provider in the sandbox is a terminating
  node only.

### What the smokes prove

`scripts/smoke-milestone4.mjs` (743 lines) is the Hidden Provider acceptance test, and it is
unusually strict about exactly the properties a personal machine would want:

- a preflight that reads the provider's `settlement_rpc_url`, **both** of its connector's
  settlement RPCs and its publisher's RPC and asserts none is a public URL;
- `assert(!IPV4.test(spawned.text()), 'no IPv4 address appears anywhere in the answer')`;
- `access.host` matches `.anyone` **and is not the connector's own address**;
- on the host daemon: the three containers exist, the workload's `NetworkMode` is
  `container:<egress>`, the egress container is on exactly one network, `anon-hs` holds the lease
  address in `GETINFO onions/detached`;
- SSH over `ProxyCommand nc -X 5`, then an HTTP fetch of `http://<lease>.anyone:<host_port>/`;
- **from inside the workload**: a what-is-my-IP service sees an address that is neither the
  provider's host IP nor any local IPv4; `ip route` has exactly two lines; `ping -c 2 -W 3 1.1.1.1`
  → "2 packets transmitted, 0 received";
- after terminate: no container within 30 s, the address gone from `onions/detached`, and the
  address no longer answering through the proxy;
- and the money check: `assert(bookAfter - bookBefore === expected)` where
  `expected = BigInt(paidCalls) * L.price` — "the free calls added nothing, and no hub took a fee."

`scripts/smoke-hs.mjs` is the buyer-side rehearsal over a circuit (`anytoon`'s credentials route),
with the same preflight discipline: it reads `GET /ilp` out of band and asserts
`described.httpEndpoint === "http://<address>/ilp"` — "the check catching 'a client dials what a
node publishes'".

### The buyer's path to a workload, in the sandbox

`conf/workload-gateway.conf` is an **env file**, not a config file: "the gateway has no config
file." `GATEWAY_DOMAIN=gw.localhost`; every granted workload is served at
`<canonical label>.gw.localhost` where the label is the lowercase unpadded base32 of the workload id
(52 chars, spec §12.2); `*.localhost` resolves to loopback "with nothing added to anyone's DNS".
TLS is a self-signed wildcard cert for `*.gw.localhost` on 3443, plain HTTP on 3280.
`TOON_SOCKS_PROXY=socks5h://anon-client:9050` is how it dials `.anyone` members. The gateway's
health check asserts an ungranted hostname answers `503` with `toon-gateway-reason: no_grant`.

So the full buyer path is: `http://<52-char-base32>.gw.localhost:3280/` → gateway reads the Gateway
Grant off the relay → calls the provider's **free** `status` route → proxies to the workload's
published port.

### What the sandbox is and is not evidence for

It _is_ run on developers' machines — "break it, wipe it, cold-start it in ~2 minutes" — which
proves the software fits on one. It is _not_ evidence that a personal machine can be a live seller:
every chain under it is local and disposable, every key is "a valueless committed throwaway", and
the whole thing is a compose project the operator starts and stops by hand.
