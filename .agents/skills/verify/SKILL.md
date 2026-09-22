---
name: verify
description: Verify a Tailcat candidate against real runtime behavior and produce auditable PASS/FAIL/BLOCKED/SKIP evidence. Use for /verify, release checks, networking changes, CLI changes, and bug-fix validation.
---

# /verify — Tailcat Runtime Verification

Verify the **exact candidate** that would be shipped. Do not equate build success, unit tests, CI green, or a process staying alive with runtime correctness.

## Contract

A verification run follows:

```text
Candidate
  ↓
Launch
  ↓
Drive from a real peer
  ↓
Capture runtime evidence
  ↓
Adjacent probe
  ↓
Judge: PASS | FAIL | BLOCKED | SKIP
```

### Non-negotiable rules

1. Record the exact candidate first:
   ```sh
   git rev-parse HEAD
   git status --short
   ```
2. Build and test the checked-out source, not an unrelated installed binary:
   ```sh
   go test ./...
   go build -o /tmp/tailcat-verify ./cmd/tailcat
   /tmp/tailcat-verify version
   ```
3. A build/test-only result is **not PASS**.
4. Exercise a real Tailcat listener and a real client peer.
5. Prefer an independent machine/network for the client. A second local process is only a lower-confidence fallback and must be labeled as such.
6. Use an **ephemeral server key** for verification. If a saved default key may exist, force a fresh key with `serve --key=new`.
7. Treat a Tailcat address as a bearer capability. Never commit it, paste it into public issues/PRs, or retain it in durable artifacts. Redact it from evidence; record only a fingerprint.
8. Stop the verification listener when finished.
9. Do not expose `no-auth-ssh` during verification. Use a narrow `exec` handler or an explicitly authenticated service.
10. If environment/network restrictions prevent a real peer from connecting, report **BLOCKED**, not PASS.

## 1. Establish the candidate

Capture:

- commit SHA
- working-tree state
- OS/architecture
- Go version
- Tailcat version built from this checkout

Recommended:

```sh
git rev-parse HEAD
git status --short
uname -a
go version
go test ./...
go build -o /tmp/tailcat-verify ./cmd/tailcat
/tmp/tailcat-verify version
```

Any test or build failure is **FAIL** unless the failure is proven unrelated to the candidate and is explicitly documented.

## 2. Launch a narrow real receiver

Use a random challenge and a single-purpose `exec` service. Keep stdout/stderr in a temporary directory.

Example shape:

```sh
TMP="$(mktemp -d)"
CHALLENGE="TAILCAT_VERIFY_$(date +%s)_$RANDOM"

nohup /tmp/tailcat-verify --verbose serve --key=new exec -- \
  /bin/sh -c 'IFS= read -r line; printf "TAILCAT_VERIFY_ACK:%s\n" "$line"' \
  >"$TMP/server.out" 2>"$TMP/server.log" < /dev/null &

SERVER_PID=$!
```

Wait until stderr contains:

```text
Server listening with new address:
```

Extract the address only in-memory. Do **not** print it into durable verification reports. For evidence, compute a fingerprint such as:

```sh
printf '%s' "$TAILCAT_ADDR" | shasum -a 256
```

Record only the SHA-256 fingerprint.

## 3. Drive it from a real peer

### Preferred: independent CLI peer

From a second machine or independent execution environment:

```sh
printf '%s\n' "$CHALLENGE" | tailcat "$TAILCAT_ADDR"
```

Expected application response:

```text
TAILCAT_VERIFY_ACK:<exact challenge>
```

Also probe connectivity:

```sh
tailcat ping "$TAILCAT_ADDR"
```

When direct P2P behavior is part of the change or release risk, additionally run:

```sh
tailcat ping --until-direct "$TAILCAT_ADDR"
```

Capture whether the pong is reported via DERP or a direct endpoint.

### Valid cross-device browser path

The official browser/WASM client at:

```text
https://tailscale.github.io/tailcat/
```

interoperates with the CLI and is a valid independent peer for send/receive verification.

Important: browser Tailcat traffic is DERP-relayed because browsers currently cannot use Tailcat's direct P2P path without WebRTC support. Therefore:

- browser → CLI successful challenge/ACK proves real cross-device tunnel connectivity;
- browser traffic remaining on DERP is **expected**;
- do not require `--until-direct` for the browser path.

## 4. Capture server-side runtime evidence

For a successful real connection, preserve the relevant **redacted** lines from the listener log. Strong evidence includes:

```text
got meow from nodekey:...
Received handshake initiation
Sending handshake response
Accept: TCP{...} ... tcp ok
```

