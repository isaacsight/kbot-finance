# RFC: Content-Addressed Request Envelope for MCP

> **Submission text for GitHub Discussion on `modelcontextprotocol/spec`.**
> Paste the body below into a new Discussion under the "Ideas" or "RFCs"
> category. The full spec lives next door at
> [`RFC-content-addressed-mcp.md`](./RFC-content-addressed-mcp.md).

---

## Title (when posting)

```
RFC: Content-addressed request envelope + deterministic replay for MCP tool calls
```

## Category

`Ideas` → or `Show and tell` if Ideas is not enabled.

## Labels (if available)

`rfc` · `spec-extension` · `audit` · `regulated-industries`

---

## Body

### Summary

I'm proposing a small, optional MCP extension that gives tool calls
**content-addressed identity** and **deterministic replay semantics**.
Every MCP request/response pair gains an opaque envelope containing:

- a SHA-256 hash over canonicalized inputs (the *replay key*),
- a pinned engine version,
- a data-as-of timestamp,
- an honesty flag (`byte_identical_replayable: true | false`) the server
  uses to declare whether identical envelopes will resolve to byte-identical
  responses.

Servers that opt in expose a `replay(request_hash)` tool. Clients that don't
implement the extension keep working unchanged — the envelope rides in
`_meta` and is harmless to ignore.

I'm posting because this came out of building an open-source audit substrate
for AI agents in regulated industries — `@kernel.chat/kbot-finance` — and
the envelope is the load-bearing primitive of that work. I'd like the
extension to belong to MCP, not to one implementation.

### Motivation

MCP has won the agent-tool protocol race for the right reasons — clean
JSON-RPC shape, capability negotiation, model-agnostic tool definition.
It is missing one thing for regulated-industry deployment: **auditable,
replayable tool calls**.

Today, a regulator (or an internal compliance officer, or a postmortem
author) asking "what did the agent see when it made this decision?" gets
a transcript with no canonical replay key. The compliance regimes that
care — EU AI Act Annex IV (Aug 2026 enforcement, possibly Dec 2027 after
the May 2026 Council/Parliament agreement), Federal Reserve SR 26-02
(Apr 2026), ESMA's algorithmic-trading supervisory briefing (Feb 2026),
FINRA 2026 ROR — all require artifacts the current MCP transcript cannot
produce: hash-chained lineage, byte-identical replay where the engine is
deterministic, audit-ready exports.

Bloomberg ASKB (Feb 2026 launch) ships exactly this pattern internally:
agents emit BQL code, the BQL engine produces the number, and the output
is replayable inside the Terminal. The pattern is closed and proprietary.
This RFC is the open equivalent, designed so any MCP server in any
domain can implement it without leaving the spec.

### The envelope shape

Servers implementing the extension attach `_meta` to every tool response:

```jsonc
{
  // ...standard MCP CallToolResult fields...
  "_meta": {
    "kbot-finance/content-addressed": {
      "version": "0.1",
      "request_hash": "<64-hex SHA-256>",
      "engine_version": "<vendor-defined string>",
      "schema_hash": "<64-hex SHA-256 over the tool's JSON schema>",
      "data_as_of": "<ISO 8601 UTC timestamp>",
      "produced_at": "<ISO 8601 UTC timestamp>",
      "byte_identical_replayable": true,
      "deterministic_seed": "<optional hex string>"
    }
  }
}
```

`request_hash` is computed as:

```
SHA-256(canonicalize({
  operation: <tool name>,
  engine_version,
  schema_hash,
  inputs: <tool args>,
  data_as_of,
  deterministic_seed?  // present iff supplied
}))
```

`canonicalize()` follows RFC 8785 JSON Canonicalization Scheme (JCS),
with the additional constraint that non-finite numbers and `undefined`
values MUST cause hashing to fail rather than be silently elided. This
is the part the financial-services and regulated audit use cases need
non-negotiable — a hash that silently swallows a NaN is worse than no
hash at all.

### The honesty primitive

The single hardest design decision was what to do when the engine isn't
deterministic. Live HTTPS APIs (Polymarket Gamma, SEC EDGAR's `recent`
table), GPU-based LLM inference (CUDA reduction order is not specified by
IEEE 754), and any tool whose backing engine isn't bit-stable cannot
guarantee byte-identical replay.

The proposal is to make the truth machine-readable. Servers MUST emit
`byte_identical_replayable: false` when they cannot promise byte-identical
replay, and `true` only when they can. A server claiming `true` when it
cannot deliver is non-conformant. The audit log still records the hash and
the response — auditors get the chain — but the replay-byte-for-byte
property is opt-in per-call rather than a universal claim.

Practical consequence: the same MCP server can serve real-time market
data (`replayable: false`) and historical computations against pinned
snapshots (`replayable: true`) from the same surface. The client decides
which it can rely on.

### The `replay` tool

Servers implementing the extension SHOULD expose a tool named `replay`:

```jsonc
{
  "name": "replay",
  "description": "Re-execute a prior request by hash and return the byte-identical result, or report a mismatch.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "request_hash": {
        "type": "string",
        "description": "The hash returned by a prior call."
      }
    },
    "required": ["request_hash"]
  }
}
```

Replay returns either the original sealed envelope (byte-identical when
the server's audit log retains the request and the engine is
deterministic for that request) or a `replay_mismatch` error with the
diff details.

### Capability advertisement

