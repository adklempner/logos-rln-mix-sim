### What the user achieves

Runs 5 `logoscore` daemons as a docker-compose stack and watches **every node obtain
an RLN membership through allocated (gifted) on-chain registration** on the hosted
LEZ testnet. One node — `relay1`, the **gifter** — holds the only funded wallet. It
registers its own membership, then serves the LIP-158 allocation protocol
(`/logos/rln/membership/1.0.0`). The other four nodes authenticate to it with an
EIP-191 signature and each receive a **distinct on-chain membership without ever
funding or signing a transaction**: a client generates its RLN identity locally and
sends only the identity *commitment*; the gifter funds and signs the registration
and returns the allocated leaf.

The memberships are then exercised in context — the 5 nodes form a 3-hop Sphinx mix
and exchange request/reply round-trips with RLN spam protection enforced **per hop
on both legs** (LIP-144):

```
sender ──Sphinx──▶ relay(hop1) ──▶ relay(hop2) ──▶ relay(hop3/exit) ──▶ dest
        generate        verify+regen    verify+regen     verify          deliver
```

Each relay verifies the incoming proof and regenerates a fresh one for the next hop,
which is why every mix node must be a member (`PathLength=3` + exit≠dest force the
5-node minimum). The headline is the allocation; the mix is the setting that makes
per-node membership necessary.

### Why it matters

**Self-service on-chain RLN registration is a barrier to entry.** To join an RLN
group the normal way, a user must hold funds, pay gas, and submit a chain
transaction from their own key — and doing so over a public RPC links their network
identity to their RLN identity. LIP-158 removes that barrier: a **membership
provider registers the client's identity commitment on the client's behalf**, so
the client needs no funds, no chain access, and no wallet of its own — and the
provider never learns the client's RLN secret, only the public commitment, so it
cannot forge the client's proofs. This sim demonstrates that primitive end to end
on a live testnet: one funded account registers four other nodes' independent
identities.

**This is possible because RLN is implemented natively in the Logos Execution Zone.**
The RLN group lives on the LEZ testnet as risc0 zkVM guest programs plus accounts
derived from `(registration_program_id, tree_id)`; `register_member` is a real
sequencer transaction that spends RLNTOK and grows an on-chain Merkle tree, and it
decouples funder≠identity — the wallet is only the tx funder/signer. Nothing is
mocked.

**Mix DoS protection is why allocation matters for a mixnet.** A mix relay can't
rate-limit by sender identity (anonymity hides it), so LIP-144 has each hop verify
"some member, within their rate" in zero knowledge and re-prove for the next hop.
A member may send at most `userMessageLimit` messages per epoch; exceeding it reuses
a nullifier, which reveals enough (two Shamir shares) to reconstruct the spammer's
secret key and slash it — enforcement is in-band, the provider is not in the loop
after allocation. Because the protection is per hop, **every relay must be a member
before it can forward a single packet** — cheap, authenticated membership
allocation is what makes growing a mix fleet practical.

### Key components

- **`logos-rln-mix-sim`** (this repo) — the compose stack, `bootstrap.sh`,
  `orchestrate.sh` (drives all 5 daemons over `logoscore call`), the image recipe,
  host-side key derivation (`keys.py`), and the demo EIP-191 auth fixtures
  (`fixtures/gifter_auth/`).
- **`logos-rln-gifter`** (sibling, `master`) — the standalone membership-gifter
  libp2p protocol module: a self-contained nim-libp2p `LPProtocol` implementing
  `/logos/rln/membership/1.0.0` (LIP-158) with EIP-191 allowlist auth, plus the
  client side. EIP-191 is used **only** for the client↔gifter handshake.
- **`logos-libp2p-module`** (sibling, `feat/enable-mix`) — the universal Logos Core
  libp2p module; exposes every method the orchestrator calls (`rlnEnable`,
  `rlnGifterServe`/`rlnGifterRequest`, `rlnIsReady`, `mixSetNodeInfo`,
  `mixNodepoolAdd`, `mixRegisterDestReadBehavior`, `mixDialWithReply`,
  `streamWrite`/`streamReadExactly`).
- **`nim-libp2p-mix`** (sibling, `feat/mix-cbind`) — LIBP2P-MIX (LIP-99): Sphinx
  packets, SURB replies, cover traffic, and the pluggable `SpamProtection`
  interface. Wire format `[Sphinx packet][σ]`: each hop strips the RLN proof σ,
  peels one Sphinx layer, appends a fresh σ.
