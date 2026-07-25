# idd-example — specification

A worked throughline **consumer**: a council-housing self-service account whose own
requirements are grounded locally and, where they restate a security control, linked
by `satisfies` to a clause borrowed from the `asvs` source
([`throughline-asvs@v4.0.3`](https://github.com/rhodium-org/throughline-asvs)).

The item blocks below are **generated from the graph** by `tl-compose docs`; the
section headings are the only hand-owned structure. Regenerate with `tl-compose docs`
and gate freshness in CI with `tl-compose docs --check`.

## Intent

<!-- tl:item INT-0001 -->
**INT-0001 — Residents manage their council-housing account securely online** — `intent`, status `approved`

> A resident of the council can view their tenancy, report repairs and update their contact details through a self-service account, trusting that access to that account is protected to a recognised security baseline rather than to whatever the delivery team happened to think of.
<!-- tl:end -->

## User requirements

<!-- tl:item UR-0001 -->
**UR-0001 — Residents authenticate securely** — `user_requirement`, status `approved`

> A resident signs in with credentials that are held to the same strength, length and breach-resistance controls a security assessor would expect, so that an account cannot be trivially guessed or reused from a leaked password set.

*Derives from:* INT-0001
<!-- tl:end -->

## System requirements

<!-- tl:item SR-0001 -->
**SR-0001 — Enforce a minimum password length of 12 characters** — `system_requirement`, status `approved`

> Registration and password-change flows reject any password shorter than 12 characters (after consecutive spaces are collapsed), enforced server-side.

*Implements:* UR-0001
*Satisfies:* asvs:SR-0003 (V2.1.1)
<!-- tl:end -->

<!-- tl:item SR-0002 -->
**SR-0002 — Accept long passphrases without truncation** — `system_requirement`, status `approved`

> The account service accepts passwords of at least 64 characters, rejects only those beyond 128 characters, and never silently truncates before hashing, so a resident using a passphrase or password manager is not quietly weakened.

*Implements:* UR-0001
*Satisfies:* asvs:SR-0004 (V2.1.2)
<!-- tl:end -->

<!-- tl:item SR-0003 -->
**SR-0003 — Reject credentials found in breach corpora** — `system_requirement`, status `approved`

> At registration and password change, the chosen password is checked against a set of known-breached passwords and rejected on a match, so a resident cannot pick a credential already circulating in public leak sets.

*Implements:* UR-0001
*Satisfies:* asvs:SR-0006 (V2.1.7)
<!-- tl:end -->

## Coverage

Every local system requirement grounds up to a user requirement, and each restated
security control cites the ASVS clause it satisfies:

<!-- tl:matrix outgoing:implements type == 'system_requirement' -->
| UID | Title | Implements (outgoing) |
|---|---|---|
| SR-0001 | Enforce a minimum password length of 12 characters | UR-0001 |
| SR-0002 | Accept long passphrases without truncation | UR-0001 |
| SR-0003 | Reject credentials found in breach corpora | UR-0001 |
<!-- tl:end -->
