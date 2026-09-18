# What breaks when a personal machine sleeps, restarts, or runs two connectors?

> **Research note, not a decision record.** Answers
> [toon-protocol/omarchy#5](https://github.com/toon-protocol/omarchy/issues/5), a child of the
> wayfinder map [toon-protocol/omarchy#1](https://github.com/toon-protocol/omarchy/issues/1).
> Both were transferred out of `toon-protocol/connector` (where they were #5 and #1) while
> this note was being written; the old numbers still redirect. The note stays in this repository
> because everything it reads is here. Facts, file paths and quotes;
> no recommendations. Every connector claim below is read off this working copy at
> `9d48ba26`; the `anon` section is read off upstream Tor/Anyone Protocol primary sources; the
> Omarchy section is read off `/usr/share/omarchy` on a live Omarchy 4.0.0.alpha install
> (`omarchy-settings` 4.0.3-1). Written 2026-09-18. No tests were run.

## Verdict first

| #   | Subject                                             | Real problem on a personal machine?                                                                                                                                                                                                                                                                            |
| --- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Two connectors over one `state_dir`                 | **YES — the only genuinely dangerous item.** No lock of any kind exists. Both directions (free service inbound, a permanently stalled peering outbound) are live.                                                                                                                                              |
| 2   | Sleep — in-flight packet, BTP session, client lease | **No**, with one narrow real cost: a packet already admitted at a _terminated_ route is charged and never delivered. Everything else self-heals.                                                                                                                                                               |
| 3   | Network change — `anon` and the address             | **No.** The address is a function of one key file; an IP is never in a descriptor. Cost is under ~2 minutes of unreachability, plus up to 180 minutes during which clients holding the _old_ descriptor get a `T01`. Not money. One separate hazard found (§3.6): an `anon` major upgrade changes the address. |
| 4   | Restart with no network                             | **Conditionally yes** — a node declaring `[settlement.*]` hard-exits 1 with no retry. Trivially absorbed by `Restart=always`, which is Omarchy house style anyway. Not a correctness problem.                                                                                                                  |
| 5   | Reject semantics for a sleeping seller              | **Confirmed not a problem.** `T01` throughout, and a forwarded route's claim is rolled back.                                                                                                                                                                                                                   |
| 6   | Omarchy's machinery                                 | **No gap that blocks anything.** No suspend/resume hook event exists, but nothing here needs one.                                                                                                                                                                                                              |

The working hypothesis in the ticket — _intermittence is mostly benign, `state_dir` concurrency
is the dangerous item_ — is **confirmed**, with one correction (§2.1) and one sharpening: the
concurrency hazard is worse than ADR 0030 states, because ADR 0030 describes only the outbound
half and the one guard it ever shipped was deleted with the `announce` verb (§1.4).

---

## 1. Concurrency — is there any lock on `state_dir`?

### 1.1 There is no lock. None. Anywhere.

```
$ grep -rn "flock\|LOCK_EX\|pidfile\|lockfile" crates/ local/ Dockerfile*
```

returns nothing but `Cargo.lock` noise and in-process `std::sync::Mutex` / `tokio::sync::Mutex`
call sites. There is no `flock(2)`, no `O_EXCL` sentinel, no pidfile, no lockdir, no
`fs2`/`file-lock`/`fd-lock` dependency, and no boot-time "is something already using this
directory" probe. The `state_dir` is opened exactly as any other directory would be.

`crates/connector-cli/src/runtime.rs:1082` — the one function that opens anything under
`state_dir`:

```rust
fn open_journal(state_dir: &Path, name: &str) -> Result<Arc<dyn Journal>, RuntimeError> {
    std::fs::create_dir_all(state_dir).map_err(...)?;
    let path = state_dir.join(name);
    let journal = FileJournal::open(&path).map_err(...)?;
    Ok(Arc::new(journal))
}
```

and `crates/connector-runtime/src/journal.rs:206`:

```rust
pub fn open(path: impl AsRef<Path>) -> Result<FileJournal, JournalError> {
    let path = path.as_ref().to_path_buf();
    let file = OpenOptions::new()
        .create(true)
        .append(true)
        .read(true)
        .open(&path)?;
    Ok(FileJournal { path, file: Mutex::new(file) })
}
```

`create(true).append(true).read(true)` — never `create_new`, never `custom_flags(O_EXCL)`. A
second process opening the same path succeeds silently and immediately.

### 1.2 The four files under `state_dir`, and how each behaves under two writers

Named at `crates/connector-cli/src/runtime.rs:1063-1073`:

| File                     | Shape                                                                                            | Two-writer behaviour                                                                                                                                                                                      |
| ------------------------ | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `peer-claims.log`        | append-only, `O_APPEND` + `sync_data`                                                            | Each process folds **only what it read at startup** into its own in-memory tables. `append` can also tear a line — see §1.2a.                                                                             |
| `client-edge-claims.log` | same                                                                                             | same                                                                                                                                                                                                      |
| `runtime-peers.json`     | whole-table JSON, temp file + `rename`                                                           | Last writer wins the **whole table**. A row added by A is silently erased by B's next write.                                                                                                              |
| `evm-channel-index.json` | whole-table JSON, temp file + `rename` (`connector-settlement-evm/src/channel_index.rs:563-590`) | Same, and worse: this one is written by a **background poller** (`runtime.rs:1810`, `tokio::spawn(syncer.run(..., DEFAULT_POLL_INTERVAL))`), so it is overwritten on a timer without any operator action. |

The repo already knows the shape of the hazard in-process — `runtime.rs:1058-1062` explains why
the two journals are two files rather than one:

> Two files rather than one because they are two different books ... and because each is replayed
> by a different owner at startup; sharing one file would mean each replaying the other's entries
> and **each holding a second writer's file handle on the same path.**

That reasoning is applied to two _books in one process_. Nothing applies it to two processes.

### 1.2a A torn line is possible, and one torn line bricks the next boot

`FileJournal::append` (`journal.rs:222`) is

```rust
let mut file = self.file.lock().expect("file journal lock poisoned");
writeln!(file, "{}", encode_line(entry))?;
file.sync_data()?;
```

`writeln!` on a bare `File` (no `BufWriter`) goes through `Write::write_fmt`, which issues a
`write(2)` **per format piece** — the entry text and then the `\n` separately. Under `O_APPEND`
each individual `write` lands atomically at the end of file, but two processes can interleave
_between_ those writes, producing `…<A's entry><B's entry>\n\n`. `append_batch` (`:232`) builds
one `String` and does a single `write_all`, so the group-commit path is much narrower — but the
single-entry path is used wherever a batch is not.

A torn line is not shrugged off. `decode_line`'s catch-all is `_ => Err(corrupt())`
(`journal.rs:190`), and `read_all` is

```rust
BufReader::new(file).lines().map(|line| decode_line(&line?)).collect()
```

— a `collect()` into `Result`, so the **first** bad line aborts the whole read. That becomes
`RuntimeError::JournalUnreplayable` at `runtime.rs:1093`, i.e. `exit(1)`. One interleaved write
makes the node permanently unbootable until someone hand-edits the file.

This is arguably the _least_ dangerous of the four two-writer outcomes, because it is loud. The
silent ones below are the problem.

### 1.3 Why a replayed-once, in-memory watermark is the whole hazard

Both claim ledgers read the journal exactly once, at build, and hold the result in an in-process
lock for the life of the process.

Peer side — `crates/connector-runtime/src/claim.rs:780`:

```rust
pub fn set_journal(&mut self, journal: Arc<dyn Journal>) -> Result<(), JournalError> {
    let entries = journal.read_all()?;
    let (outbound, inbound_watermarks, projection) = Self::rebuild_from(...);
    let projection = Arc::new(RwLock::new(projection));
    let outbound = Arc::new(RwLock::new(outbound));
    let inbound_watermarks = Arc::new(RwLock::new(inbound_watermarks));
    ...
}
```

Client edge — `crates/connector-client-edge/src/claim_gate.rs:488`:

```rust
pub fn restore(
    channels: ClientChannelRegistry,
    journal: Arc<dyn Journal>,
) -> Result<ClientClaimGate, JournalError> {
    let watermarks = Arc::new(RwLock::new(replay_watermarks(&journal.read_all()?)));
    ...
}
```

`read_all()` is never called again. There is no re-read, no inotify, no version counter, no
generation check. Two processes are therefore two independent watermark tables over one file, and
they diverge from the first claim either one accepts.

**Two failure directions, not one. ADR 0030 documents only the second.**

**(a) Inbound — free service, silently.** Process A and process B both replay watermark `N` for a
client channel. A buyer presents claim `N+1`; A accepts it, serves, journals. The same claim
bytes presented to B find B's in-memory watermark still at `N`, so B accepts the _same claim_ and
serves again. This needs no malice — a buyer retrying after a timeout, or anything round-robining
two ports, produces it. `docs/operators/onion-endpoint-bringup.md:194` states the single-process
version of this (an unpersisted `state_dir`) as "a restarted node accepts a nonce it already
spent — free service, silently"; two live processes are that same sentence without needing a
restart.

**(b) Outbound — the peering stops being paid, permanently.** ADR 0030's own words
(`docs/adr/0030-…:153-157`):

> **A second process must not share a serving node's `state_dir`.** A node's outbound peer-claim
> ledger is replayed from the journal at startup and held in memory, and the journal has no lock.
> Two processes over one `state_dir` both resume at nonce N, both sign N+1 against different
> cumulative amounts, and the counterparty refuses one as a replay — after which the serving
> node's claims never advance the far side's watermark again and the peering silently stops being
> paid.

This one does not self-heal even after one process is killed: the survivor's local nonce is now
behind the counterparty's watermark and every claim it signs is refused as non-advancing.

**(c) A third direction ADR 0030 does not name: the settlement key.** Both processes read the
same `[settlement.evm] key_file` and each wraps it in its own
`NonceManagerMiddleware::new(signer, address)`
(`crates/connector-settlement-evm/src/lib.rs:705`), which seeds from the chain once and then
increments locally. Two processes settling from one key issue transactions at colliding EVM
nonces; one of each pair is dropped or replaced by the node.

### 1.4 The only guard that ever existed was deleted with the `announce` verb

ADR 0030 §159-163 describes a guard:

> `--via-own-routing` therefore refuses when all three of these hold: the config names a
> `state_dir`, the destination resolves to a `peer_id` route ..., and something is already
> listening on this config's client edge.

That guard lived on `connector announce`, which ADR 0046 / #1074 removed. `grep -rn
"via_own_routing" crates/` returns **zero hits** — the string survives only in
`docs/adr/0030-*.md` and `docs/devnet-pricing.md`. So the repository today ships no guard of any
kind against a second process over one `state_dir`, and the one it had was scoped to a
subcommand rather than to the directory.

### 1.5 What partially saves you today, and exactly how far it goes

`crates/connector-bin/src/main.rs:37`:

```rust
let server = axum::Server::bind(&node.client_edge_addr).serve(node.router.into_make_service());
```

`hyper` 0.14.32 `Server::bind` **panics** on a bind failure
(`hyper-0.14.32/src/server/server.rs:79-84`):

```rust
pub fn bind(addr: &SocketAddr) -> Builder<AddrIncoming> {
    let incoming = AddrIncoming::new(addr).unwrap_or_else(|e| {
        panic!("error binding to {}: {}", addr, e);
    });
    ...
}
```

So **two processes started from the same config file collide on `client_edge_addr` and the second
dies** — `EADDRINUSE`, panic, exit 101. This is a real and load-bearing accident, and it is the
reason the hazard has not bitten this repo's fleet.

Its limits are exact, and all three are ordinary on a personal machine:

1. It is a **port** guard, not a `state_dir` guard. Two configs with different
   `client_edge_addr` and the same `state_dir` both serve happily. That is the natural shape of
   "I copied `connector.toml` to try something".
2. It fires **after** the whole runtime is built — config load, key reads, all settlement RPC
   dials, `create_dir_all(state_dir)`, both journal opens and both replays, and
   `tokio::spawn` of the EVM channel-index syncer (`runtime.rs:1810`), the rate poller
   (`runtime.rs:1981`) and the client-channel reaper (`runtime.rs:2463`). A duplicate that dies at
   bind has already had a background task with a live `rename`-over-`evm-channel-index.json` loop
   running against the survivor's file.
3. It says nothing about `state_dir` in the error. The message is `error binding to
127.0.0.1:3000: Address already in use`, which points an operator at the port.

There is no test anywhere in the workspace that starts two processes over one `state_dir`, and no
documentation of the hazard outside ADR 0030 and the onion runbook.

### 1.6 The binary's _other_ verb is harmless, and that is why the guard was removable

`connector send` never loads a config and never touches `state_dir`.
`crates/connector-cli/src/lib.rs:219`:

```rust
pub async fn run<S: AsRef<str>>(args: &[S]) -> Result<Command, CliError> {
    match parse_args(args)? {
        Invocation::Serve { config_path } => {
            let config = Config::load(Path::new(&config_path))?;
            let runtime = build(&config).await?;
            ...
        }
        Invocation::Send { options } => { ... }
        Invocation::PrintKeyid { key_file } => ...
    }
}
```

`Config::load` and `build` are reached only on the `Serve` arm; `grep -n "state_dir\|Config::load"
crates/connector-cli/src/send.rs` returns nothing. So running `connector send` (or
`--print-keyid`) beside a serving node is safe by construction. The residual risk is entirely
"two `Serve` processes", which is a packaging and lifecycle question rather than a CLI one.

`local/stack-guard.sh` is the only "don't run two of these" machinery in the repository, and it
is at the **docker-compose project** level, not the process level — it refuses a second `make
local-up` and reports which checkout owns the stack, explicitly to avoid the failure arriving "as
a port bind failure inside a container". It has no counterpart for the binary.

---

## 2. Sleep

### 2.1 An in-flight packet: the one real cost, and it is by design

A terminating node that admits a claim and _then_ suspends has already charged the payer.
`crates/connector-runtime/src/connector.rs:3297-3304`:

> the claim paying for this packet was taken _before_ the app was called — at the client edge by
> `ClientClaimGate::ingest`, or on a peer arrival by `ClaimBook`, in both cases with the watermark
> advanced and journalled before routing began — and only a forwarded route's terminal reject
> gives one back (`roll_back_uncarried_forward`, issue #1012).

`roll_back_uncarried_forward` is scoped exactly (`crates/connector-client-edge/src/lib.rs:1368`):

```rust
if !is_forwarded_route || !matches!(response, PacketResponse::Reject(_)) {
    return;
}
```

So: a laptop that **terminates** a route, admits the claim, and suspends mid-app-call keeps the
money and delivers nothing. The buyer's HTTP request dies with the socket. This is ADR 0064's
deliberate asymmetry, not a bug —
`docs/adr/0064-a-deadline-bounds-the-wait-for-an-app-not-the-answer.md`:

> **The claim was taken before the app was called.** ... A termination's reject rolls back nothing.

It is bounded to **one packet's `price`, once per suspend**, and only for a packet already in
flight at the moment the lid closes. It is the single place where the ticket's "intermittence
costs callers nothing" hypothesis needs a correction.

On resume the wall clock has jumped. Expiry is checked against `self.clock.now()`
(`connector.rs:1887`), so every packet still in flight is past `expires_at` and answers `R00`
(`reject_ineligible` → `RejectCode::r00_transfer_timed_out()`, message `"prepare has expired"`).
That is correct and is the same code path an ordinary slow path takes.

### 2.2 A BTP peer session: one `T01`, up to 30 s, then it self-heals

There is **no websocket ping/pong keepalive and no TCP keepalive** configured anywhere in
`connector-peer-btp` or `connector-btp` (`grep -n "ping\|keepalive"` over
`crates/connector-peer-btp/src/ws.rs` and `crates/connector-btp/src/session.rs` returns nothing).
Liveness is discovered by traffic, which is the right shape for a machine that is off most of the
day but means the discovery is lazy.

The mechanism, added by #1240 (`crates/connector-peer-btp/src/dial.rs:340`):

```rust
async fn session(&self, state: &RelationState) -> Result<BtpSessionHandle, DialError> {
    let mut slot = state.session.lock().await;
    match slot.as_ref() {
        Some(handle) if !handle.is_gone() => return Ok(handle.clone()),
        Some(_) => { tracing::info!(..., "peer session is gone; redialling"); *slot = None; }
        None => {}
    }
    let handle = self.dialer.dial(&state.relation.peer_id, &state.relation.endpoint).await?;
    *slot = Some(handle.clone());
    Ok(handle)
}
```

`is_gone()` is `self.replies.is_closed()` (`crates/connector-btp/src/session.rs:204`), and the
writer task is stopped when the read loop ends (`ws.rs:220-227`) precisely so that a dead socket
closes the channel. Two cases on resume:

- **Far side sent a FIN/RST** (it noticed first, or a NAT sent one): the read loop ends, the
  channel closes, the next packet sees `is_gone()` and redials. Cost: nothing.
- **The connection silently evaporated** (NAT table expiry during suspend, no RST): the first
  write after resume succeeds into the void, then `OUTBOUND_ANSWER_TIMEOUT` fires. That constant
  is 30 s (`crates/connector-btp/src/session.rs:60`), and the peering's own
  `peer_answer_timeout_ms` / `claim_ack_timeout_ms` default to `30_000`
  (`crates/connector-config/src/config.rs:1685-1686`). A timeout is **never** retried —
  `dial.rs:404-411` retries only `OriginateError::SessionGone`, because "the frame is on the wire
  and the peer may be acting on it" — so it falls to `drop_session` and the packet answers `T01`.
  The _next_ packet dials fresh.

Worst case for a peering after a suspend: **one packet, `T01`, ~30 s**, self-healing, with no
money moved and no state corrupted. §6.3's byte-identical retransmission cache
(`dial.rs:428+`) means a claim that rode a lost frame is re-sent byte-for-byte rather than
re-rendered at a new timestamp, so nothing on the recovery path can be mistaken for a replay.

### 2.3 A client session lease: 120 s, checked lazily, and it answers `T01`

`crates/connector-client-edge/src/session_registry.rs:82`:

```rust
pub const SESSION_LEASE_BACKSTOP_TTL: Duration = Duration::from_secs(120);
```

Its header (`session_registry.rs:16-19`) says what it is and is not:

> [`SESSION_LEASE_BACKSTOP_TTL`] exists only for the case that lifecycle cannot see — a socket
> that looks alive at the TCP layer but has stopped producing frames — never as the primary
> mechanism.

`resolve` (`:190`) evicts lazily on read, not by a sweep:

```rust
let stale = bindings.get(address).is_some_and(|binding| {
    now.saturating_sub(binding.last_seen) > SESSION_LEASE_BACKSTOP_TTL.as_secs()
});
if stale { bindings.remove(address); return None; }
```

Every failure path — no binding, stale binding, wrong fencing generation, `SessionGone`,
`Timeout` — funnels into one reject (`:288`):

```rust
fn no_live_session_reject(address: &str) -> Reject {
    Reject {
        code: RejectCode::t01_peer_unreachable(),
        ...
        message: format!("no live client session for '{address}'"),
        accumulated_cost: 0,
    }
}
```

`accumulated_cost: 0`. The doc comment at `:205-212` is explicit that `R00` would be the wrong
answer here — it is "singled out as the wrong answer for 'the provider's Wi-Fi dropped.'"

Note the cross-plane consequence stated at `session_registry.rs:72-81`: this value is published
on the wire as `extra.sessionLeaseTtlMs` in every x402 greeting, so anything advertising a
sleeping machine's reachability must not advertise for longer than 120 s.

**A lease is a socket, and a suspended machine's socket is gone.** Nothing re-binds it
automatically — a client session is re-established by the client reconnecting. A laptop that
sleeps for an hour holds no leases at all on waking, which is the correct state.

---

## 3. Network change, and what `anon` does across suspend/resume

This is the one section not answerable from this repository. Sources: the `tor(1)` manual and C
source from the official release tarball `https://dist.torproject.org/tor-0.4.9.12.tar.gz`; the
onion service spec at `https://spec.torproject.org/rend-spec/`; the control spec at
`https://spec.torproject.org/control-spec/commands.html`; and the `anon` fork at
`https://github.com/anyone-protocol/ator-protocol` (main is `AC_INIT([anon],[0.4.10.4-git])`).
`anon` is a fork of C tor and, in every mechanism below but §3.5, is byte-identical to upstream
apart from branding strings.

### 3.0 The connector holds no address state a network change can invalidate

Before the daemon: the connector side of this question is empty. A peer `endpoint` is a static
config string, and the only rule applied to it is a host-suffix test
(`crates/connector-config/src/peer.rs:217`):

```rust
pub fn is_onion_endpoint(url: &Url) -> bool {
    url.host_str().is_some_and(|host| {
        let host = host.to_ascii_lowercase();
        ONION_SUFFIXES.iter().any(|suffix| host.ends_with(suffix))
    })
}
```

The name is handed to the SOCKS proxy on every dial (`socks5h://`, so resolution happens at the
proxy — `crates/connector-peer-btp/src/ws.rs:36-42`), and `socks_proxy` points at a daemon on the
same machine. There is no cached IP, no resolver state, no re-binding, and no per-peer proxy key.
A machine's IP changing is invisible to the connector.

### 3.1 The address survives everything, because it is a function of one file

`https://spec.torproject.org/rend-spec/encoding-onion-addresses.html`:

```
onion_address = base32(PUBKEY | CHECKSUM | VERSION) + ".onion"
CHECKSUM = SHA3_256(".onion checksum" | PUBKEY | VERSION)[:2]
```

> where: PUBKEY is the 32 bytes ed25519 master pubkey (KP_hs_id) of the hidden service.

No IP, no port, no circuit state. The on-disk set in the C implementation
(`src/feature/hs/hs_service.c`) is `hs_ed25519_secret_key`, `hs_ed25519_public_key`, `hostname`,
optionally `authorized_clients/` and `ob_config`:

```c
static const char fname_keyfile_prefix[] = "hs_ed25519";
static const char fname_hostname[] = "hostname";
static const char address_tld[] = "onion";
```

**Only `hs_ed25519_secret_key` matters.** `load_service_keys()` (`hs_service.c:1082`) derives the
address from the key and calls `write_address_to_file()` (`:1039`), which rewrites `hostname`
every startup. Deleting `hostname` loses nothing; deleting the secret key loses the identity
permanently.

This is exactly what `docs/operators/onion-endpoint-bringup.md:181-195` already demands a
persisted volume for, and ADR 0070's Consequences state:

> **The hidden-service key is now operational state.** If `HiddenServiceDir` is not on a persisted
> volume, the node's address changes on every restart and every counterparty's config goes stale
> silently. This belongs in the operator runbook beside `state_dir`, and for the same reason.

So: suspend, resume, restart, reboot and IP change **cannot** change the address. A wiped
`HiddenServiceDir` can.

### 3.2 Suspend and resume: tor detects the clock jump and tears its own circuits down

This is the mechanism, and it is better than waiting for TCP. `src/core/mainloop/mainloop.c:2265`:

```c
#define NUM_JUMPED_SECONDS_BEFORE_WARN 100
#define NUM_IDLE_SECONDS_BEFORE_WARN 3600

if (seconds_elapsed < -NUM_JUMPED_SECONDS_BEFORE_WARN) {
  circuit_note_clock_jumped(seconds_elapsed, false);
} else if (seconds_elapsed >= NUM_JUMPED_SECONDS_BEFORE_WARN) {
  /* If the monotonic clock deviates from time(NULL) ... On some systems, this
   * means we have been suspended or sleeping. */
  const bool clock_jumped = abs(discrepancy) > 2;
  if (clock_jumped || seconds_elapsed >= NUM_IDLE_SECONDS_BEFORE_WARN) {
    circuit_note_clock_jumped(seconds_elapsed, ! clock_jumped);
  }
}
```

Suspend is named in the comment. The trigger is a wall-clock jump of **≥ 100 s** that the
monotonic clock disagrees with by more than 2 s. `circuit_note_clock_jumped()`
(`src/core/or/circuitbuild.c:1213`):

```c
tor_log(severity, LD_GENERAL,
        "Your system clock just jumped %"PRId64" seconds %s; "
        "assuming established circuits no longer work.", ...);
note_that_we_maybe_cant_complete_circuits();
circuit_mark_all_unused_circs();
circuit_mark_all_dirty_circs_as_unusable();
```

Service-side introduction circuits are never marked `timestamp_dirty`, so
`circuit_mark_all_unused_circs()` closes them: **an onion service's intro points are torn down by
the resume handler and must be rebuilt.** That is the cost of a suspend, and it is paid
automatically.

Keepalives exist but are not what recovers a laptop — `tor.1.txt`:

> **KeepalivePeriod** — To keep firewalls from expiring connections, send a padding keepalive cell
> every NUM seconds on open connections that are in use. (Default: 5 minutes)

with `run_connection_housekeeping()` (`mainloop.c:1178`) closing a connection wedged for
`KeepalivePeriod*10` (50 min at defaults). The clock-jump path fires long before that.

### 3.3 Dormant mode cannot engage on a machine publishing an onion service

This is the trap a laptop would otherwise fall into, and upstream has already closed it.
`tor.1.txt`, `== DORMANT MODE OPTIONS`:

> **DormantClientTimeout** N — If Tor spends this much time without any client activity, enter a
> dormant state where automatic circuits are not built, and directory information is not fetched.
> **Does not affect servers or onion services.** Must be at least 10 minutes. (Default: 24 hours)

Enforced in `check_network_participation_callback()` (`mainloop.c:1858`):

```c
if (server_mode(options)) goto found_activity;
if (! options->DormantTimeoutEnabled) goto found_activity;
/* If we're running an onion service, we can't become dormant. */
if (hs_service_get_num_services()) goto found_activity;
```

The fork carries this verbatim (`ator-protocol/src/core/mainloop/mainloop.c:1876-1880`), and
`anon.1.txt:1899-1904` carries the same man-page sentence. A **selling** Omarchy machine
(publishing a service) therefore never goes dormant. A machine that is only a **client** (a
SOCKS-only daemon, `local/anyone`'s `anonrc-a` shape) _can_ go dormant after 24 h idle — and
recovers on the next request, or on `SIGNAL ACTIVE`
(`https://spec.torproject.org/control-spec/commands.html`):

> ACTIVE — Tell Tor to stop being "dormant", as if it had received a user-initiated network request.

Suspend does not cause spurious dormancy: the clock-jump handler shifts the idle clock forward
(`netstatus.c`, `netstatus_note_clock_jumped`):

```c
void netstatus_note_clock_jumped(time_t seconds_diff) {
  time_t last_active = get_last_user_activity_time();
  if (last_active) reset_user_activity(last_active + seconds_diff);
}
```

Neither `local/onion/anonrc-*` nor `local/anyone/anonrc-*` sets any dormant-mode option, so both
run at the defaults described here.

### 3.4 IP change: noticed lazily, and there is nothing address-shaped to republish

There is no netlink or route monitor. The one mechanism is `client_check_address_changed()`
(`src/core/mainloop/connection.c:5106`), called from `connection_finished_connecting()` (`:5292`)
— i.e. **only when the next outbound connect succeeds**:

```c
/** Called when our attempt to connect() to a server has just succeeded.
 * This function checks if the interface address has changed (clients only) … */
if (!server_mode(get_options())) {
  client_check_address_changed(conn->s);
}
```

On a change it logs `"Our IP address has changed.  Rotating keys..."` and calls
`ip_address_changed(1)`. For a relay that marks the descriptor dirty; **for an onion service there
is nothing to republish, because a descriptor lists intro points, not the service's IP.** Until
the next successful connect, existing OR connections simply fail, which drives guard reconnection;
if every directory is unreachable, `directory_all_unreachable()` (`mainloop.c:1113`) logs _"Is your
network connection down?"_ and fires `DIR_ALL_UNREACHABLE`.

`SIGHUP` is a config reload, not a "renetwork" signal — `tor.1.txt` SIGNALS:

> The signal instructs Tor to reload its configuration (including closing and reopening logs), and
> kill and restart its helper processes if applicable.

So a laptop hopping Wi-Fi networks needs no intervention and no restart.

### 3.5 How long until it is reachable again

Four things must be re-established, in order:

1. **OR connections to guards** — rebuilt lazily.
2. **A "reasonably live" consensus.** The HS periodic event is gated (`hs_service.c:2193`):
   ```c
   if (!have_completed_a_circuit() || net_is_disabled() ||
       !networkstatus_get_reasonably_live_consensus(now, usable_consensus_flavor()))
     goto end;
   ```
   "Reasonably live" allows `REASONABLY_LIVE_TIME (24*60*60)` slack, so a cached consensus stays
   usable until roughly 27 h past its `valid_after`. Note the `have_completed_a_circuit()` gate:
   the clock-jump handler cleared it, so nothing HS-related runs until one circuit completes.
3. **Three introduction circuits.** `tor.1.txt`: _"**HiddenServiceNumIntroductionPoints** NUM :: …
   (Default: 3)"_. Each is a normal 3-hop build (`CircuitBuildTimeout` default 60 s, adaptive),
   with `INTRO_CIRC_RETRY_PERIOD (60*5)` and `MAX_INTRO_POINT_CIRCUIT_RETRIES 3` on failure.
4. **A descriptor upload.** `should_service_upload_descriptor()` (`hs_service.c:3438`) refuses
   until _all_ intro circuits are established. When intro points change,
   `service_desc_schedule_upload(desc, now, 1)` sets `next_upload_time = now` (`:2356`), so the
   upload is immediate — the 60–120 min timer is the steady-state republish, not a recovery delay.
   Up to 8 HSDirs receive it (`hsdir_n_replicas` 2 × `hsdir_spread_store` 4).

**Net: tens of seconds, typically under two minutes, with a fresh consensus.** With a consensus
expired past ~27 h, a consensus fetch precedes all of it — matching this repository's own
empirical note at `local/README.md:363`: _"Bootstrapping a circuit takes minutes rather than
seconds, so the sidecars' health gates wait on `Bootstrapped 100%`."_

**The one caller-visible cost.** The _old_ descriptor stays cached at the HSDirs for its published
lifetime and still lists the now-dead intro points. C tor publishes 180 minutes
(`src/feature/hs/hs_descriptor.h`, used at `hs_service.c:1941`):

```c
#define HS_DESC_DEFAULT_LIFETIME (3 * 60 * 60)
```

and the spec (`https://spec.torproject.org/rend-spec/hsdesc-outer.html`) permits 30–720 minutes,
resolving conflicts by `revision-counter`:

> If an HSDir receives a second descriptor for a key that it already has a descriptor for, it
> should retain and serve the descriptor with the **higher revision-counter**.

So a client that fetched the descriptor _before_ the machine slept will fail to introduce until it
refetches. In connector terms that is a failed dial — `T01`, `accumulated_cost: 0` — which is
§5's answer exactly. **No money is at risk; the cost is latency and a retry.**

### 3.6 A finding outside this ticket's scope: the `.anyone` rename changes the address bytes

The `anon` fork renamed the TLD **and the checksum domain-separation string** together:

| `anon` version                | `address_tld` (`hs_service.c`) | `HS_SERVICE_ADDR_CHECKSUM_PREFIX` (`hs_common.h`) |
| ----------------------------- | ------------------------------ | ------------------------------------------------- |
| v0.4.9.7 (2024-09-30)         | `"onion"`                      | `".onion checksum"`                               |
| v0.4.9.13                     | `"anon"`                       | `".anon checksum"`                                |
| v0.4.10.2 (2026-05-25) / main | `"anyone"`                     | `".anyone checksum"`                              |

Because the prefix is inside `SHA3_256(prefix | PUBKEY | VERSION)[:2]`, **the same
`hs_ed25519_secret_key` yields a different 56-character address in each era** — the base32 body
differs, not merely the suffix. And `write_address_to_file()` uses
`write_str_to_file_if_not_equal()`, so upgrading the daemon across an era boundary **silently
rewrites `hostname` with a new address while the key file is untouched**. The rename landed in
commit `6dac11b0` _"WIP: Initial pass of changing .anon -> .anyone"_ (2025-10-14); the GitHub
release notes document none of it.

This repository pins `ANON_VERSION=v0.4.10.2` in `local/anon-image/Dockerfile` and describes the
rename accurately as a routing incompatibility:

> v0.4.9.7 routes `.onion` and does not contain the string `.anyone`; v0.4.10.2 routes `.anyone`
> and contains ZERO occurrences of `.onion`, refusing an address spelled that way.

What is **not** recorded anywhere here is that it is also an _identity_ change: a persisted
`HiddenServiceDir` does not protect an address across an `anon` major bump, which is the one thing
ADR 0070's Consequences and the runbook both say a persisted volume is for. See "Possible
follow-ups" below.

Also relevant, and a documentation gap rather than a behaviour one: `https://docs.anyone.io/llms.txt`
(the complete docs index) has **no page** on hidden services, `HiddenServiceDir`, or the `.anyone`
service TLD. (`https://docs.anyone.io/dashboard/anyone-domains.md` describes `.anyone` as an
Unstoppable Domains NFT on Base for labelling relay-operator wallets — an unrelated product that
shares the string.)

---

## 4. Restart with no network

### 4.1 Settlement is dialled at build, with no retry, and a failure is `exit(1)`

`crates/connector-settlement-evm/src/lib.rs:129` does **three** live RPC calls before it will
return a backend:

1. `build_client` → `provider.get_chainid().await` (`:696`)
2. `registry.get_token_network(token_address).call().await` (`:139`)
3. `token.decimals().call().await` (`:152`), refusing if it disagrees with the configured
   `decimals`

`crates/connector-settlement-solana/src/lib.rs:182` does **four**:

1. `rpc.get_account(&program_id)` — and refuses a non-executable account
2. `rpc.get_account(&token_mint)` — and refuses one not owned by SPL Token
3. `Mint::unpack(...).decimals` vs configured `decimals`
4. `rpc.get_genesis_hash()` → `cluster_for_genesis_hash` (#1131)

Each is `?`-propagated to `build`, which returns `Err`, which `connector-bin/src/main.rs:31-34`
turns into:

```rust
Err(err) => { eprintln!("{err}"); std::process::exit(1); }
```

No retry, no backoff, no degraded mode, no "start and reconcile later". ADR 0041 names this
behaviour by reference (`:169`): "ADR 0009 makes an unreachable chain a refuse-to-start". This is
why both promotion gates classify failures — "only a config-shape error ... refuses the tag move;
any other failure is a warning" — because "a gate that cried wolf on a flaky RPC would be routed
around within a week."

### 4.2 It is conditional, and the condition is having declared settlement at all

A config with no `[settlement.*]` table starts fine with no network:
`crates/connector-bin/tests/refuses_to_start.rs:64` (`serves_traffic_with_a_valid_config`) boots
the real binary on a config that is a `[signer]` plus one `[[routes]]` and asserts `GET /ilp`
answers `200`. A node that declares no chain dials no chain.

But `state_dir` and settlement are coupled in the other direction: declaring **any** of
`[[peer_channels]]`, `[[pay_channels]]`, `[[client_channels]]` or a settlement backend without a
`state_dir` is a _load_ refusal by name
(`crates/connector-config/src/error.rs:503,646,904,920`), e.g.

> "a settlement backend is configured but 'state_dir' is not: this node resolves an ... Set a top-level state_dir to a directory this node can write"

So any machine that can actually be paid has both a `state_dir` and a chain dial at boot.

### 4.3 A _running_ node is far more tolerant of a missing chain than a _booting_ one

This asymmetry is the single most laptop-relevant fact in §4, and it is deliberate (#649).
`crates/connector-client-edge/src/channels.rs:273,287,297`:

```rust
pub const DEFAULT_LIVENESS_TTL: Duration = Duration::from_secs(60);
pub const DEFAULT_SERVE_STALE_UNTIL: Duration = Duration::from_secs(600);
pub const DEFAULT_MIN_REATTEMPT_INTERVAL: Duration = Duration::from_secs(2);
```

with the reasoning stated at `:276-285`:

> Ten minutes, and the number is chosen against the threat model rather than for comfort: what
> the expiry defends is a channel that settles on chain, and settling one takes a close, a
> challenge period and then a settle. A worst-case ten minutes of staleness — reached only while
> this connector's RPC endpoint is down, and logged at `warn` every time it is used — sits far
> inside that, while the alternative (refusing) turns somebody else's outage into this node's own
> refusal to serve paying clients.

Both are operator-tunable (`channel_liveness_ttl_secs`, `channel_serve_stale_secs`;
`crates/connector-config/src/config.rs:199,206`). So:

| Chain unreachable… | Behaviour                                                                                                                                                                                      |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| at **boot**        | `exit(1)` immediately, no retry (§4.1)                                                                                                                                                         |
| at **runtime**     | memoised channel facts believed for 60 s, then served stale for up to a further 600 s with a `warn` per use, then refusals — and it recovers by itself the moment RPC answers, with no restart |

A laptop resuming from suspend onto a not-yet-associated Wi-Fi link is in the second row, and
has roughly eleven minutes of grace before anything is refused. A laptop _booting_ onto the same
link is in the first row and does not start at all.

### 4.4 What that means for a laptop booting offline

A connector that can take money hard-exits 1 on a boot with no network. Under systemd
`Restart=always` / `RestartSec=5` (Omarchy house style, §6.2) this is a restart loop that
terminates the moment the link comes up — and it does **not** trip systemd's start rate limit:
the defaults are `DefaultStartLimitIntervalSec=10s` / `DefaultStartLimitBurst=5`, and five
restarts at `RestartSec=5` span 25 s.

Restart is otherwise clean. The outbound peer ledger re-arms itself
(`crates/connector-runtime/src/claim.rs:816-823`):

> A peer left with `pending` unacknowledged is _always_ re-armed with a freshly signed claim of
> the same nonce/cumulative amount ... resending an already-acknowledged claim costs nothing (the
> peer's own `accept_inbound` simply rejects a nonce that does not advance its watermark), so
> recovery needs no separate "was this acknowledged" record.

That is the property that makes a single-process restart safe, and it is precisely the property
two processes destroy (§1.3b): re-arming at the same nonce is safe only when exactly one process
decides what that nonce is.

Reload is a restart by design — ADR 0009:

> Reload is a restart. Since the configuration value is immutable, changing the file means
> restarting the process.

---

## 5. Reject semantics — what a sleeping seller costs its callers

**Confirmed: nothing beyond a failed dial, in every case but §2.1.**

ADR 0051's class-only table:

| code  | situation                               | why class is enough                                          |
| ----- | --------------------------------------- | ------------------------------------------------------------ |
| `R00` | the packet expired                      | relative by construction — the sender's own clock decided it |
| `T01` | the app or the next peer is unreachable | retry later                                                  |

Every path a sleeping machine produces lands in that table:

- **Buyer dials a sleeping seller directly.** TCP refused / timed out. Nothing was delivered, no
  claim was presented to anyone, nothing is journalled. Cost: zero. This is not even a reject —
  it is a client-side connection failure.
- **An intermediary forwards to a sleeping peer.** The peer dial fails and the packet answers
  `T01` naming the peer and the endpoint. `crates/connector-peer-btp/src/dial.rs:19-25` states
  the rule: "It surfaces as an ordinary dial failure with the peer id and the attempted endpoint
  named, and packets routed to that peer reject **`T01`** — never `T00`, and never a silent drop."
  The HTTP carriage's twin (`crates/connector-peer-http/src/dial.rs:595`) builds the same reject
  with `accumulated_cost: 0`:

  ```rust
  fn dial_failed(peer_id: &str, endpoint: &Url) -> PacketResponse {
      PacketResponse::Reject(Reject {
          code: RejectCode::t01_peer_unreachable(),
          message: format!("peer '{peer_id}' unreachable at {endpoint}"),
          accumulated_cost: 0,
          ...
      })
  }
  ```

- **And the buyer is refunded for that hop.** The intermediary already admitted the buyer's
  covering claim (ADR 0028 prices a forwarded route at the client edge), and #1012 gives it back
  in full — `roll_back_uncarried_forward` restores the channel to `watermark.nonce` /
  `watermark.cumulative_amount`, i.e. "to what it was immediately before this request". Not the
  price minus the fee: the whole admitted advance.
- **Session-registry delivery to a sleeping client.** `T01` with `accumulated_cost: 0` (§2.3).
- **A packet that outlived the nap.** `R00`, `"prepare has expired"`, with the sender's own clock
  having decided it.

The `accumulated_cost` a reject carries is **discovery information, not a charge** — ADR 0011 is
the record, and its own reasoning settles the question directly:

> **Understating a fee is unprofitable.** Because ADR 0004 moves value on fulfilment, a hop that
> advertises a low fee to attract traffic and then rejects the real packet **earns nothing** and
> has spent its own bandwidth. Honesty needs no enforcement.

A hop that rejects earns nothing. That is the protocol-level statement of what §1's rollback is
the implementation of.

`F99` is the only reject in ADR 0051's binding table whose meaning is "stop trusting that
counterparty", and nothing a sleeping machine does produces it. A sleeping seller is
indistinguishable, to the protocol, from a busy one — which is exactly the property a machine
that is off for most of the day needs.

**So intermittence is a quality-of-service problem, not a correctness one** — with the single
exception in §2.1, which is one packet's price, bounded, and is ADR 0064's stated position rather
than an oversight.

---

## 6. Omarchy's own machinery

Read off `/usr/share/omarchy` on a live install; units are shipped by `omarchy-settings` 4.0.3-1
and installed to `/usr/lib/systemd/user/`.

### 6.1 Suspend and resume

`/usr/share/omarchy/default/systemd/user/omarchy-sleep-lock.service`:

```
[Unit]
Description=Lock Omarchy before suspend
After=dbus.socket wayland-session-waitenv.service
Requires=dbus.socket
PartOf=graphical-session.target
ConditionEnvironment=OMARCHY_PATH
ConditionEnvironment=WAYLAND_DISPLAY

[Service]
Type=simple
ExecStart=/usr/bin/omarchy-system-sleep-monitor
Restart=always
RestartSec=2

[Install]
WantedBy=graphical-session.target
```

`/usr/bin/omarchy-system-sleep-monitor` ends in:

```bash
exec systemd-inhibit \
  --what=sleep \
  --mode=delay \
  --who=Omarchy \
  --why="Lock screen before suspend" \
  "$sleep_monitor" --inhibited
```

and under that inhibitor watches `org.freedesktop.login1.Manager.PrepareForSleep` with
`dbus-monitor --system`. On `boolean true` it runs `omarchy-system-sleep-lock` **synchronously**,
then returns — **the process exits, which closes the inhibitor fd, which is how the delay is
released**. `Restart=always` / `RestartSec=2` brings a fresh monitor (and a fresh inhibitor) back
~2 s after the user manager is thawed on resume. There is no `PrepareForSleep(false)` handler.

It is a **delay** lock, not a block. `/etc/systemd/logind.conf.d/20-inhibit-delay.conf`
(`omarchy-settings`) widens the window and says so:

```
# A delay inhibitor is a timer, not a promise: logind suspends anyway once the
# window expires, locked or not.
[Login]
InhibitDelayMaxSec=15
```

`omarchy-system-sleep-lock` budgets against `InhibitDelayMaxUSec` read live over `busctl`, caps
at `budget_cap_ms=12000`, and notifies critically on overrun — its header: _"Overrunning the
budget is the failure this whole path exists to prevent: logind stops honouring the inhibitor and
suspends mid-lock."_

**There is no suspend or resume hook event for third parties.** `/usr/bin/omarchy-hook` in full:

```bash
HOOK_PATH="$HOME/.config/omarchy/hooks/$1"
HOOK_DIR="$HOOK_PATH.d"
shift
if [[ -f $HOOK_PATH ]]; then bash "$HOOK_PATH" "$@" || echo "Hook failed: $HOOK_PATH"; fi
if [[ -d $HOOK_DIR ]]; then
  for hook in "$HOOK_DIR"/*; do
    [[ -f $hook ]] || continue
    [[ $hook == *.sample ]] && continue
    bash "$hook" "$@" || echo "Hook failed: $hook"
  done
fi
```

The complete set of firing sites, i.e. the six named events: `post-boot`
(`default/hypr/autostart.lua:13`), `battery-low`, `font-set`, `pre-refresh-pacman`, `theme-set`,
`post-update`. No suspend, resume, wake, sleep, network or login event exists. A failing hook
prints `Hook failed:` and never aborts the caller.

The system-level `/usr/lib/systemd/system-sleep/` path is available and Omarchy uses it — its
`unmount-fuse` script is the only resume-side code the distribution ships, and its comment names
the constraint anything else there would meet:

> Run in background — user.slice is still frozen at this point, so a synchronous restart would
> block the thaw for up to 90 seconds.

There is no hypridle/hyprlock at all (`pacman -Q hypridle hyprlock` → not found); idle and lock
are Quickshell plugins under `/usr/share/omarchy/shell/plugins/`.

### 6.2 House style for a long-lived user daemon

Shipped pattern, seen on `omarchy-crash-watch`, `omarchy-fcitx5`, `omarchy-sleep-lock`,
`omarchy-tailscale-receive`: `Type=simple`, `Restart=always`, `RestartSec=2` or `5`,
`WantedBy=graphical-session.target`, `After=`+`PartOf=graphical-session.target`, and a
`ConditionEnvironment=WAYLAND_DISPLAY` to keep it from starting outside a graphical session.
`omarchy-fcitx5.service` states the doctrine on start-gating:

> `After=` is ordering only — it does not stop anything from starting this unit while the target
> is inactive... Skip the start instead; the unit stays enabled and starts for real at graphical
> login.

**No shipped unit references `network.target` or `network-online.target`,** and Omarchy actively
masks the wait (`install/config/enable-services.sh:9-12`):

```
# Don't let network-online.target hold up graphical.target waiting for
# DHCP/Wi-Fi association. Nothing in the session needs to block on the network.
systemctl mask NetworkManager-wait-online.service
```

Omarchy ships **no NetworkManager dispatcher script** and nothing reacts to a link going up or
down. Combined with §4.1, an Omarchy user unit for a settling connector would boot before the
network exists and rely on `Restart=` to converge.

### 6.3 Toggles

`/usr/bin/omarchy-toggle` — presence of an empty file is the state:

```bash
FLAG="$HOME/.local/state/omarchy/toggles/$FLAG_NAME"
enable() { mkdir -p "$(dirname "$FLAG")"; touch "$FLAG"; }
disable() { rm -f "$FLAG"; }
```

tested by `omarchy-toggle-enabled`: `[[ -f "$HOME/.local/state/omarchy/toggles/$1" ]]`. Naming is
negative (`crash-capture-off`, `suspend-off`). The unit gates on it rather than being
`disable`d — `omarchy-crash-watch.service`:

```
# Set by omarchy-toggle-crash-capture. Checked here so a disabled watcher stays
# disabled across logins without the unit having to be disabled.
ConditionPathExists=!%h/.local/state/omarchy/toggles/crash-capture-off
```

and `omarchy-toggle-crash-capture` states the invariant:

> The unit checks the same flag with ConditionPathExists, so stopping and starting only settles
> this session; the next login reads the flag itself.

A second namespace, `toggles/hypr/<name>.lua`, holds a **copy of a Lua fragment** rather than an
empty marker (`omarchy-hyprland-toggle`), sourced wholesale by `default/hypr/toggles.lua`.

### 6.4 What Omarchy does about "must not run twice" — directly relevant to §1

**Units rely on systemd unit identity and nothing else.** No shipped unit uses `PIDFile=`,
`flock`, `RefuseManualStart`, or a lock in `ExecStartPre`. One enabled unit is one process, and
collision cases are handled with _conditions_ — `omarchy-fcitx5.service`'s comment on a second
instance ("fcitx5 exits 0 when it detects another instance owning its bus name, and a clean exit
still leaves the user with no input method") is answered with `ConditionEnvironment`, not a lock.

**`flock` is the house style for scripts**, always advisory, fd-based, in
`${XDG_RUNTIME_DIR:-/tmp}`, and almost always `-n` with "if held, exit 0". The canonical example
is `/usr/bin/omarchy-update-lock`:

```bash
lock_path="$lock_dir/omarchy-update.lock"
exec {OMARCHY_UPDATE_LOCK_FD}>"$lock_path"
flock -n "$OMARCHY_UPDATE_LOCK_FD" || { echo "An Omarchy update is already running."; exit 1; }
exec "$@"
```

with the fd exported so children inherit it, and a `held` subcommand that verifies by identity
(`readlink -f /proc/$$/fd/$OMARCHY_UPDATE_LOCK_FD`) rather than by file existence. A _oneshot
unit_ probes that same lock cross-process — `omarchy-migrate-notify:12-13`:

```bash
! flock -n "$lock" true 2>/dev/null   # "flock -n only fails here when the lock is held,
                                      #  since the file is ours"
```

Others in the same shape: `omarchy-hyprland-monitor-watch` (`flock -n 9 || exit 0`),
`omarchy-brightness-display` ("Drop overlapping brightness key events so concurrent invocations
do not race"), `omarchy-system-lock`, `omarchy-theme-set` (blocking `flock 9`),
`omarchy-agent-usage-*` (Python `fcntl.flock`). Run-once markers use
`(set -o noclobber; : >"$marker")` (`omarchy-done:34`).

`pgrep -x` / `pgrep -f` appears widely but only for "is this _other_ app running" decisions, never
as a self-exclusion guard.

**So Omarchy's answer to "must not run twice" is: be a unit; and if a script can be invoked
concurrently, take a non-blocking `flock` on an `$XDG_RUNTIME_DIR` file and exit 0 when you lose.**
Neither half covers a _directory_ that two differently-configured processes might both open,
which is exactly the connector's §1 hazard.

### 6.5 Everything else Omarchy ships

For completeness, the full shipped user-unit set: `bt-agent.service`,
`omarchy-crash-watch.service`, `omarchy-fcitx5.service`, `omarchy-migrate-notify.service`
(`Type=oneshot`; a `.path` unit watching the migrations directory was **deliberately removed** —
"pacman writes that directory during every update... Checking once per login is the only trigger
that cannot collide with a running update"), `omarchy-recover-internal-monitor.service`
(`Type=oneshot`, `WantedBy=graphical-session-pre.target`), `omarchy-sleep-lock.service`,
`omarchy-speaker-tuning.service` (copied per-user into `$XDG_CONFIG_HOME/systemd/user/` by
`omarchy-audio-tuning`, not installed system-wide), `omarchy-tailscale-receive.service`. No
`.timer`, `.path`, `.target` or `.socket` is shipped anywhere.

---

## Possible follow-ups

Stated as findings, not as work. Each is a fact this note established that is not currently
recorded anywhere in the repository.

1. **`state_dir` has no single-instance guard, and the only one that ever existed was deleted
   with the `announce` verb (§1.4).** ADR 0030 states the hazard as prose and its guard is gone;
   `grep -rn "via_own_routing" crates/` is empty. ADR 0030 also describes only the outbound half
   — §1.3 finds three failure directions, two of which (free inbound service, colliding EVM
   settlement nonces) it does not name. On a server this is exotic; on a personal machine it is
   ordinary.

2. **`FileJournal::append` can tear a line under two writers, and one torn line makes the node
   permanently unbootable (§1.2a).** `writeln!` on an unbuffered `File` is two `write(2)` calls;
   `read_all`'s `collect()` turns the first `Err(corrupt())` into `JournalUnreplayable` →
   `exit(1)`. `append_batch` is already immune (one `write_all`). This is a property of the
   encoder, independent of whether anything is ever done about §1.

3. **An `anon` major upgrade changes a node's onion address even with a persisted
   `HiddenServiceDir` (§3.6).** The checksum domain-separation string moved with the TLD
   (`".onion checksum"` → `".anon checksum"` → `".anyone checksum"`), so the same
   `hs_ed25519_secret_key` produces a different base32 body per era, and `hostname` is silently
   rewritten on first start of the new binary. `local/anon-image/Dockerfile` and ADR 0070's
   amendment both describe the rename as a _routing_ incompatibility; neither records that it is
   also an _identity_ one. This bears directly on ADR 0070's Consequences and on
   `docs/operators/onion-endpoint-bringup.md:181-195`, which present a persisted volume as the
   thing that keeps an address stable.

4. **The boot/runtime asymmetry on an unreachable chain is undocumented outside the code
   (§4.3).** A running node tolerates a dead RPC for 60 s + 600 s of stale-serving and recovers by
   itself; a booting node exits 1 immediately. Both are deliberate and neither is wrong, but the
   pair is not written down anywhere an operator would find it.

## What this note does _not_ invalidate

- **ADR 0030's hazard statement is correct**, just incomplete (§1.3) and describing a guard that
  no longer exists (§1.4).
- **ADR 0064 stands** and is the reason §2.1 is a designed cost rather than a bug.
- **ADR 0070's "an onion endpoint is a host, not a carriage" needs nothing** for a personal
  machine. §3.0 confirms the connector holds no address state a network change can invalidate,
  which is the property the ADR's decision 3 buys.
- **The map's premise that "the connector is expected to need nothing"** survives §2, §3 and §5
  intact. §1 is the exception, and §1 is a _lifecycle and packaging_ problem — one process per
  `state_dir` — before it is a connector-code problem.
