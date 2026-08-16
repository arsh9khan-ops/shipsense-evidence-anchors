# ShipSense evidence anchor log — field schema and corrections

This repository is an append-only, third-party-hosted log of the head of ShipSense's
`evidence_audit_chain`. It contains cryptographic hashes and counts only. It contains no
shipment, merchant, courier or consignee data, and never has.

Nothing in this repository is ever deleted or rewritten, including the entries corrected below.
A log that edits its own history proves nothing.

## schema_version 3 — from 2026-08-16

| field | meaning |
|---|---|
| `schema_version` | 3 |
| `anchored_at` | UTC instant the anchor was published |
| `head_row_id` | primary key of the newest row in the chain at that instant |
| `head_row_hash` | `row_hash` of that row |
| `chain_row_count` | exact `count(*)` of rows in the chain at that instant |
| `chain_digest` | `sha256` of every `row_hash` joined by `|` in ascending `id` order |
| `segment_verified_from_row_id` | first row id re-verified on this run |
| `segment_rows_verified` | rows re-verified on this run (linkage + hash recompute) |
| `prev_anchor_ref` | git commit SHA of the previous anchor, chaining the log itself |

**Row ids are a sparse sequence.** They come from a Postgres sequence that advances on rolled-back
inserts, so `head_row_id` is always >= `chain_row_count` and usually much larger. On 2026-08-16
the chain held 79 rows spanning ids 1 to 697. **A gap in ids is not a missing row.**

### How to check an anchor yourself

1. Ask ShipSense for the chain rows (id, prev_hash, payload_sha256, row_hash, hash_input_ts).
2. Confirm `prev_hash` of each row equals `row_hash` of the previous row; row 1 is `GENESIS`.
3. Recompute each row: `sha256(prev_hash | payload_sha256 | dispute_id | hash_input_ts)`.
4. Recompute `chain_digest` and compare it to the value in the anchor file for that date.
5. Confirm the file's git commit predates the date on which the evidence was disputed.

## CORRECTION — schema_version 2 (anchors of 2026-07-15 to 2026-08-16, ids 1..22)

Version 2 files carry a field named `rows_in_chain`. **That field does not hold a row count.**
It holds the head row id. The generator assigned the last row's `id` to a field named for a count.

On 2026-08-16 the v2 anchor reported `rows_in_chain: 697` when the chain held **79** rows.

The error is confined to that one field. `head_row_id`, `head_row_hash` and `prev_anchor_ref`
in every v2 file were and remain correct, and every previously anchored head row still exists in
the chain with an unchanged hash (re-verified 2026-08-16 across all 22 anchors).

**Read `rows_in_chain` in any v2 file as `head_row_id`. It duplicates that field; it adds nothing.**

Corrected by `anchor-chain-head@v3`, 2026-08-16.