- **`mix-rln-spam-protection-plugin`** (sibling, `feat/cbind-rln`) — the RLN
  implementation of `SpamProtection` (LIP-144) over zerokit (RLN-v2/Poseidon,
  Merkle depth 20), proofs bound to the packet bytes so they can't be replayed;
  nullifier log for double-signaling detection. Unchanged by the gifter work —
  only *how a node acquires its identity* changed.
- **`logos-lez-rln`** (cloned inside the image, `feat/spel`) — the wallet + RLN
  Logos Core modules as `.lgx` bundles; implements `register_member`, the
  funder≠identity registration the gifter drives. Carries the RLN patches in-branch
  and fetches the execution zone (lssa) at tag `v0.2.0-rc6` via its flake — nothing
  is vendored.

### Repository

https://github.com/adklempner/logos-rln-mix-sim

### Runtime target

testnet v0.2

### Prerequisites

- OS: Linux or macOS (all scripts are bash-3.2 compatible)
- Tools: **Docker** running (~30 GB free in its VM — the image is ~17 GB and a cold
  Nix build needs headroom) and `git`
- Network: outbound access to GitHub, the Nix caches, and the LEZ testnet RPC
  (`https://testnet.lez.logos.co/`). There is no offline / local-chain mode.
- **No accounts, keys, or local toolchain** — `bootstrap.sh` fetches/builds
  logoscore, the wallet/RLN modules, the libp2p `.lgx` (mix + RLN + gifter), and
  bakes the deployment profile (RLN tree + funded wallet) into the image. SSH keys
  optional (HTTPS override below). The EIP-191 auth fixtures are demo keys checked
  into this repo — **NOT for production**.

### Commands and expected outputs

```sh
# 1. Clone and bootstrap (clone 4 sibling repos + build the .lgx + the image;
#    ~30-45 min first run, fast on re-runs)
git clone git@github.com:adklempner/logos-rln-mix-sim.git
cd logos-rln-mix-sim
bash docker/testnet/mix_e2e/bootstrap.sh

#    No SSH keys? The forks are public — clone + bootstrap over HTTPS:
#      git clone https://github.com/adklempner/logos-rln-mix-sim.git
#      REPO_BASE=https://github.com/adklempner bash docker/testnet/mix_e2e/bootstrap.sh
#    (Manual equivalent: clone logos-libp2p-module@feat/enable-mix,
#     mix-rln-spam-protection-plugin@feat/cbind-rln, nim-libp2p-mix@feat/mix-cbind,
#     logos-rln-gifter@master as siblings; LOGOS_ROOT="$(cd .. && pwd)" bash
#     docker/build_lgx_linux.sh; docker build -f docker/Dockerfile.testnet-e2e
#     -t lp2p-mix-e2e .  The image build clones logos-co/logos-lez-rln@feat/spel
#     internally. Changed a source repo? Re-run only the .lgx build — the compose
#     entrypoint overlays it over the image's baked copy.)

# 2. Run the full E2E (~15 min; the 5 sequential on-chain registrations dominate)
cd docker/testnet/mix_e2e
bash orchestrate.sh

# 3. Tear down
docker compose down
```

Real output of a passing run (peer IDs, leaf indices, and block heights vary per
run; section headers, log strings, counts, and the verdict are stable):