Servers advertise via standard MCP capability negotiation:

```jsonc
{
  "capabilities": {
    "tools": {},
    "experimental": {
      "kbot-finance/content-addressed": { "version": "0.1" }
    }
  }
}
```

The `experimental.` prefix is the right caution for v0.1. If the spec
ratifies, the prefix drops and the capability moves into the standard
namespace.

### Reference implementation

[`@kernel.chat/kbot-finance@0.2.0`](https://www.npmjs.com/package/@kernel.chat/kbot-finance)
ships:

- `sealEnvelope()` — wraps an engine call, computes the hash, returns
  the envelope.
- `AppendOnlyAuditLog` — hash-chained append-only log with `verify()`
  for integrity checks.
- `mcp-server.ts` — MCP server exposing four audit-grade tools
  (`polymarket_query`, `edgar_query`, `annex_iv_export`, `audit_log_verify`)
  using this envelope shape.

Quick install:

```bash
npm install @kernel.chat/kbot-finance
npx -y @kernel.chat/kbot-finance demo   # end-to-end against live Polymarket
npx -y @kernel.chat/kbot-finance mcp    # start the MCP server on stdio
```

Apache 2.0. The full spec draft is in the package as
`RFC-content-addressed-mcp.md`. Source on GitHub at
[`isaacsight/kernel/tree/main/packages/kbot-finance`](https://github.com/isaacsight/kernel/tree/main/packages/kbot-finance).

### What this is not

To save discussion cycles:

- **Not a replacement for MCP's existing schema or transport.** Purely
  additive `_meta` payload + one optional tool.
- **Not a determinism mandate.** Tools that can't be deterministic
  emit `byte_identical_replayable: false` honestly. The envelope still
  carries the hash for audit purposes.
- **Not a cryptographic algorithm bake-off.** SHA-256 is the v0.1
  default. The spec leaves room for a `hash_algorithm` field if BLAKE3,
  SHA-3, or post-quantum hashes need to be supported later.
- **Not a financial-services-only spec.** Finance is the wedge because
  the regulatory pull is sharpest there, but the same audit primitive
  applies to healthcare AI (FDA SaMD), legal AI (regulatory-rule-as-code),
  drug discovery, defense, and any other domain where "show me what
  happened" is a real question with legal consequences.

### Security considerations

- **Hash collisions.** SHA-256 collision resistance is ~2^128 — adequate
  for the audit horizon of regulated industries (10 years per EU AI Act
  Art. 19). Post-quantum migration handled via a `hash_algorithm` tag.
- **Replay attacks.** The envelope is a content identifier, not an
  authentication token. Session authentication remains the client's
  responsibility.
- **Privacy.** Canonicalized inputs may contain sensitive data. Audit
  log storage must respect the same access controls as the inputs.
  The hash itself reveals nothing about the inputs.
- **Side-channel determinism.** Bit-identical replay across hardware
  (x86 vs ARM, AVX-512 vs not, GPU vs CPU) requires pinned-architecture
  execution or correctly-rounded math (CRlibm, sleef). Servers that
  cannot pin their execution environment MUST emit
  `byte_identical_replayable: false`.

### Open questions

These are unresolved in the v0.1 draft. I'd like the working group's
read on each:

1. **Hash-algorithm agility.** Should v0.1 mandate a `hash_algorithm`
   tag even if SHA-256 is the only currently-supported option?
2. **Tool naming.** Is `replay` too generic a tool name for a global
   namespace? Alternatives: `audit_replay`, `__replay`, `mcp.replay`.
3. **Audit log shape.** Should the spec standardize the JSONL audit
   entry format from the reference implementation, or leave the log
   shape to implementations and only standardize the envelope?
4. **Approval-token integration.** The reference implementation
   includes a `governance.ts` module with HMAC-signed material-gate
   approval tokens bound to `request_hash`. Should the approval-token
   shape be part of this RFC, or a separate follow-up RFC?
5. **JSON-RPC `_meta` versus a top-level field.** `_meta` is the
   conservative choice (additive, transparent to non-implementing
   clients). A top-level `envelope` field would be more discoverable
   but breaks backwards compatibility. v0.1 picks `_meta`; reopening
   the decision before adoption seems wise.

### What I'd ask for from the working group

In rough order:

1. **Reactions on the basic shape** — does the envelope-in-`_meta`
   pattern fit MCP's intended evolution, or is there a different
   shape the working group already has in mind?
2. **Adoption of the honesty primitive** as a normative requirement
   in any audit-related extension — `byte_identical_replayable`
   matters in any spec that touches regulated deployment.
3. **Discussion of the five open questions** above, in the
   Discussion thread or via individual issues if any of them is
   load-bearing enough to warrant one.
4. **An owner from the spec maintainers** who would champion this
   through to a real RFC track if the working group is interested.

If the answer is "this should be a separate spec entirely, not an
MCP extension," I'm fine to take it elsewhere — the work needs a home
more than it needs a particular flag on a particular protocol. But
MCP is the natural place because the audit substrate is most useful
when it travels with the tool-call surface every agent already speaks.

Happy to revise the draft based on feedback, or split it into smaller
RFCs if the working group prefers a more incremental approach. The
reference implementation is real, public, and Apache 2.0, so any
direction the spec takes can adopt the parts it likes and leave the
rest.

— Isaac Hernandez · kernel.chat
  isaacsight@gmail.com · github.com/isaacsight
