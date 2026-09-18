# What a bare Omarchy machine needs to become a paying client

> **Research note, not a decision record.** Answers
> [#3](https://github.com/toon-protocol/omarchy/issues/3), a child of the wayfinder map
> [#1](https://github.com/toon-protocol/omarchy/issues/1). It records facts and quotes
> against primary sources — the shipped Omarchy installer on disk, `toon-protocol/toon-client`'s
> own tree and its npm manifest, the four seller repositories' committed deploy bundles, and the
> live free self-descriptions the three devnet boxes answered on the day it was written. It
> recommends nothing. Probed **2026-09-18**; every price and every reachability claim below is a
> reading, and a reading is stale the moment an operator edits a bundle
> ([`docs/devnet-pricing.md`](../devnet-pricing.md) makes that point at length and it applies here
> unchanged).
>
> Nothing in this note spent money, opened a channel or submitted a transaction. Every live figure
> came from a free, unauthenticated `GET` — the surface
> [ADR 0022](../adr/0022-a-connector-answers-it-does-not-announce.md) and
> [ADR 0050](../adr/0050-a-connectors-url-resolves-to-its-self-description.md) exist to provide.

## The short answer

**Nothing is missing from the machine. One thing is missing from the network.**

A fresh Omarchy 4.0 install already has the entire runtime: Node and npm ship through `mise`, on
PATH, user-owned, no sudo. `npm install @toon-protocol/client` and `npx toon` work with zero setup.
The client needs exactly one fact it cannot derive — a connector's URL — plus a key it generates
itself and a payment channel the user opens on chain with their own gas.

But **the published client cannot pay any live connector today.** `@toon-protocol/client@3.0.0`
vendors the post-[ADR 0069](../adr/0069-the-execution-condition-leaves-the-wire.md) wire; all three
live devnet boxes run `rust-2026.08.28.1`, which predates it. The two are not compatible, and the
automated pin bumps that would fix it have been failing daily since 2026-09-11. §7 has the
evidence. This falsifies the map's day-one premise as of today, and it is a fleet-operations
problem, not a connector one.

---

## 1. Runtime

### What the client requires

`@toon-protocol/client@3.0.0`, published `2026-09-07T15:43:05Z`, `dist-tags.latest = 3.0.0`
(npm registry). Its manifest:

```json
"type": "module",
"engines": { "node": ">=22" },
"bin": { "toon": "dist/cli/main.js" },
"dependencies": {
  "viem": "^2.0.0", "@scure/base": "^1.2.0", "@scure/bip32": "^1.4.0",
  "@scure/bip39": "^1.3.0", "@noble/curves": "^2.0.0", "@noble/hashes": "^1.4.0",
  "@noble/ciphers": "^2.0.0"
},
"optionalDependencies": { "ws": "^8.0.0", "socks": "^2.8.10", "undici": "^7.16.0" }
```

Four facts follow:

- **Node ≥ 22.** Stated in `packages/client/package.json` and in the root `package.json`
  (`"node": ">=22", "pnpm": ">=8"`), pinned again in `/devbox.json` (`"nodejs": {"version": "22"}`)
  and `.github/workflows/ci.yml` (`node-version: '22'`). There is no `.nvmrc` and no
  `.node-version` in the tree. Nothing in the code parses `process.version` — the only runtime
  guard is `packages/client/src/keys/keystore-node.ts`'s `assertNode()`, which checks Node-vs-browser,
  not a version. So `>=22` is enforced by the package manager, not by the program.
- **ESM only.** `"type": "module"`, and the `exports` map carries an `import` condition with no
  `require` condition. `require('@toon-protocol/client')` fails.
- **No native dependencies.** Every dependency is pure JavaScript. No compiler, no `node-gyp`, no
  prebuild step. The only non-JS tool the client ever shells out to is `unzip` (or PowerShell's
  `Expand-Archive`), and only on the managed-daemon path (§2).
- **The CLI ships in the package.** `bin: { "toon": ... }`, so `npx toon` needs no global install.
  `toon-client`'s own `README.md`: _"The CLI ships in the same package. `npx toon` runs it without
  a global install."_

`undici` is pinned to `^7` deliberately — `packages/client/src/transport/socks.ts` and
`docs/hidden-service.md`: _"Node's global `fetch` hands a userland dispatcher a handler defined by
Node's own bundled undici; undici 8 dropped the shape Node 22 passes, and this package's
`engines.node` is `>=22`. `^8` is not an upgrade here."_

### What Omarchy ships

**Node is present on a fresh install, through `mise`, never through pacman.**

The decisive file is `/usr/share/omarchy/install/user/mise-work.sh`, invoked unconditionally from
`/usr/share/omarchy/install/user/all.sh` line 5 (`run_logged "$OMARCHY_INSTALL/user/mise-work.sh"`):

```bash
case ${OMARCHY_SETUP_CONTEXT:-runtime} in
  iso-chroot)       NODE_PACKAGE_DIR=/opt/packages ;;
  provision-owner)  NODE_PACKAGE_DIR=/var/lib/omarchy/provisioning/packages ;;
  *)                NODE_PACKAGE_DIR="" ;;
esac
```

and its two terminal branches:

```bash
    NODE_VERSION=$(basename "$NODE_TARBALL" | sed 's/node-v\(.*\)-linux-x64.tar.gz/\1/')
    NODE_INSTALL_DIR="$HOME/.local/share/mise/installs/node/$NODE_VERSION"
    mkdir -p "$NODE_INSTALL_DIR"
    tar -xzf "$NODE_TARBALL" --strip-components=1 -C "$NODE_INSTALL_DIR"
    mise use -g node@"$NODE_VERSION"
  fi
else
  mise use -g node@latest
fi
```

So an ISO install unpacks a bundled `node-v*-linux-x64.tar.gz`; a network install takes
`node@latest`. Official Node tarballs carry npm, and they do here — `ls
~/.local/share/mise/installs/node/22.23.2/bin` → `corepack node npm npx`.

Supporting facts, all from the shipped tree:

- `mise-bin` is line 80 of `/usr/share/omarchy/install/omarchy-base.packages`. Neither that file
  nor `omarchy-other.packages` contains `nodejs`, `nodejs-lts-*`, `npm`, `bun`, `pnpm` or `yarn`.
  A full grep of the installer tree for those names returns only `mise` references plus one
  unrelated comment.
- PATH is wired in `/usr/share/omarchy/default/bash/env-bootstrap`, which appends
  `$HOME/.local/share/mise/shims` and `$HOME/.local/bin`; `/usr/share/omarchy/default/bash/init`
  runs `eval "$(mise activate bash)"`; and `install/config/ssh-command-path.sh` extends the PAM
  path so non-shell SSH commands see the shims too.
- `npm config get prefix` resolves to the mise install dir under `$HOME`, so `npm i -g` needs **no
  sudo**. Default registry, no `~/.npmrc`, no `/etc/npmrc`.

**The agent CLIs come the same way**, which is the mechanism the ticket asked about.
`/usr/share/omarchy/install/user/mise.sh`:

```bash
omarchy-mise-install codex
omarchy-mise-install claude
...
omarchy-mise-install npm:playwright playwright
omarchy-cmd-missing cursor-agent && omarchy-mise-install cursor-agent
omarchy-mise-install npm:@kitlangton/ghui ghui
if omarchy-cmd-missing muse; then
  omarchy-mise-install "http:muse[url=https://api.meta.ai/muse-launcher.sh,bin=muse,...]" muse
fi
```

`omarchy-mise-install` writes a lazy wrapper into `~/.local/bin/<cmd>`:

```bash
#!/bin/bash
export MISE_MINIMUM_RELEASE_AGE=0
mise use -g --quiet "npm:playwright" || exit 1
exec mise x "npm:playwright" -- "playwright" "$@"
```

Three of those entries are `npm:` backend packages, so mise's Node is the intended runtime for
npm-distributed tools. Cursor and Muse are `http:` backends and install no Node of their own —
`/etc/mise/conf.d/omarchy.toml` (owned by `omarchy-settings 4.0.3-1`) exists specifically to keep
Cursor's bundled Node from shadowing the user's: _"Expose only its launcher so the bundled Node
cannot shadow the user's Node."_ The relevant migrations are `1788577553.sh` (Cursor CLI) and
`1788724825.sh` (Muse Code); **no migration installs Node**.

### The four caveats

1. **The Node is per-user, under `$HOME`.** Root, a system service, or a `useradd` that never ran
   Omarchy's user provisioning has no node and no npm. `/usr/bin/npm` does not exist on Omarchy at
   all. This matters for #7's "where is the spend limit enforced" and for any systemd unit: a
   `WantedBy=graphical-session.target` **user** unit sees the shims; a system unit does not.
2. **The version is not pinned by Omarchy on the network path.** `mise use -g node@latest` means a
   machine provisioned today and one provisioned six months ago get different majors. The
   `engines: {"node": ">=22"}` range is the only guard, and it is advisory.
3. **No anonymity tooling ships.** `grep -rniE '\b(tor|anon|onion|socks|ngrok|cloudflared)\b'`
   across `/usr/share/omarchy/{install,bin,config}` returns **zero matches**; `pacman -Qs '^tor$'`
   finds nothing. This is the same finding the map recorded while charting, confirmed on disk.
4. **No credential or key store.** Omarchy installs `gnome-keyring`/`libsecret` and configures the
   keyring never to lock (`install/user/default-keyring.sh`); `~/.local/state/omarchy/toggles/` is
   boolean flags. There is nowhere Omarchy-shaped for a key today — #6's question, and the
   client's own answer is §3's `~/.toon/keystore.json`.

---

## 2. The daemon

**The CLI runs `anon` itself. The library never does, and that split is a recorded decision in the
client's own repo.** `packages/client/src/cli/anon-daemon.ts`, header:

> The managed `anon` daemon — the CLI's, and only the CLI's (ADR 0001).
>
> Reaching a hidden-service connector takes a running Anyone Protocol daemon and a SOCKS port.
> `@toon-protocol/client` never provides one: a library that an application embeds must not
> download and execute a binary at runtime. The `toon` command may, because running it is already
> a decision to run our executable — so this module lives under `src/cli/`, which is a separate
> build entry that a library consumer never loads.
>
> The release is pinned and checksummed, and the gate fails **closed**: an asset whose hash is
> unknown is not downloaded at all, rather than downloaded and trusted.

### It downloads; it does not look on PATH

There is no config key naming an `anon` path and no PATH lookup. The whole of discovery is
"is it in the version-keyed cache? else fetch it":

```ts
export const ANON_VERSION = 'v0.4.10.2';
const RELEASE_BASE = `https://github.com/anyone-protocol/ator-protocol/releases/download/${ANON_VERSION}`;

export function defaultCacheDir(): string {
  const xdg = process.env['XDG_CACHE_HOME'];
  return xdg ? path.join(xdg, 'toon', 'anon', ANON_VERSION)
             : path.join(os.homedir(), '.toon', 'anon', ANON_VERSION);
}

export async function ensureAnonBinary(cacheDir, log = () => undefined): Promise<string> {
  const exe = os.platform() === 'win32' ? 'anon.exe' : 'anon';
  const anonPath = path.join(cacheDir, exe);
  if (fs.existsSync(anonPath)) return anonPath;
```

Five platforms are pinned by sha256. The linux-x64 figure —
`9c6498b8d27de54d78842a1b854979a605f9c140ccc34f2b4c267bf094eaeb17` — is **byte-identical** to the
one this repository records in [`local/anon-image/Dockerfile`](../../local/anon-image/Dockerfile)
(`ARG ANON_SHA256=9c6498b8…`, with the comment _"the same figure toon-client records in
`packages/client/src/cli/anon-daemon.ts`"_). Both sides pin `v0.4.10.2`, the release that writes
`.anyone` — the rename [ADR 0070's amendment](../adr/0070-an-onion-address-is-a-host-not-a-carriage.md)
and [`local/anyone/README.md`](../../local/anyone/README.md) document. An unpinned platform is
refused rather than trusted: _"No pinned anon binary for `${platform}-${arch}` … Run your own
daemon and pass `--socks` instead."_

### `AgreeToTerms` is accepted on the user's behalf, silently

```ts
/**
 * A SOCKS-only torrc.
 *
 * `AgreeToTerms 1` is REQUIRED: without it `anon` exits immediately, and the
 * failure reads as "the daemon died" rather than "you did not accept the terms".
 */
export function renderTorrc(cacheDir: string, socksPort: number): string {
  return [
    'AgreeToTerms 1',
    `DataDirectory ${path.join(cacheDir, 'data')}`,
    `SOCKSPort 127.0.0.1:${socksPort}`,
    'SOCKSPolicy accept *',
    `GeoIPFile ${path.join(cacheDir, 'geoip')}`,
    `GeoIPv6File ${path.join(cacheDir, 'geoip6')}`,
    'Log notice stdout',
    'RunAsDaemon 0',
    '',
  ].join('\n');
}
```

Written to `<cacheDir>/torrc` mode `0600`, data dir `0o700`. There is **no prompt**: running `toon`
against a `.anyone` connector accepts Anyone Protocol's terms for you. That is the same trap this
repository documents from the other side —
[`docs/operators/onion-endpoint-bringup.md`](../operators/onion-endpoint-bringup.md) §2 and
[`local/anyone/anonrc-a`](../../local/anyone/anonrc-a): _"WITHOUT THIS THE DAEMON DOES NOT START …
It is the most common reason a first bring-up leaves a container that exited immediately beside a
connector that looks healthy and is unreachable."_ An Omarchy-level facility that runs a daemon on
a user's behalf inherits that acceptance decision; it is a policy question, not a technical one.

### The port is ephemeral, and the lifetime is the command

```ts
const port = await freePort(); // OS-chosen loopback port, not 9050
const child = cp.spawn(anonPath, ['-f', torrcPath], { stdio: ['ignore', 'pipe', 'pipe'] });
return { socksProxy: `socks5h://127.0.0.1:${port}`, port, stop };
```

Bootstrap wait is `options.bootstrapTimeoutMs ?? 90_000`, polled by TCP connect every 250 ms,
aborting early if the child exits (_"anon exited before opening its SOCKS port. Last log line: …"_).
`packages/client/src/cli/main.ts` ends its `finally` block with `context?.stopManagedAnon()` —
_"leaving an `anon` process behind would hold the terminal and outlive the command."_

**So there is no long-lived daemon today, on either side.** The CLI starts one per command and kills
it. A machine-level facility that wanted a persistent circuit would be building something neither
repository has, which is exactly the "lifecycle shape" the map lists as not yet specified.

### It is automatic, and only under one condition

`packages/client/src/cli/context.ts`:

```ts
  private async withProxy(settings: CliSettings): Promise<CliSettings> {
    if (settings.socksProxy !== undefined) return settings;
    if (!isHiddenServiceUrl(settings.connector)) return settings;
    log(`${settings.connector} is a hidden service; starting a local anon daemon`);
    this.managedAnon = await start(log);
    return { ...settings, socksProxy: this.managedAnon.socksProxy };
  }
```

`docs/cli.md`: _"A clearnet connector starts nothing and downloads nothing. … The library never
starts a daemon — only this command does."_

### The library refuses, in both directions

`packages/client/src/client/config.ts`, `resolveSocksProxy`:

```ts
if (socksProxy === undefined) {
  if (!connectorIsHiddenService) return undefined;
  throw new ConfigError(
    `connector ${JSON.stringify(connector)} is a hidden service, which is reachable ` +
      'only through a SOCKS5h proxy. Set `socksProxy` to a running Anyone Protocol ' +
      '`anon` daemon (e.g. "socks5h://127.0.0.1:9050"), or use the `toon` CLI, which ' +
      'can start one for you.'
  );
}
if (!connectorIsHiddenService) {
  throw new ConfigError(
    `socksProxy is set, but connector ${JSON.stringify(connector)} is a clearnet ` +
      'address, so nothing would ride the proxy. …'
  );
}
```

`socks-url.ts` enforces `socks5h://` by name, with this repository's own reasoning restated:
_"The 'h' makes the proxy resolve the hostname, so a .anyone address never leaks into a local DNS
query."_ Its header says _"Mirrors the connector's `transport/socks-url.ts`."_ Default port when
unspecified is **1080**.

### One cross-repo mismatch: the client is narrower than the connector

`packages/client/src/transport/hs-hostname.ts`:

```ts
export const HS_HOSTNAME_REGEX = /^[a-z2-7]+\.anyone$/;
```

with `.onion` refused **by name**:

```ts
if (typeof hostname === 'string' && /\.onion$/.test(hostname)) {
  throw new Error(
    `"${hostname}" is a Tor hidden service, which this client does not dial. ` +
      'Packets are routed through the Anyone Protocol `anon` daemon, whose ' +
      'hidden services live under the .anyone TLD.'
  );
}
```

The connector accepts **both** suffixes — `connector_config::is_onion_endpoint` matches `.onion`
or `.anyone` ([`crates/connector-config/src/peer.rs:217`](../../crates/connector-config/src/peer.rs),
ADR 0070 as amended by #1284). The client accepts only `.anyone`. This is not a defect on either
side — the connector dials whatever its own operator's daemon speaks, and the client pins one
daemon it downloads itself — but it means **a connector reachable at a `.onion` host is unpayable
by this client**, and an Omarchy machine that ran plain Tor rather than `anon` would be a client
the published library refuses. Worth stating because ADR 0070's preconditions say _"plain Tor works
identically: the connector's whole surface is a `socks5h://` URL and a hidden-service host."_ That
is true of the connector and **not** of `toon-client`.

Finally, `packages/client/src/transport/socks.ts` makes the SOCKS path an _optional_ dependency
with a legible failure:

```ts
`Reaching a hidden service needs the optional dependency "${name}", which is not installed. ` +
  `Run \`npm install ${name}\`, or drop socksProxy (${socksProxy}) and dial a clearnet connector.`;
```

---

## 3. Keys and channel

### What must exist

**Both a key and, for any priced route, an already-open and already-funded payment channel on a
chain the seller settles on.** Neither is optional, and neither is something the seller can do for
you.

The only structurally required config field is the URL — `packages/client/src/client/types.ts`:

```ts
export interface ToonClientConfig {
  /**
   * The connector's client-edge base URL. …
   * This is the whole of bootstrapping. There is no discovery, no relay and no
   * peer list: one free `GET` on this URL returns every fact needed to transact
   * with the node (`GET /ilp`, connector ADR 0050).
   */
  connector: string;
  mnemonic?: string;
  evmPrivateKey?: string | Uint8Array;
  solanaSecretKey?: Uint8Array | string;
```

but key material is required in practice — `packages/client/src/client/config.ts`:

```ts
if (identity.evm === undefined && identity.solana === undefined) {
  throw new ConfigError(
    'No key material: supply a `mnemonic` (which derives both an EVM and a ' +
      'Solana key), or a raw `evmPrivateKey` / `solanaSecretKey`. A claim is ' +
      "authorised by its signature against the channel's on-chain counterparty " +
      'and by nothing else (connector ADR 0052), so there is no unauthenticated ' +
      'way to pay.'
  );
}
```

That citation is exact. [ADR 0052](../adr/0052-permissionless-payment-is-guaranteed-and-a-claim-is-what-authorises.md):
_"An unaffiliated buyer registers with the chain, not with the operator. That is a public fact this
connector reads for itself, which is what makes anonymity a first-class path rather than a
concession."_

### The key file, and who makes it

The CLI generates one. `toon init` writes an encrypted BIP-39 keystore at `~/.toon/keystore.json`
(scrypt + AES-256-GCM, mode `0600`). `packages/client/src/cli/commands/init.ts`:

```ts
if (keystoreExists(path)) {
  throw new CliConfigError(
    `a keystore already exists at ${path}`,
    'Move it aside first — overwriting it would destroy the only copy of its phrase, ' +
      'and any channel collateral that phrase controls.'
  );
}
```

Resolution order is `TOON_MNEMONIC` → keystore file → error
(`packages/client/src/cli/context.ts`, `resolveKeyMaterial`):

```ts
throw new CliConfigError(
  `no keys: ${MNEMONIC_ENV} is unset and there is no keystore at ${settings.keystorePath}`,
  "Run 'toon init' to create one, or set TOON_MNEMONIC."
);
```

`--import` reads a phrase from **stdin only**, and `docs/cli.md` is categorical: _"There is no
`--mnemonic` flag and there will not be one — a flag value is written to shell history and is
readable in `ps` by every other user on the machine for as long as the process runs."_

Two commands run without any key at all, on a throwaway identity (`KeyMaterial = { kind:
'ephemeral' }`) — _"nothing is signed with it, no chain is touched, and it is discarded when the
process exits. That is what lets `toon describe` work on a machine that has never run `toon
init`."_ Those are `describe` and `price`.

One phrase derives both chains (`docs/getting-started.md`): _"an EVM key at `m/44'/60'/0'/0/0` and
a Solana key at `m/44'/501'/0'/0'`."_ **Which chain is used is the connector's choice, not the
caller's** — the default is _"the first chain in the connector's own `settlements[]` for which this
client holds a key."_

### The channel is the user's own transaction

`packages/client/src/cli/commands/channel.ts` header:

> Every subcommand here is _your_ transaction on _your_ chain account. Nothing about it goes
> through the connector: a connector has no endpoint that opens a channel, it discovers yours by
> reading the chain (connector ADR 0052).

This repository says the same from the seller's side, for Solana
([`docs/solana-deployment.md`](../solana-deployment.md)):

> **From a counterparty that is not a connector** — `rig`, `toon-client`, a wallet — the deposit is
> submitted directly against the deployed program under that participant's own key. There is no
> operation on any node, on either settlement backend, or on the settlement port that does it for
> them, and adding one is not possible without changing the program.

The library defaults to `autoOpenChannel: true`; **the CLI forces it off** and says why
(`packages/client/src/cli/context.ts`, and `docs/cli.md`): _"`toon send` would submit chain
transactions, spend gas and lock collateral as a side effect of asking for one HTTP request. So the
CLI turns it off, and lets the refusal explain itself."_ The refusal, from
`packages/client/src/client/channel-facade.ts`:

```ts
throw new ChannelNotOpenError(
  `No payment channel is open with ${config.connector} on ${terms.chain}, ` +
    'and `autoOpenChannel` is off, so nothing will be opened on your behalf. ' +
    'Open one explicitly first — it locks collateral on chain and spends gas, ' +
    'which is exactly why it is not done as a side effect of sending.'
);
```

`code: 'CHANNEL_NOT_OPEN'`, which `main.ts` maps to **exit 4**.

### Which chain

Either. All three live boxes settle on **both** Base Sepolia (`evm:84532`, mock USDC
`0x49beE1Bca5d15Fb0963117923403F9498119a9Ce`, 6 decimals) and **Solana devnet** (program
`2aEVJ8koKD8LTZrLRSGtAtU7LBt4e7QjjCgf1kzQ7Rip`, mint
`34eSxY7qxQ4GzyhDJ8GpUcTz1WWzruGbJbR8q6TtxfQU`, 6 decimals) — read live, §5. anytoon settles on
**Ethereum mainnet**, in ANYONE at 18 decimals, which is real money.

### A route priced at zero needs no channel

`toon-client`'s `docs/channels.md`: _"Such a route runs no claim gate: an unpaid request to it is
simply routed. So this client does not open a channel, sign a claim or touch a chain to use one,
and `send()` returns a result whose `claim` is absent rather than zero-valued."_ The relay publishes
exactly one such route, `g.toon.relay.ephemeral` at price `0` (§5). It is the cheapest end-to-end
reachability test in existence — and it is **not** a paying client, so it does not satisfy the
ticket's question, only bounds it.

### The channel store is a third precondition, and it is the laptop hazard

`packages/client/src/client/config.ts`, when `channelStore` is unset:

```ts
console.warn(
  '[toon] No `channelStore` configured — the claim watermark will be held in ' +
    'memory and lost when this process exits. A restarted client re-signs at ' +
    'nonces the connector has already banked, and the connector refuses every ' +
    "one of them while the channel's collateral stays locked. …"
);
```

That is the payer-side twin of the hazard the map quotes from
[ADR 0030](../adr/0030-an-operator-announces-a-node-the-node-still-does-not.md), and it is worse
than the map's framing: the map worries about two _connector_ processes over one `state_dir`; this
is one _client_ process losing a watermark across a restart, which on a laptop is every sleep-wake
cycle if the store is not on disk. The CLI always uses `~/.toon/channels.json`
(`defaultChannelStorePath()`); the **library defaults to memory and only warns**. `local/anyone`'s
own harness has the same rule and states it as a teardown requirement
([`local/anyone/run.sh`](../../local/anyone/run.sh)): _"a store left behind would sign the next
run's claims against a channel that no longer exists, at a nonce the fresh connector has never
seen — and the rehearsal would fail as a replay, for a reason that looks like a bug."_

---

## 4. Reaching a seller

**A URL. That is the entire requirement, and it must arrive out of band.** Confirmed on both sides.

### The client's side

`toon-client`'s `README.md`:

> - **Not a Nostr client, and there is no relay.** Versions before 1.0 published events to relays
>   and bootstrapped from announcements. All of it is gone: a node is a URL you configure, and its
>   self-description is the whole of bootstrapping.
> - **Reading a node's facts is free.** Its addresses, endpoints, sealing key, settlement terms and
>   route prices come from one unauthenticated `GET`.

`toon-client`'s `CONTEXT.md`: _"**Self-description**: … **It replaces peer discovery entirely.**"_

`docs/api.md`: _"`connector` | required | Base URL or `…/ilp` … There is no discovery and no peer
list — one URL is the whole of bootstrapping."_

Everything else is read off the node and **not** taken from the client's own presets. The
destination is optional (`docs/cli.md`): _"Omit it and the request goes to the address the node
published for itself in its `GET /ilp`, so a connector URL is the whole of the configuration —
there is no route string to copy out of a document and get wrong."_ And
`packages/client/src/presets.ts` refuses to be an authority: _"**These are conveniences, never
authorities.** … Two declarations of one fact is how a mainnet node comes to be described as
devnet, so nothing here is ever consulted in preference to the document a node answers with."_
That is ND-07 restated in the client's own words.

### The connector's side

[`docs/protocol/self-description-spec.md`](../protocol/self-description-spec.md) §1.2 lists exactly
what one free `GET` carries, including the two facts a payer cannot proceed without:

| fact                                                                                             | why a stranger needs it                                         |
| ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- |
| **edge identity** — the key a packet is sealed to                                                | without it a packet cannot be sealed, so it cannot be delivered |
| per chain: chain id, settlement address, token network and its registry, token address, decimals | what a buyer needs to **open a channel**                        |

So a URL really does bootstrap the channel as well as the packet.

### ND-15/#1083 — confirmed, and the shape of the gap is narrower than the ticket implies

[ADR 0046](../adr/0046-the-kind-10032-announce-is-removed-a-connector-needs-no-relay.md) is
explicit: _"**What replaces it: nothing, inside the connector.**"_ ND-15 is the rule that would
close half the gap and it is **not built**:

> **Not built:** the unsealed reject's URL (#1083, ND-15) and route descriptions (ADR 0044).
>
> … The half #1026 actually lacks is the _discovery_: how a client learns the terminating
> connector's URL without asking a hop, which ND-14 forbids answering. That is ND-15/#1083. **Until
> it is built, a forwarded route is reachable only by a client that already knows the terminating
> node's URL out of band.**

Issue #1083 is **CLOSED**, and its title reads as if shipped — _"An unsealed termination reject
carries the terminating connector's URL (ADR 0054 / #1071) — closes #1026"_. The spec's own
falsifier says otherwise and is the authority: `crates/connector-runtime/src/connector.rs` matching
`fn unsealed_termination_reject\([^)]*,` — _"The reject builder takes a message and nothing else;
the URL has to be handed to it."_ **Treat #1083 as closed-but-unbuilt.** That is a discrepancy
worth a ticket in its own right (§9).

Two refinements the ticket did not ask for but that bear on §6:

- The gap is **specific to forwarded routes.** For a route the addressed node _terminates_, one URL
  genuinely suffices — the identity in its own `GET /ilp` is the one to seal to. Every purchase in
  §6 is of that kind.
- For a **forwarded** route the client needs a second URL, which it takes as an explicit parameter.
  `SendOptions.sealTo` — _"Needed only when paying a route the addressed node **forwards**, since a
  payload must be sealed to the connector that terminates it and no hop may name that key on its
  behalf."_ That is ND-13 and ND-16 implemented as an API argument, i.e. the out-of-band
  requirement made into a function signature. `toon-client`'s `docs/devnet.md`:

  ```ts
  const answer = await client.send(
    'g.toon.relay.gas',
    { body: 'hello' },
    {
      sealTo: 'https://proxy.gas.devnet.toonprotocol.dev',
    }
  );
  ```

The only fallback to "a URL arrives out of band" is a hard-coded default with a loud warning
(`packages/client/src/cli/context.ts`):

```ts
connector = DEVNET.store.url;
warnings.push(
  `toon: no connector given, using the devnet store node ${DEVNET.store.url}.\n` +
    `      Set --connector or ${CONNECTOR_ENV} to talk to another node.`
);
```

---

## 5. The live list

Read **2026-09-18** by `curl` on each box's free `GET /ilp`. Prices are in base units of 6-decimal
USDC; `1000` is 0.001 USDC and `1` is 1 µUSDC ([`docs/devnet-pricing.md`](../devnet-pricing.md),
[ADR 0010](../adr/0010-flat-per-packet-fee-and-minimum-delivery.md)).

| Connector   | ILP address(es)                          | Live price           | Chain                             | Reachable from a laptop      | Payable today |
| ----------- | ---------------------------------------- | -------------------- | --------------------------------- | ---------------------------- | ------------- |
| **relay**   | `g.toon.relay`, `g.toon.relay.ephemeral` | `1` / `0`            | Base Sepolia **or** Solana devnet | Yes — clearnet TLS           | **No** (§7)   |
| **store**   | `g.toon.store`                           | `1000 + 10/KiB`      | same two                          | Yes — clearnet TLS           | **No** (§7)   |
| **gas**     | `g.toon.gas`, `g.toon.relay.gas`         | `1000`               | same two                          | Yes — clearnet TLS           | **No** (§7)   |
| **anytoon** | `g.anyone.credentials`                   | `1e16` (0.01 ANYONE) | **Ethereum mainnet**, 18 dp       | **No — address unpublished** | No            |

### relay — `https://proxy.relay.devnet.toonprotocol.dev/ilp`

```json
{
  "ilpAddresses": ["g.toon.relay", "g.toon.relay.ephemeral"],
  "httpEndpoint": "https://proxy.relay.devnet.toonprotocol.dev/ilp",
  "btpEndpoint": "wss://proxy.relay.devnet.toonprotocol.dev/ilp/btp",
  "peerCarriages": [],
  "edgeIdentity": {
    "keyId": "connector-signer",
    "publicKey": "0x04915d29908235be4b53f8f23cd7ac72c88c99be3bcca876dadf5c1a449433b8f900bed774f10f2360fc01d27e92e0138f018e54c9558d2deb05efd9688d032bdd"
  },
  "settlements": [
    {
      "chain": "evm:84532",
      "settlementAddress": "0x3f43d923a611bcb2d0bfb5d6ee2c3ac3efeaf308",
      "tokenNetworkRegistry": "0x0c41d9d424d6b075a3cea1068a694f7847a8cca5",
      "tokenNetwork": "0xe9e05dfecfe165266c88d73e61d483612651952a",
      "tokenAddress": "0x49bee1bca5d15fb0963117923403f9498119a9ce",
      "decimals": 6
    },
    {
      "chain": "solana",
      "settlementAddress": "GzvGVjq3dnNM79MpWRvYCvVcAgPWzDdYisMwGxHF4u9F",
      "programId": "2aEVJ8koKD8LTZrLRSGtAtU7LBt4e7QjjCgf1kzQ7Rip",
      "tokenAddress": "34eSxY7qxQ4GzyhDJ8GpUcTz1WWzruGbJbR8q6TtxfQU",
      "decimals": 6
    }
  ],
  "routes": [
    { "prefix": "g.toon.relay", "price": "1" },
    { "prefix": "g.toon.relay.ephemeral", "price": "0" },
    { "prefix": "g.toon.relay.gas", "price": "1001" },
    { "prefix": "g.toon.relay.store", "price": "1001", "pricePerKib": "10" }
  ],
  "supportedVersions": [1],
  "defaultVersion": 1
}
```

`toon-protocol/relay:deploy/connector.toml` pins the paid route to one carriage:

```toml
[[routes]]
prefix      = "g.toon.relay"
handler_url = "http://relay:3100/write"
price       = 1
transport   = "btp"
```

So **`g.toon.relay` is BTP-only** — a one-shot HTTP buyer is refused with `TRANSPORT_REQUIRED` /
`F02` (issue #701; `toon-client`'s `docs/troubleshooting.md`: _"The devnet relay route is BTP-only:
`--transport btp`, or `transport: 'btp'`."_). `g.toon.relay.ephemeral` is price `0` and takes the
default `both`. The two forwarded rows (`.gas`, `.store`, each the far side's price plus a 1-unit
fee) are runtime peer routes and appear in no committed file, exactly as `docs/devnet-pricing.md`
describes.

### store — `https://proxy.ario.devnet.toonprotocol.dev/ilp`

```json
{"ilpAddresses":["g.toon.store"],
 "peerCarriages":["btp"],
 "edgeIdentity":{"keyId":"connector-signer","publicKey":"0x04499cdd71c7c3eab8d9b35f88ec9cde29018461e4bef86389004abcd7cfa1108a96cdc183d6ded3bca7cc5aa56afb1158193cf70c9e45011745f965e098838988"},
 "settlements":[{"chain":"evm:84532","settlementAddress":"0x6b6c2dacf7ac1f1273f72bef2e6084f9ee6d3bff",…},
                {"chain":"solana","settlementAddress":"W6yK72j365eK7t4Qj5An1AaYtUEJcJK7TBPvGeDk1LV",…}],
 "routes":[{"prefix":"g.toon.relay.store","price":"1000","pricePerKib":"10"},
           {"prefix":"g.toon.store","price":"1000","pricePerKib":"10"}]}
```

**`g.toon.ario` is retired and answers nothing.** Probed live:

```
$ curl "https://proxy.ario.devnet.toonprotocol.dev/ilp/routes/price?destination=g.toon.ario"
no route this connector serves matches 'g.toon.ario'
```

The ticket, the map and #8 all still name `g.toon.ario`. `ario` has been a box label and a DNS
name only since store#109 (2026-08-27); the address is `g.toon.store`
([`docs/devnet-pricing.md`](../devnet-pricing.md), "Retired names").

One live limit the self-description cannot show: `GET https://dvm.devnet.toonprotocol.dev/health`
reports `"paidUploads":"off"` and `"freeTierMaxBytes":107520`. **You can pay the store, and the app
behind it refuses anything over ~105 KiB.** That is an app fact, correctly absent from the node's
self-description under ND-08.

### gas — `https://proxy.gas.devnet.toonprotocol.dev/ilp`

```json
{
  "ilpAddresses": ["g.toon.gas", "g.toon.relay.gas"],
  "peerCarriages": ["btp"],
  "edgeIdentity": {
    "keyId": "connector-signer",
    "publicKey": "0x04de501ea1139c07adba8cb725ef0b7549ee5ac5999740368393b35494fa5443f3deeef200b27c75e9e849801da33dde76706064fc7e82a27ae12188dfdf8e797e"
  },
  "routes": [
    { "prefix": "g.toon.gas", "price": "1000" },
    { "prefix": "g.toon.relay", "price": "2" },
    { "prefix": "g.toon.relay.gas", "price": "1000" }
  ]
}
```

`g.toon.gas.quote` is in `gas-station:deploy/connector.toml.template` and is **not live** — the
deployed render predates the quote/execute split. `/describe` on the app reports
`handlerKinds:[5096,5098]` and both `solana:devnet` and `evm:84532`.

### anytoon — cannot be paid, because the address is not published

`toon-protocol/anytoon:config/connector.toml`:

```toml
[node]
addresses = ["g.anyone.credentials"]
http_endpoint = "http://placeholderplaceholderplaceholderplaceholderplaceholdera.anyone/ilp"

[[routes]]
prefix = "g.anyone.credentials"
handler_url = "http://claim-minter:8080/"
price = 10000000000000000

[[routes]]
prefix = "g.anyone.credentials.keys"
handler_url = "http://issuer:3000/v1/keys/"
price = 0

[settlement.evm]
rpc_url = "https://ethereum-rpc.publicnode.com"
contract_address = "0x61d31e7Fd9a57A0611e29Bd7eB162f15AC8B3427"
token_address = "0xFeAc2Eae96899709a43E252B6B92971D32F9C0F9"
decimals = 18
```

Header: _"ETHEREUM MAINNET. THIS IS REAL MONEY."_ The host above is a committed placeholder;
`scripts/hs-address.sh` renders the real one into `config/.rendered/connector.toml`, which
`.gitignore` excludes. The README says _"Tell buyers the address `make up` prints"_ — and that
address appears nowhere in the repo, in any issue, or in any workflow run. There is no `deploy/`
bundle and no deploy workflow, so there is no evidence any instance is running and, if one is, a
buyer has nothing to dial.

The buyer recipe is nonetheless the closest thing that exists to a spec for this ticket's question.
`config/anonrc-client` is a committed client-only `anonrc` with the same `AgreeToTerms 1` /
`SocksPort 0.0.0.0:9050` shape as [`local/anyone/anonrc-a`](../../local/anyone/anonrc-a);
`buyer/package.json` wants **Node ≥ 24** and `@toon-protocol/client` `^3.0.0`; and the invocation is
two environment variables:

```bash
TOON_SOCKS_PROXY=socks5h://127.0.0.1:9050 \
TOON_CONNECTOR=http://<56-char-address>.anyone \
  npm run buy
```

`buyer/src/hidden-service.ts` leaves `proxyRpc` **on** — _"reading chain state on the clearnet would
broadcast the buyer's settlement address either side of every paid request"_ — which is the
opposite of the one opt-out [`local/anyone/pay.mjs`](../../local/anyone/pay.mjs) takes, and for the
documented reason that anvil is already private there.

Two flags on anytoon's own documentation: its README §4 _Routes_ table states the price as `10000`,
which is the **local anvil** figure from `config/connector.local.toml`, not the mainnet
`10000000000000000` — a reader working from that table is off by 10¹². And its ADR 0003 (accepted
2026-09-11) says the price _"is a placeholder, and it stays one deliberately… Set a price that is
commercially real for you before taking payment from anyone."_

### On reachability generally

None of relay, store or gas has a `socks_proxy` key, an `anon` sidecar, or any hidden-service
tooling; all three are clearnet TLS. anytoon is the only `.anyone` node and publishes no clearnet
endpoint and no host port. So **today the clearnet three are reachable from any laptop and the
onion one is reachable by nobody**, which inverts the shape the map assumed.

---

## 6. The shortest path, marked

Target: **the store, `g.toon.store`, on Base Sepolia** — the client's own default
(`DEVNET.store.url`), a route the addressed node terminates (so no `sealTo`), and the only
non-BTP-pinned paid route on the live fleet.

Each step is marked **[A]** automatable (a script or a systemd user unit could do it unattended),
**[H]** human-only (needs a decision, a secret, or a resource a machine cannot obtain), or
**[A*]** mechanically automatable but deliberately gated by the tool.

| #   | Step                      | Command                                                        | Mark                                         |
| --- | ------------------------- | -------------------------------------------------------------- | -------------------------------------------- |
| 0   | Runtime                   | _(nothing)_ — `node`/`npm`/`npx` are already on PATH via mise  | **[A]** — Omarchy's installer already did it |
| 1   | Reach the seller's facts  | `npx toon describe https://proxy.ario.devnet.toonprotocol.dev` | **[A]**                                      |
| 2   | Learn the price           | `npx toon price g.toon.store`                                  | **[A]**                                      |
| 3   | Create the key            | `npx toon init`                                                | **[H]**                                      |
| 4   | Record the phrase         | _(write it down)_                                              | **[H]**                                      |
| 5   | Get the settlement token  | `npx toon faucet`                                              | **[A]**                                      |
| 6   | Get native gas            | _(see below)_                                                  | **[H]**                                      |
| 7   | Choose a consistent RPC   | `export TOON_RPC_URL=…`                                        | **[H]**                                      |
| 8   | Open and fund the channel | `npx toon channel open --deposit 100000`                       | **[A\*]**                                    |
| 9   | The paid request          | `npx toon send --body 'hello'`                                 | **[A]**                                      |

As a transcript:

```bash
# 0. nothing — Omarchy's mise already put node/npm/npx on PATH.       [A]

# 1. free, no wallet, no channel, no account.                          [A]
npx toon describe https://proxy.ario.devnet.toonprotocol.dev
export TOON_CONNECTOR=https://proxy.ario.devnet.toonprotocol.dev

# 2. the price is asked for, never derived.                            [A]
npx toon price g.toon.store        # -> 1000 base units + 10/KiB

# 3. write an encrypted keystore at ~/.toon/keystore.json (0600).      [H]
npx toon init

# 4. record the recovery phrase.                                       [H]

# 5. mint devnet mock USDC to the address just created.                [A]
npx toon faucet

# 6. native gas — NOT from the faucet. Base Sepolia ETH, or on Solana: [H]
#    solana airdrop 1 <address> --url https://api.devnet.solana.com

# 7. a read-after-write-consistent RPC (the default is known-bad).     [H]
export TOON_RPC_URL=https://base-sepolia-rpc.publicnode.com

# 8. YOUR transaction, YOUR gas, collateral locked on chain.           [A*]
npx toon channel open --deposit 100000     # 0.10 USDC

# 9. one paid request: sealed payload, signed claim, 32-byte fulfilment. [A]
npx toon send --body 'hello'               # ~1010 base units
```

### Why each mark is what it is

**Step 0 [A].** §1. This is the finding the whole ticket turns on: Omarchy's installer runs
`mise use -g node@…` unconditionally during user provisioning, so the runtime precondition is
already satisfied on a machine nobody has touched.

**Steps 1–2 [A].** `describe` and `price` run on a throwaway ephemeral identity, sign nothing and
touch no chain. Verified live today against all three boxes (§5) — these work right now, wire split
notwithstanding, because they are plain HTTP GETs rather than packets.

**Step 3 [H].** `toon init` prompts for a keystore password when stdin is a TTY. It is automatable
only by setting `TOON_KEYSTORE_PASSWORD` or `--password-file`
(`docs/troubleshooting.md`: _"The keystore password has nowhere to come from and stdin is not a
terminal"_), which moves the secret into the environment or onto disk in the clear — a policy
decision belonging to #6, not a mechanical one.

**Step 4 [H].** Irreducibly. _"Overwriting it would destroy the only copy of its phrase, and any
channel collateral that phrase controls."_

**Step 5 [A].** The faucet is live and both legs ready. `GET https://faucet.devnet.toonprotocol.dev/api/info`,
read today:

```json
"baseSepolia":{"enabled":true,"route":"/api/base-sepolia/request","ready":true,
  "chainId":84532,"drips":{"usdc":"1000"},
  "tokenAddress":"0x49beE1Bca5d15Fb0963117923403F9498119a9Ce",
  "rpcUrl":"https://sepolia.base.org","mintMode":"ungated-mint"}
```

`mintMode: ungated-mint` is why it cannot run dry — `packages/faucet/src/index.js`: _"Because
`mint()` is ungated the faucet key holds no USDC — it coins fresh tokens on demand."_ The drip is
`ethers.parseUnits('1000', 6)` = 1000 whole USDC, cooldown 24 h per address. At 1010 base units per
store write that is ~10⁶ requests — funding the _token_ is not the constraint.

**Step 6 [H], and it is the real wall.** The faucet's ETH leg is best-effort and gated on the
faucet key holding a surplus (`packages/faucet/src/base-sepolia.js`: _"best-effort drips `ethAmount`
wei of gas ETH ONLY when the faucet key's balance sits above (ethReserve + ethAmount)"_); today
`/api/info` reports `"faucetBalances":{"eth":null,…}`. `toon-client`'s `docs/devnet.md` is blunter:
_"**Neither leg funds gas.** Both drip the settlement token and nothing else, so a wallet needs
Base Sepolia ETH, or devnet SOL, from elsewhere."_ And `docs/troubleshooting.md` names the failure:
_"**`ChannelFundingError` on `channel open`.** The wallet holds the settlement token but no native
gas. Opening is a transaction; paying for requests is not."_

The Solana escape hatch needs the `solana` CLI, which **Omarchy does not ship** (§1 caveat 3 is
about tor; this is a separate absence — no `solana` in either package list). So step 6 is human-only
on both legs, and on Solana it is human-only _plus an install_.

This is precisely the boundary the map put out of scope ("Funding, key custody, recovery — the
wallet"). It is worth naming exactly where the boundary bites: **between step 5 and step 8, for one
transaction's worth of native gas, once per machine.**

**Step 7 [H].** `docs/troubleshooting.md`: _"**The open reverts with `InvalidChannelState()`
(`0xf806e9d9`).** The RPC is not read-after-write consistent … The cure is a consistent RPC."_ The
package's own preset for Base Sepolia is `https://sepolia.base.org`, which is the one named as
known-bad; the three live boxes all use `https://base-sepolia-rpc.publicnode.com`. A machine could
hard-code a known-good endpoint, but choosing one is a trust decision.

**Step 8 [A\*].** Mechanically a scripted transaction. Deliberately gated: the CLI sets
`autoOpenChannel: false` — _"a command never opens a channel as a side effect"_ — and the library's
default is the opposite. An Omarchy facility that automated this would be overriding a decision
`toon-client` made on purpose; that is #7's business, and the lever is real, because funding the
channel only to the cap is one of the four enforcement points #7 lists.

**Step 9 [A].** One request, no prompt.

### Two variants worth recording

- **The free variant, which is not a paying client.** Steps 0–1, then
  `npx toon send g.toon.relay.ephemeral --body 'hello'` against the relay. No key, no channel, no
  chain, price `0`. It proves reachability end to end and proves nothing about payment.
- **The relay's paid route needs BTP.** `npx toon send g.toon.relay --transport btp --body 'hello'`,
  per `transport = "btp"` in its bundle and the client's own troubleshooting entry. The client
  defaults to `--transport auto`.

---

## 7. The blocker: the published client cannot pay the live fleet

This is the load-bearing finding, and it is verifiable in four reads.

1. **`@toon-protocol/client@3.0.0` vendors the post-ADR-0069 wire.**
   `packages/client/src/wire/vectors/wire-vectors.provenance.json`:

   ```json
   {
     "sourceRepo": "toon-protocol/connector",
     "sourcePath": "vectors/wire-vectors.json",
     "connectorCommit": "fe996af255d6a6b9456ef4e6e88f6c18313b72ae",
     "connectorCommitDate": "2026-09-04T01:04:29Z",
     "connectorCommitSubject": "The execution condition leaves the wire (issue #1269, ADR 0069) (#1271)",
     "schemaVersion": 5
   }
   ```

2. **Its changelog says so, and names the symptom.** `packages/client/CHANGELOG.md`, `## 3.0.0`:

   > `connector@main` no longer carries an `executionCondition` on a PREPARE: a one-byte `greeting`
   > flag sits where the 32-byte field was, and a packet still carrying the condition is refused at
   > the wire — misleadingly, as `invalid packet type byte`. The vendored wire vectors move to
   > schema 5 and this client's codec follows them.

3. **All three live boxes run an image that predates it.** `relay:deploy/docker-compose.yml`,
   `store:deploy/docker-compose.yml` and `gas-station:deploy/docker-compose.yml` each pin
   `ghcr.io/toon-protocol/connector:rust-2026.08.28.1`. `gh release list` gives two handles:
   `2026.08.28.1` (2026-08-28) and `2026.09.11.1` (2026-09-11). `fe996af2` is dated 2026-09-04 —
   after the pinned release, before the unadopted one.

4. **The adoption is stuck, not merely pending.** Each of the three repositories has an unmerged
   `deploy/adopt-connector-2026.09.11.1` branch, and each repository's scheduled _Adopt connector
   release_ run is failing — on `toon-protocol/relay`, twice on 2026-09-18 alone. The failure is
   environmental, not a test:

   ```
   HANDLE: 2026.09.11.1
   Switched to a new branch 'deploy/adopt-connector-2026.09.11.1'
   pull request create failed: GraphQL: GitHub Actions is not permitted to create or approve pull requests
   ```

So a bare Omarchy machine that runs `npm install @toon-protocol/client` today installs `3.0.0` and
gets `invalid packet type byte` at step 9 — a message this repository already documents as reading
_"like a transport fault and is not one"_
([`local/anyone/README.md`](../../local/anyone/README.md), which reproduces it over plain loopback
with no circuit at all).

Three consequences worth stating plainly:

- **Steps 0–7 of §6 all work today.** The break is at the packet, not at the runtime, the keys, the
  faucet or the channel. `describe`, `price` and `channel open` are unaffected — they are HTTP GETs
  and chain transactions, not TOON packets.
- **`npm install @toon-protocol/client@2.2.0` is the pinned workaround**, since `2.2.0` predates the
  schema-5 bump and already carries `socksProxy`, the managed daemon and proxied RPC. This note
  records that as a fact about the version line, not as a recommendation.
- **[ADR 0068](../adr/0068-a-node-repository-pins-the-connector-nothing-here-moves-a-tag-onto-a-box.md)
  is working as designed and the mechanism around it is not.** Nothing in this repository moves a
  tag onto those boxes, correctly; but the node repositories' own bump automation cannot open its
  own PR, so the pin sits in a branch nobody merges. This is a fleet-operations finding, **not** a
  connector change, and per the map it is to be flagged loudly rather than started.

---

## 8. Against the map's three falsifiers

**1. "It needs Omarchy to exist."** Steps 0, 3, 4, 6 and 7 of §6 are where an OS could be
load-bearing, and only one of them is currently unique to Omarchy. Step 0 is already solved by
Omarchy and is therefore _not_ a differentiator — it is a step Omarchy removes, which the map's own
falsifier calls distribution rather than integration (_"would this purchase be just as easy without
Omarchy — `npm i` and a command?"_, #8 Q5 — on this evidence, **yes**). The genuinely
OS-shaped items are steps 3/4 (a key store the machine owns, where Omarchy today has none) and
step 8's gate (a spend limit below the agent, #7). Step 6 is out of scope by the map's own
ruling.

**2. "It is useful at one machine."** Conditionally true and currently false. Three clearnet
sellers exist, are reachable from any laptop, and are priced in fractions of a cent on a chain a
faucet mints freely. One machine can pay them the moment §7's pin lands. Today it cannot pay any of
them, so day one is blocked on a fleet pin bump rather than on anything this map decides.

**3. "It is not `anytoon` or `provider` with a wallpaper."** The client edge is not the connector
edge. Everything in §§1–6 concerns a machine that _pays_, holds no channel it did not open itself,
signs no claim for anyone else, and runs no `anon` daemon of its own beyond the per-command one the
CLI spawns. `anytoon` is a seller behind a hidden service; this is a buyer on the clearnet. They do
not overlap — and the one place they nearly do, `anytoon/buyer/`, is the only committed artefact in
the TOON estate that resembles what this ticket describes, and it is 40 lines of environment
variables rather than a facility.

---

## 9. Findings for other tickets

**Corrections to sibling tickets, all factual:**

- **`g.toon.ario` is dead.** #3 itself, #8 and the map all name it. The store's address is
  `g.toon.store`; `g.toon.ario` answers `404`, probed today. Retired by store#109, 2026-08-27.
- **#8's candidate list needs the gas box's real shape and the store's app limit.** The gas box
  terminates `g.toon.gas` at `1000`, not `g.toon.gas.quote` (that route is committed but not
  deployed); the store's app has `paidUploads: off` and a `freeTierMaxBytes` of 107520, so a paid
  store write over ~105 KiB fails at the handler, not at the connector.
- **#8's "does it work from a laptop" question is answered, and inverted.** The three clearnet
  boxes work from a laptop; `anytoon` cannot be reached by anyone because its `.anyone` address is
  generated into a gitignored file and published nowhere.
- **#6 has an answer to build on, not from scratch.** `toon-client` already defines the key's
  location, format and permissions: `~/.toon/keystore.json`, BIP-39 under scrypt + AES-256-GCM,
  mode `0600`, one phrase deriving both an EVM key at `m/44'/60'/0'/0/0` and a Solana key at
  `m/44'/501'/0'/0'`, with `TOON_MNEMONIC` taking precedence. The question is whether Omarchy
  should relocate or share it, not what it is.
- **#7 has four named enforcement points and one of them is already the CLI's default.**
  `autoOpenChannel: false` means the channel's collateral _is_ the cap today, set once at
  `channel open --deposit`, and raising it is a second explicit `channel deposit`. That is the
  "fund it only to the cap" option in #7 Q1, already implemented, already the default, and
  already per-machine rather than per-agent.

**Candidates for new tickets — none of them in this repository's own work:**

1. **The fleet pin bump is stuck (§7).** Three node repositories cannot open their own adoption
   PRs (`GitHub Actions is not permitted to create or approve pull requests`), so all three boxes
   run a wire the published client cannot speak. Highest-value, lowest-effort item found; belongs
   in `toon-protocol/{relay,store,gas-station}`, not here.
2. **#1083 is closed but its spec says unbuilt.** `docs/protocol/self-description-spec.md` lists
   ND-15 under "Not built" and carries a falsifier saying the reject builder still takes a message
   and nothing else, while issue #1083 is CLOSED with a title that reads as shipped. Either the
   spec is stale or the issue closed without the build. One of the two is wrong and a reader
   trusting the wrong one draws the wrong conclusion about discovery.
3. **The client refuses `.onion`; the connector accepts it.** `is_onion_endpoint` matches both
   suffixes (ADR 0070 as amended); `hs-hostname.ts` matches `.anyone` only and refuses `.onion` by
   name. ADR 0070's preconditions say _"plain Tor works identically"_ — true of the connector,
   false of `toon-client`. Not a bug in either, but an undocumented asymmetry that makes a Tor-hosted
   connector unreachable by the reference client.
4. **anytoon publishes no address.** Its README instructs the operator to _"tell buyers the address
   `make up` prints"_ and there is no mechanism, no bundle and no record that does so. Belongs in
   `toon-protocol/anytoon`.
5. **anytoon's README states the mainnet price as the anvil figure** — `10000` where the config says
   `10000000000000000`, a factor of 10¹². Belongs in `toon-protocol/anytoon`.

---

## Sources

**This repository:** [`local/anyone/`](../../local/anyone/) (`README.md`, `run.sh`, `pay.mjs`,
`connector.toml`, `compose.yml`, `anonrc-a`), [`local/anon-image/Dockerfile`](../../local/anon-image/Dockerfile),
[`docs/operators/onion-endpoint-bringup.md`](../operators/onion-endpoint-bringup.md),
[`docs/protocol/client-edge-spec.md`](../protocol/client-edge-spec.md),
[`docs/protocol/self-description-spec.md`](../protocol/self-description-spec.md),
[`docs/devnet-pricing.md`](../devnet-pricing.md), [`docs/solana-deployment.md`](../solana-deployment.md),
[`docs/evm-deployment.md`](../evm-deployment.md), [`CONTEXT.md`](../../CONTEXT.md),
ADRs 0003, 0010, 0022, 0046, 0050, 0052, 0063, 0068, 0069, 0070,
[`crates/connector-config/src/peer.rs`](../../crates/connector-config/src/peer.rs),
[`packages/faucet/src/`](../../packages/faucet/src/).

**`toon-protocol/toon-client`** @ `main`, 2026-09-14: `README.md`, `CONTEXT.md`,
`docs/{cli,api,getting-started,channels,devnet,hidden-service,troubleshooting,development}.md`,
`packages/client/{package.json,CHANGELOG.md}`,
`packages/client/src/cli/{anon-daemon,context,main}.ts`,
`packages/client/src/cli/commands/{init,channel}.ts`,
`packages/client/src/client/{config,types,channel-facade}.ts`,
`packages/client/src/transport/{socks,socks-url,hs-hostname}.ts`,
`packages/client/src/keys/keystore-node.ts`, `packages/client/src/presets.ts`,
`packages/client/src/wire/vectors/wire-vectors.provenance.json`,
`packages/client/src/__integration__/hidden-service.integration.test.ts`.

**Seller repositories** @ `main`: `relay:deploy/connector.toml` + `deploy/docker-compose.yml` +
`README.md`; `store:deploy/connector.toml.template` + `deploy/docker-compose.yml` + `README.md`;
`gas-station:deploy/connector.toml.template` + `deploy/docker-compose.yml`;
`anytoon:config/{connector.toml,connector.local.toml,anonrc,anonrc-client}` + `buyer/` + `README.md`

- `docs/adr/0003`.

**Registries and live endpoints, read 2026-09-18:** `registry.npmjs.org/@toon-protocol/client`;
`GET /ilp` on `proxy.{relay,ario,gas}.devnet.toonprotocol.dev`;
`GET /ilp/routes/price` on the relay and the store;
`GET faucet.devnet.toonprotocol.dev/api/info`; `GET dvm.devnet.toonprotocol.dev/health`;
`gh release list --repo toon-protocol/connector`; `gh run list` and branch listings for the three
node repositories.

**Local Omarchy 4.0.0.alpha** (`/usr/share/omarchy/version`; package `omarchy 4.0.3-1`), read-only:
`install/user/{all.sh,mise-work.sh,mise.sh}`, `install/omarchy-base.packages`,
`install/omarchy-other.packages`, `install/user/first-run/enable-user-units.sh`,
`install/user/default-keyring.sh`, `install/config/ssh-command-path.sh`,
`default/bash/{env-bootstrap,init}`, `default/systemd/user/*.service`,
`bin/omarchy-mise-install`, `migrations/{1786952219,1787215483,1788577553,1788724825}.sh`,
`/etc/mise/conf.d/omarchy.toml`, plus `pacman -Q`/`-Qo`/`-Qi` and `which -a`.