```text
=== up: 5 daemons (force-recreate for FRESH daemons) ===
  config=FUhP8quu5WKEL33oALSgDnXq9JZ8Qx72en7zSrzmPrDC holding(funder)=9xhSHTkuFj8m4BbB1QA5W3pQHAeiRMkQdk4TGG8fCZz4
=== per-node setup (load chain -> wallet+rln -> start -> mixSetNodeInfo -> peerInfo -> register) ===
  relay1 wallet synced to 3385
  relay1 peerId=16Uiu2HAm3c6...  leaf_opt=58 leaf_actual=58 confirmed=true rlnIsReady=True
  relay1 gifter service mounted (/logos/rln/membership/1.0.0, allowlist=4 clients)
  relay2 peerId=16Uiu2HAmCCm...  leaf_opt=59 leaf_actual=59 confirmed=true rlnIsReady=True
  relay3 peerId=16Uiu2HAmUqo...  leaf_opt=60 leaf_actual=60 confirmed=true rlnIsReady=True
  dest   peerId=16Uiu2HAmGJr...  leaf_opt=61 leaf_actual=61 confirmed=true rlnIsReady=True
  sender peerId=16Uiu2HAmMSU...  leaf_opt=62 leaf_actual=62 confirmed=true rlnIsReady=True
=== mesh: every node adds the other 4 ===
  meshed.
=== rlnIsReady status (each node was confirmed ready before the next registered) ===
   relay1=True relay2=True relay3=True dest=True sender=True
=== root convergence: wait until every node's valid-roots window includes the final root ===
  valid-roots converged across all 5 nodes; settling one epoch for verifier windows
=== register dest-read-behavior on all nodes (the SURB exit is random) ===
  registered (/ipfs/ping/1.0.0, READ_EXACTLY, 32 bytes)
=== exchange: 3 request/reply round-trip(s) per initiator ===
  sender->dest: 3/3 replies received
  dest->sender: 3/3 replies received
=== observe: RLN proofs (forward request + SURB reply legs) ===
  relay1: generated=8 verified=8
  relay2: generated=11 verified=11
  relay3: generated=11 verified=11
  dest: generated=3 verified=3
  sender: generated=3 verified=3
  replies: sender->dest=3 dest->sender=3 ; total verifications=36 ; sender proofs=3
  gifter(relay1): 'RLN gifter registration succeeded' x4 (expect 4 in the happy path)
=== VERDICT ===
  PASS: every round-trip got a reply (sender->dest=3 dest->sender=3).
DONE (NEG=0)
```

**How to read it — each output section is one orchestration stage:**

1. **`up` + `config=… holding(funder)=…`** — compose force-recreates 5 fresh
   daemons; the entrypoint installs the freshly-built `.lgx` and sets the libp2p
   listen address to the **container IP** (not `0.0.0.0`, which would also
   advertise loopback and break mix/SURB/gifter dials). The two accounts come from
   the baked deployment profile: the RLN instance's config account (a
   program-derived address) and the funded payment account — only `relay1` ever
   spends from it.
2. **`per-node setup`** — for each node in turn: load the wallet + RLN + libp2p
   modules, open and sync the wallet to chain head (`wallet synced to <block>`),
   `rlnEnable` against the profile's config account (must precede `mixSetNodeInfo`
   — the mix reads the spam-protection factory at mount), `start`, `mixSetNodeInfo`.
   Clients open the wallet **read-only** (the RLN module resolves accounts and
   reads the tree through it); only relay1 signs or spends. Then the allocation
   line, one per node: `leaf_opt` is the leaf the registration optimistically
   returned, `leaf_actual` what the on-chain tree assigned once confirmed — **they
   must match**; `confirmed=true` is the on-chain `is_member_registered` check;
   `rlnIsReady=True` means the node holds its identity + Merkle proof and can
   generate proofs. relay1 self-registers (`rlnRegister`) and mounts the gifter
   service; each client then calls `rlnGifterRequest` — generate identity from a
   seed locally, sign EIP-191 over the idCommitment with its fixture key, dial
   relay1; the gifter authenticates against its allowlist (one membership per
   address), runs `register_member` funded by its own wallet, returns the leaf; the
   client adopts it (`rlnSetIdentity` + proof-refresh timer). Allocations are
   **serialized by the on-chain confirmation barrier** (next client waits for
   `registered:true`), keeping the gifter wallet's txs nonce-ordered and every
   membership on a distinct leaf.
3. **`mesh`** — every node `mixNodepoolAdd`s the other four. Pubkeys are derived
   host-side by `keys.py` (byte-returning RPCs are UTF-8-corrupted over `--json`).
4. **`root convergence`** — every registration advanced the on-chain Merkle tree,
   and a verifier's valid-roots window refreshes only on its ~10 s epoch timer, so
   the sim re-syncs all wallets, polls `get_valid_roots` until all 5 nodes agree on
   the final root, then settles one epoch — otherwise a hop can reject the first
   proof (`Proof rejected: invalid Merkle root`).
5. **`register dest-read-behavior`** — `mixRegisterDestReadBehavior` on all nodes
   (the SURB exit is random), so whichever node is the exit echoes `/ipfs/ping`.
6. **`exchange`** — 3 round-trips per direction:
   `mixDialWithReply` → `streamWrite` → `streamReadExactly`, the reply returning
   over the SURB path. RLN is generated/verified at every hop on both legs. A
   "reply received" means `streamReadExactly` returned success (reply bytes are
   UTF-8-corrupted over `--json`, so it is not a byte compare — the proof counts
   corroborate delivery).
