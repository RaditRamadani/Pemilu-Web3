# Pemilu Web3

**An auditable tally for Indonesian election recapitulation.**
Every polling-station result is signed by witnesses, recorded append-only, and
hashed on-chain — so anyone can recompute the national total without trusting
the authority that published it.

> **Status:** design + specification. The contracts and dashboard are specified
> below and are next on the roadmap. No working prototype yet.

---

## The problem

Indonesian election results stay unofficial for weeks after voting day. In the
2024 election (~204 million registered voters, ~820,000 polling stations), the
official recapitulation took roughly a month. That gap is filled with
screenshots, premature victory claims, and hoaxes — not because the count is
wrong, but because there is no way for the public to *check* it.

The recognised attack surface is not the database. It is the step where numbers
travel upward: **TPS → kecamatan → kabupaten → national**. A number that changes
at any of those hops is hard to prove and easy to allege.

Existing efforts (Kawal Suara, quick counts) proved that *speed* is achievable.
What is still missing is **integrity that a stranger can verify**.

## What this is

A tally that is:

- **Append-only** — a correction is a new entry, never an overwrite.
- **Signed at the source** — KPU officer plus witnesses, not a single uploader.
- **Anchored** — a rolling hash is written to a public chain, so history cannot
  be rewritten by the authority that owns the data.
- **Recomputable** — a public verifier derives the total from raw records.

## What this is not

- **Not e-voting.** Paper ballots and manual counting at the TPS stay exactly as
  they are. This is the single most important constraint: ballot secrecy in a
  booth is a *security feature*, not a legacy limitation.
- **Not an official result.** KPU's manual recapitulation remains authoritative.
  This produces an *auditable tally*, labelled unofficial, published alongside.
- **Not a fix for vote buying.** See [Non-goals](#non-goals).

## Design principle

> **Nobody reports the total. The contract computes it.**

There is no `setNationalTotal()` anywhere in this design. The national figure
is a derived value, a function of the record set. You cannot alter a derived
value without altering the records, and every record is signed and chained.

## How it works

```
  TPS                 Ingestor                 Ledger (EVM)            Public
  ───────────────────────────────────────────────────────────────────────────
  count C-hasil  ───►  hash photo        ───►  submitResult()   ───►  dashboard
  signed by KPU        + numbers               ├ invariant check      ├ live tally
  + 2 witnesses        + witness sigs          ├ revert if invalid    └ verify TPS
                                               ├ tally += votes
                                               └ rollingRoot = H(prev, rec)
```

1. The TPS counts paper ballots and fills in the C-hasil form, signed by hand as
   it is today.
2. The KPU officer and each witness photograph **the same form** through the
   ingestor app — offline-first, synced when there is signal. Each copy is an
   independent observation.
3. The ingestor hashes the photo and submits the numbers with witness
   signatures. An invalid record reverts.
4. The dashboard shows a running tally, always with a **reporting-coverage
   denominator** (e.g. `42,318 / 820,000 TPS reporting — 5.2%`).
5. Anyone can download the raw records, recompute the total, and compare it with
   the on-chain rolling root.

## Components

| Component | Responsibility |
| --- | --- |
| `contracts/ElectionLedger.sol` | Invariant checks, on-chain tally, rolling audit root, role gating |
| `scripts/deploy.js` | Deploy to a testnet |
| `scripts/seed.js` | Load sample records and produce `data/seed.json` |
| `web/` | Static dashboard + per-TPS verifier, reads the chain directly via `viem` |
| `data/seed.json` | Public leaf set used by the verifier |

## Record schema

```solidity
struct Result {
    bytes32 tpsId;
    uint32  dpt;              // registered voters at this TPS
    uint32  ballotsIssued;    // ballots handed out
    uint32  ballotsValid;
    uint32  ballotsInvalid;
    uint32  abstain;          // legal outcome, not a residual
    uint16[] votes;           // per candidate / party
    bytes32 photoHash;        // keccak256 of the signed C-hasil photo
}
```

`abstain` is a first-class field. An election has a **ceiling** (eligible
voters), never a target. Any design that requires turnout to hit an exact number
must fabricate votes to reach it — which is the definition of the fraud it
claims to prevent.

## Invariants enforced on-chain

```solidity
ballotsValid + ballotsInvalid + abstain == ballotsIssued
ballotsIssued <= dpt
sum(votes) == ballotsValid
```

A record violating any of these **reverts at write time** and never enters the
ledger. This is the strongest reason to use a ledger rather than a database: the
rule is enforced by code, not by the honesty of the operator.

The rolling root makes revisions visible:

```solidity
rollingRoot = keccak256(abi.encode(rollingRoot, recordHash));
```

Re-submitting a TPS appends a new version. The history of corrections stays
readable — which reads as transparency rather than as tampering.

> `shortcut:` a hash chain gives chronological integrity but O(n) inclusion
> proofs. Upgrade to a Merkle tree / MMR if per-record proof size matters at
> national scale.

## Why a ledger and not a database?

A database can prove *what it currently stores*. A ledger can prove that
**nothing changed since a given time** — a claim that cannot be made by the party
we are trying to verify. That is the entire justification. Everything else in
this system (signatures, hashing, append-only storage) would work with plain
infrastructure.

## Data

`data/seed.json` is **synthetic** and labelled as such. It exists so the whole
pipeline can be demonstrated end to end. Replace it with real C-hasil data
before making any claim about accuracy.

## Non-goals

Written down on purpose — the boundaries are part of the design.

- **Vote buying.** The main defence against it is that a ballot in a closed
  booth cannot be proven to anyone. Any system with a verifiable receipt lets the
  buyer demand proof before paying. Traceable voting makes this worse, not
  better.
- **Remote / internet voting.** Same reason, plus client-side malware and the
  digital divide.
- **Truth of the source data.** If a form was filled in wrong, the ledger stores
  the error immutably and with a valid signature. Garbage in, gospel out.
- **Trustless operation.** Trust does not disappear; it moves — to the credential
  issuer, the key holders, and the contract author. This is a shift, not a
  removal.

## Roadmap

- [x] Problem framing, threat model, non-goals
- [ ] `ElectionLedger.sol` + invariant tests
- [ ] Witness multi-signature (`k`-of-`n` per record)
- [ ] Rolling root + Merkle inclusion proofs
- [ ] `seed.json` generation from sample data
- [ ] Dashboard: live tally with reporting coverage
- [ ] Per-TPS verifier page
- [ ] Deploy to a public testnet + anchor tx hash in this README
- [ ] Rolling root anchored to a public chain on a schedule

## Tech stack

Solidity `^0.8.24` · Hardhat · `viem` · static frontend (no backend; the
dashboard reads chain state directly).

## License

MIT
