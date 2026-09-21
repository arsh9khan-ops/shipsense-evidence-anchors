# ShipSense evidence anchor log — field schema and corrections

This repository is an append-only, third-party-hosted log of the heads of ShipSense's
`evidence_audit_chain` and (from schema_version 4) `commitment_log`. It contains
cryptographic hashes and counts only. It contains no shipment, merchant, courier or
consignee data, and never has.

Nothing in this repository is ever deleted or rewritten, including the entries corrected below.
A log that edits its own history proves nothing.

## OpenTimestamps upgrades — from 2026-09-21 (ots-upgrade v5)

A calendar's first reply is a PENDING attestation: a promise to put the anchor file's sha256
into a Bitcoin transaction. ShipSense's upgrader comes back for the completed proof.

| file | meaning |
|---|---|
| `<anchor>.json.ots` | a standard OpenTimestamps proof file for the anchor `.json` beside it, written ONCE, when the first calendar's Bitcoin attestation arrives. Every calendar's proof as it stood then is merged into it; a calendar still waiting appears as a pending attestation that `ots upgrade` can complete. |
| `<anchor>.proofs.upgraded-<YYYY-MM-DDTHHMMSSZ>.json` | appended at each change of the proofs, never rewritten (before 2026-09-21 the stamp stopped at the minute: `…THHMMZ`). |

Each calendar in an upgraded file carries `proof_b64` (its best proof so far),
`pending_proof_b64` (its ORIGINAL pending reply, kept once a newer proof replaces it — before
2026-09-21 the original was overwritten in ShipSense's database, but it is still in the
`<anchor>.proofs.json` published beside the anchor), `status` (`pending_attestation`,
`upgraded_no_bitcoin_yet`, `bitcoin_attested`, `unattestable` = no Bitcoin attestation 14 days
after the anchor, or a submission error such as `http_503`), `bitcoin_block_height`, and
`bitcoin_attestations[]` (`height`, `block_merkle_root` as block explorers print it).

**Before 2026-09-21 the upgrader stopped asking an anchor's other calendars once one had
attested.** On 2026-09-21 every calendar of every anchor carrying proofs was asked again; the
`.ots` and upgraded files for older anchors are dated when they were written, and the Bitcoin
blocks inside them date the anchors.

To verify: download the anchor `.json` and its `.json.ots` and run `ots verify <anchor>.json.ots`
(opentimestamps-client, which reads block headers from a Bitcoin node); or, with no node, check
that each Bitcoin attestation's message is the merkle root of its block at any block explorer.

## proof_schema_version 5 — proofs files from 2026-09-21

The anchor files are unchanged (schema_version 4). Each `<anchor>.proofs.json` gains:

| field | meaning |
|---|---|
| `proof_schema_version` | 5 |
| `timestamp_authorities[]` | every RFC-3161 authority asked: `tsa_url`, `label`, `status`, `pki_status`, `token_present`, `imprint_matches`, and the TimeStampResp (`tsr_b64`, base64 DER) whenever one arrived. `status` is `granted` only when the PKIStatus is 0 or 1 AND a signed token is present AND that token carries this file's sha256; otherwise it names what was missing (`rejected`, `waiting`, `no_token`, `imprint_mismatch`, `malformed`, `http_<n>`, `timeout`). From 2026-09-21 the authorities are freetsa.org and DigiCert, a CA-audited authority. |
| `rfc3161` | kept for schema-4 readers: the first granted authority's response |
| `opentimestamps[].pending_uri` | the calendar that will answer for that pending attestation |
| `witness_source` | `anchor_witnesses` (the configured list) or `fallback_defaults` (the list could not be read) |
| `anchor_document` | present only when the anchor file could not be published: the exact bytes the witnesses stamped |

**Before 2026-09-21 an RFC-3161 answer was recorded as `ok` on HTTP 200 alone; its PKIStatus
was not read.** A timestamp authority can answer HTTP 200 with a rejection and no token. Verify
any `rfc3161` proof yourself with `openssl ts -verify` rather than trusting its `status`.

To verify a token with openssl, give it the certificates the token itself carries — `ts -verify`
does not use them unless told to: `openssl ts -reply -in x.tsr -token_out -out x.tst`, then
`openssl pkcs7 -inform DER -in x.tst -print_certs -out chain.pem`, then
`openssl ts -verify -in x.tsr -digest <anchor_file_sha256> -CAfile <public roots> -untrusted chain.pem`.
DigiCert's tokens chain to DigiCert Assured ID Root CA, a public root (checked 2026-09-21:
`Verification: OK`); freetsa.org's chain to freetsa.org's own root certificate.

## schema_version 4 — from 2026-08-17

Everything in schema_version 3, plus:

| field | meaning |
|---|---|
| `commitment_log.head_seq` | newest seq in ShipSense's commitment_log at that instant |
| `commitment_log.head_entry_hash` | `entry_hash` of that entry |
| `commitment_log.row_count` | exact `count(*)` of commitment_log |
| `commitment_log.digest` | `sha256` of every `entry_hash` joined by `|` in ascending `seq` order |
| `merkle.tree_size` | leaves in the Merkle tree over evidence_audit_chain (from 17 Aug 2026) |
| `merkle.root_hash` | RFC-6962-style root: leaf sha256(0x00||bytes(row_hash)), node sha256(0x01||L||R), odd node promoted |

Beside each anchor file, a `<anchor>.proofs.json` file may carry external timestamp
proofs of the anchor file itself:

| field | meaning |
|---|---|
| `anchor_file_sha256` | sha256 of the exact bytes of the anchor .json file |
| `opentimestamps[]` | per-calendar pending attestations (base64). Complete after Bitcoin confirmation via the `ots` client (upgrade against the same calendar). |
| `rfc3161` | TimeStampResp (base64, DER) from the named TSA. Verify: `openssl ts -verify`. |

Why: the GitHub commit clock is controlled by ShipSense's own account. The
OpenTimestamps attestation (Bitcoin) and the RFC-3161 token are clocks nobody here
controls. Together, one artefact carries three independent timestamps.

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
6. (v4) Recompute sha256 of the anchor file and check it against the OpenTimestamps and
   RFC-3161 proofs in `<anchor>.proofs.json` — timestamps no ShipSense account controls.
7. (v4.1) For a single chain row, ask ShipSense for its Merkle inclusion proof and fold it
   from the leaf up — it must reproduce `merkle.root_hash` in this file.

## CORRECTION — schema_version 2 (anchors of 2026-07-15 to 2026-08-16, ids 1..22)

Version 2 files carry a field named `rows_in_chain`. **That field does not hold a row count.**
It holds the head row id. The generator assigned the last row's `id` to a field named for a count.

On 2026-08-16 the v2 anchor reported `rows_in_chain: 697` when the chain held **79** rows.

The error is confined to that one field. `head_row_id`, `head_row_hash` and `prev_anchor_ref`
in every v2 file were and remain correct, and every previously anchored head row still exists in
the chain with an unchanged hash (re-verified 2026-08-16 across all 22 anchors).

**Read `rows_in_chain` in any v2 file as `head_row_id`. It duplicates that field; it adds nothing.**

Corrected by `anchor-chain-head@v3`, 2026-08-16.