7. **`observe`** — counts scraped from daemon logs. sender/dest generate exactly
   one proof per message they originate (3 each); relays run higher, uneven counts
   because each relay verifies + regenerates per forwarded packet on both legs and
   Sphinx paths are re-randomized per message. Total verifications ≈ 6 per
   round-trip (3 forward + 3 reply hops) × 6 round-trips = **36**. The gifter line
   confirms exactly 4 gifted registrations in relay1's log.

The daemon logs (`docker compose logs <service>`) carry the corroborating strings:
relay1 logs `handling RLN gifter request` + `RLN gifter registration succeeded` per
client; each client logs `RLN membership granted`; every node logs
`Generated RLN proof successfully` / `Proof verified successfully` per hop.

**The negative runs** (both prove RLN gates delivery — the sender comes up with mix
fully configured but no membership, and gets nothing through):

```text
# NEG=1 — sender never asks the gifter:
  sender peerId=16Uiu2HAm... UNREGISTERED (negative) rlnIsReady=False
  sender->dest: 0/3 replies received
  PASS (negative): sender got 0 replies and generated 0 proofs -> rejected (NEG=1).

# NEG=2 — sender asks, but signs with a key NOT on the allowlist; the gifter
# refuses authentication (exercises the allocation auth gate specifically):
  sender peerId=16Uiu2HAm... REFUSED (negative, non-allowlisted key) rlnIsReady=False
  sender->dest: 0/3 replies received
  PASS (negative): sender got 0 replies and generated 0 proofs -> rejected (NEG=2).
```

### Success command

```sh
bash orchestrate.sh   # in docker/testnet/mix_e2e, after bootstrap.sh
```

### Expected result

`VERDICT: PASS`, exit code 0. The allocation lines are the point: **5 distinct
on-chain leaves (1 self + 4 gifted), `leaf_opt == leaf_actual`, `confirmed=true`,
`rlnIsReady=True` on every node**, all funded by the single gifter. Then the mix
exercise: `sender->dest: 3/3` and `dest->sender: 3/3` replies, ~36 per-hop RLN
verifications, `'RLN gifter registration succeeded' x4`. The negative runs print
`PASS (negative)` with 0 replies and 0 sender proofs. Peer IDs and leaf indices are
non-deterministic across runs; counts, log strings, and the verdict are stable.

### Configuration details

Near-zero-config: `bash orchestrate.sh` runs the full E2E. Everything is fixed in
`orchestrate.sh` (3 round-trips each way, `/ipfs/ping` echo, RLN
`userMessageLimit`=100, `epochDurationSeconds`=10.0, testnet RPC) except:

| Knob | Purpose | Example |
|---|---|---|
| `NEG` | Negative enforcement tests: `0` (default) full E2E; `1` sender never asks the gifter → rejected; `2` sender signs with a non-allowlisted key → gifter refuses auth | `NEG=2 bash orchestrate.sh` |
| `DEPLOYMENT` (build-arg) | Which on-chain deployment (RLN tree + funded wallet) is baked into the image as the profile (`docker/testnet/deployments/<name>/`, default `shared-5ade`) | `docker build -f docker/Dockerfile.testnet-e2e --build-arg DEPLOYMENT=fresh-tree -t lp2p-mix-e2e .` |

See `../deployments/README.md` for provisioning new profiles.

### Failure modes and limits

`orchestrate.sh` **auto-detects** the two real-world registration failures by
scanning relay1's log and prints the exact fix (set
`LEZ_RLN_DIR=/path/to/logos-lez-rln` so the printed commands show real paths). Both
fixes provision a fresh deployment and rebuild the image — the gifter signs with the
*baked* wallet, so a new payment account or tree must be baked in. Build the host
tools once first (a host `logos-lez-rln` checkout also needs a plain `lssa/` sibling
clone at `v0.2.0-rc6` — the flake fetches it for nix builds; host cargo builds read
it from disk):

```sh
(cd "$LEZ_RLN_DIR/lez-rln" && PYO3_PYTHON=$(command -v python3) \
    cargo build --release --bin run_setup --bin derive_accounts)
```

1. **Payment account out of funds** — registrations stop confirming; relay1's log
   shows `Insufficient balance` (each registration costs `price_per_unit * rate`
   RLNTOK). Re-provision on the **same tree** (`run_setup` mints a fresh funded
   payment account) and rebuild:
   ```sh
   D=docker/testnet/deployments/shared-5ade
   LEZ_RLN_DIR=/path/to/logos-lez-rln bash docker/testnet/provision.sh \
     --name shared-refunded --tree $(jq -r .tree_id "$D/deployment.json") --adopt-wallet "$D/storage.json"
   docker build -f docker/Dockerfile.testnet-e2e --build-arg DEPLOYMENT=shared-refunded -t lp2p-mix-e2e .
   ```
   If the log says `supply holding may be out of funds` instead, the master supply
   is exhausted → provision a brand-new tree (next item).