For CLI-to-CLI direct-path verification, evidence may additionally include:

```text
via=direct
```

or a `tailcat ping --until-direct` pong that reports a direct endpoint.

Do not infer direct P2P from a successful TCP session alone. DERP fallback is a legitimate successful transport.

## 5. Verify the application payload

Network handshake evidence alone proves the tunnel formed, but not that the requested application data path worked.

Require an exact challenge/response:

```text
client sends:  <random challenge>
server returns: TAILCAT_VERIFY_ACK:<same challenge>
```

For a human-driven phone/browser test, capture both:

- server-side handshake/TCP evidence; and
- the ACK observed by the client.

If only the handshake/TCP evidence is available, report that specific layer as verified and the application ACK as unverified.

## 6. Run an adjacent probe

Every verification must include at least one nearby failure/edge case.

Default adjacent probe:

1. stop the ephemeral listener;
2. retry connectivity to the same ephemeral address with a bounded ping timeout;
3. require the attempt to fail.

Example:

```sh
kill "$SERVER_PID"
wait "$SERVER_PID" 2>/dev/null || true

/tmp/tailcat-verify ping --timeout=3s "$TAILCAT_ADDR"
```

Expected: non-zero exit because the ephemeral server no longer exists.

Other valid adjacent probes when relevant:

- malformed/truncated Tailcat address is rejected;
- unauthorized client is rejected by `serve --allow`;
- wrong port/service cannot silently reach another service;
- forced DERP fallback still transfers data;
- direct upgrade occurs after initial DERP bootstrap when the environment supports UDP hole punching.

Choose the probe closest to the code that changed.

## 7. Judge

Use exactly one overall state.

### PASS

PASS requires all of the following:

- exact candidate identified;
- required build/tests pass;
- real Tailcat listener starts from that candidate;
- independent peer establishes a real connection;
- expected application payload completes end-to-end;
- required runtime evidence is captured;
- adjacent probe behaves as expected;
- listener is stopped and ephemeral capability is no longer active.

Direct P2P is required only when the change/acceptance criterion specifically concerns direct-path/NAT-traversal behavior. DERP fallback can still be a valid PASS for ordinary connectivity.

### FAIL

Use FAIL when the candidate is testable but violates the expected behavior, including:

- listener cannot start;
- peer cannot establish the tunnel under otherwise-valid conditions;
- challenge is corrupted, missing, or receives the wrong response;
- security boundary behaves incorrectly;
- adjacent probe unexpectedly succeeds;
- a required direct-path criterion fails.

### BLOCKED

Use BLOCKED when the candidate cannot be meaningfully exercised because of an external constraint, for example:

- no independent peer is available;
- current runner blocks required network/DNS/DERP access;
- required platform is unavailable.

State exactly what was verified before the block and what remains unverified.

### SKIP

Use SKIP only when a verification surface is demonstrably irrelevant to the candidate. Give the reason.

## 8. Evidence report format

Keep the final report compact and auditable:

```text
VERIFY: PASS | FAIL | BLOCKED | SKIP
Candidate: <commit SHA>
Working tree: clean | <summary>
Build/tests: <result>
Server: started from candidate
Address fingerprint: sha256:<redacted fingerprint>
Peer: <independent CLI | browser/WASM | fallback local process>
Tunnel: <DERP | direct | upgraded DERP→direct | unknown>
Handshake: <evidence summary>
Payload: <challenge → exact ACK>
Adjacent probe: <probe and result>
Cleanup: listener stopped, ephemeral address inactive
Evidence: <paths/log excerpts/screenshots as applicable>
```

## Known-good baseline

On 2026-09-22, a Tailcat CLI listener running on a Mac mini was reached from the official Tailcat browser/WASM client on a phone over the public internet. The server recorded a new peer, WireGuard handshake initiation/response, and an accepted TCP connection.

Use that result only as a **historical known-good topology**:

```text
Phone browser/WASM
      ↓
DERP
      ↓
Tailcat CLI on Mac mini
      ↓
WireGuard handshake + TCP accept
```

It is not proof that a future candidate passes. Re-run verification against the exact candidate.

## What does not count as verification

None of these alone is sufficient:

- `go test ./...` passes
- `go build` succeeds
- GitHub Actions is green
- a binary prints `--help` or `version`
- the listener prints a Tailcat address
- a process remains alive
- a deployment or release artifact exists
- a local mock/simulated response returns success

The verifier must drive the real runtime and produce evidence tied to the exact candidate.
