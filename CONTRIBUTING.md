# Contributing to the pips

Specs are the part of picocash that most wants outside eyes. Here's how changes happen.

## The process

1. **Discuss first** — open an issue describing the problem or proposal. Design arguments happen in issues; PRs are for agreed text.
2. **Propose text** — PR against the relevant pip. Keep normative language crisp (MUST/SHOULD/MAY per RFC 2119) and separate rationale into non-normative prose.
3. **New capability?** New numbered pip, not a bolt-on to an existing one. Claim the next number in your PR.
4. **Implementation follows spec, never the reverse.** A pip changes here *before* the reference stack implements the change. If the reference implementation and a pip disagree, the pip wins and the implementation has a bug (or the pip needs an erratum PR — say which you think it is).

## Statuses

- **draft** — shape may change freely; implement at your own risk.
- **RFC** — implemented in the reference stack; feedback actively sought; changes still possible without ceremony.
- **frozen** — breaking changes require a new pip. Nothing is frozen yet.

## Test vectors

[vectors/](vectors/) are first-class normative artifacts: a conforming implementation in any language MUST reproduce them. If your implementation disagrees with a vector, either your code or the spec is wrong — both outcomes are worth an issue. Vectors are versioned; regenerating them is deterministic (see the provenance notes in [vectors/README.md](vectors/README.md)).

## Ground rules

- All amounts are integer base units, end to end. Proposals introducing floats to money paths will be declined.
- Public-facing text says **eCash**; the cryptography's NUT-00/12 byte-compatibility is documented in PIP-00 as attribution, where it belongs.
- Every mint API error carries a machine-readable `code` and a `recovery` hint. New endpoints follow suit.

Contributions are accepted under [Apache-2.0](LICENSE).