2. **Tree full** — relay1's log shows `Would exceed max total rate limit` (~10k
   members at rate 100 is the practical cap). `tree_id` is the single knob, no
   source edits — provision a brand-new tree and rebuild:
   ```sh
   LEZ_RLN_DIR=/path/to/logos-lez-rln bash docker/testnet/provision.sh --name fresh-tree
   docker build -f docker/Dockerfile.testnet-e2e --build-arg DEPLOYMENT=fresh-tree -t lp2p-mix-e2e .
   ```
3. **Testnet unreachable** — everything needs `https://testnet.lez.logos.co/`
   (chain head, wallet sync, registration); setup stops at the first sync or
   registration barrier. No offline mode.
4. **Transient registration/gifter-request failures are retried** — `rlnRegister
   attempt N failed … re-sync + retry in 15s` (or the gifter equivalent) is the
   harness re-syncing and retrying; only repeated failures with one of the log
   signatures above are real.
5. **`!! LEAF MISMATCH`** (`leaf_opt != leaf_actual`) — two registrations raced the
   leaf counter; relays will reject that node's proofs. The confirmation barrier
   prevents this; seeing it means a barrier was bypassed.
6. **`Proof rejected: invalid Merkle root`** in a relay's daemon log — a hop's
   valid-roots window didn't include the proof's root. The root-convergence barrier
   exists to prevent this; seeing it means the barrier was skipped or timed out
   (the run prints a warning in that case). Reply counts below 3/3 fail the
   verdict — check the relays' daemon logs for these rejection strings.

**Limits:** each run allocates 5 fresh identities, so leaves accumulate on the
shared tree across runs. Use `logoscore call` for lifecycle calls, not one-shot
`-c "…"`/`--quit-on-finish` (the one-shot client doesn't await async calls and
reports a spurious timeout).

### GitHub handle

adklempner

### Discord handle

arseniy

### Existing docs or specs

- RLN Membership Allocation spec (LIP-158): https://lip.logos.co/anoncomms/raw/rln-membership-service.html
- RLN DoS Protection for Mixnet spec (LIP-144): https://lip.logos.co/anoncomms/raw/mix-spam-protection-rln.html
- LIBP2P-MIX spec (LIP-99): https://lip.logos.co/anoncomms/raw/mix.html
- Repo README (overview + quick start): https://github.com/adklempner/logos-rln-mix-sim
- Deployment profiles (provision/verify): `docker/testnet/deployments/README.md`

### Hardware requirements

~30 GB free disk in Docker's VM (the image is ~17 GB; a cold Nix build needs
headroom); no special CPU/RAM/bandwidth beyond a typical dev machine.

### Estimated time to complete

~30-45 min first bootstrap (image build dominates; re-runs fast), then ~15 min per
sim run (the 5 sequential on-chain registrations dominate).

### Security notes

The gifter is a **membership provider / gatekeeper**, and the LIP-158 trade-offs
apply:

- **Centralization / censorship.** The provider decides which commitments get
  registered and under what policy, and can refuse or stall any request. Here that
  authority is one node (`relay1`) — `NEG=2` demonstrates the refusal path.
- **Sybil resistance = the auth policy, nothing more.** LIP-158 authentication is
  pluggable and there is no protocol-level cap on memberships per authenticated
  party. This sim uses the spec's demo mode — a static EIP-191 allowlist with one
  membership per address (`fixtures/gifter_auth/`); the demo keys are checked into
  the repo and are **not for production**.
- **Custody and metadata.** The provider fronts the funds for every registration
  and learns which identity commitment belongs to which authenticated client — not
  the RLN secret (the client generates its identity locally and sends only the
  commitment), but a durable auth-identity↔commitment mapping.
- **IP↔identity correlation is out of scope.** LIP-158 reduces client↔chain
  linkability, but the client still reveals its network address to the provider;
  the spec defers this to future work (RLN Stealth Commitments), and so does this
  sim.
- **Slashing is the in-band enforcement.** Spam protection itself (LIP-144) is not
  provider-mediated: exceeding `userMessageLimit` per epoch reuses a nullifier,
  letting any relay reconstruct the offender's secret key from the two Shamir
  shares and remove it from the group. Honest single-rate traffic stays anonymous.
