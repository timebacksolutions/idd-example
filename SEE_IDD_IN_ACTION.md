# See IDD in action

A hands-on tour of this worked example. Each step is a command you can run against
this repo, with the output you should expect and a note on *why it matters*. By the
end you will have composed a published standard into a local requirements graph,
watched the validator treat the two as one, walked the traceability both ways, asked
"what breaks if I change this?", and seen how an AI agent can use the same graph to
find what's **missing**.

Everything here is read-only — nothing you run changes the repo.

## Setup

```sh
pip install 'throughline>=3.11.0'
cd idd-example
```

That installs `tl`. From 3.11.0, `tl` composes the sources a project declares
itself, so every command below answers over the union of this project and the
standard it borrows.

The first `tl` command that composes the source fetches
[`throughline-asvs@v4.0.3`](https://github.com/rhodium-org/throughline-asvs) from its git
origin into a per-user cache outside this repo (`~/.cache/throughline/sources/`, or
wherever `TL_CACHE` points). Later runs reuse the cache and refetch only if the
pinned ref has moved; `TL_OFFLINE=1` composes from the cache alone.

---

## 1. Read the project the way an agent does

```sh
tl context
```

`context` prints an agent brief **generated from `throughline.toml`** — the item
types, their attributes, the link rules, the grounding roots, the composition the
project declares, and the non-goals. It
is the ground truth an AI agent should read before touching the graph: it describes
exactly the rules the validator enforces, so an agent cannot invent a capability the
tool does not have.

## 2. Compose the standard and validate the union

```sh
tl check --strict
```

```
tl check · Council housing resident service (composed throughline example)
  Items      298 live   system_requirement 281 · user_requirement 15 · intent 2
  Status     approved 298
  Links      299        implements 281 · derives_from 15 · satisfies 3
  Grounding  4/4 local non-root items trace to a root · 1/1 local delivery roots served
  Local      5 of 298 local   system_requirement 3 · intent 1 · user_requirement 1  ·  293 borrowed

tl check · 1 source(s) composed: asvs (https://github.com/rhodium-org/throughline-asvs@v4.0.3) [7507659cb0df]

0 error(s), 0 warning(s)  — composed graph is sound (strict)
```

This is the heart of composition. `tl` fetches the pinned `asvs` source,
merges it with this project's own items into **one in-memory graph**, and runs
throughline's ordinary validator over the union. The council-housing requirements
and the ASVS clauses they cite are checked together, as if they had always been one
project. Exit code `0` means the whole thing is sound.

## 3. Tell what is yours from what is borrowed

```sh
tl ls --local
```

```
INT-0001  [intent/approved]  Residents manage their council-housing account securely online
SR-0001  [system_requirement/approved]  Enforce a minimum password length of 12 characters
SR-0002  [system_requirement/approved]  Accept long passphrases without truncation
SR-0003  [system_requirement/approved]  Reject credentials found in breach corpora
UR-0001  [user_requirement/approved]  Residents authenticate securely

5 item(s) (local only · 293 borrowed item(s) across 1 source(s) not searched — drop --local to search them)
```

Plain `tl ls` lists all 298 items of the composed graph; `--local` narrows it to the
five this project owns. The union is what `check` validates; the local items are the
ones this repository holds and edits.

`tl` never pretends the graph is clean when it could not compose it. If the source
cannot be fetched — say `TL_OFFLINE=1` on a machine whose cache has never seen
`throughline-asvs@v4.0.3` — `check` stops with an error naming the source rather
than validate the local half alone. A `tl` older than 3.11.0 cannot compose at all
and reports each `asvs:` reference as `namespace-unresolved`. (Free external
references — a bare URL, a linked standard with no namespace — stay opaque and pass;
only `namespace:UID` references need the source.)

## 4. Trace a requirement up to its reason for existing

```sh
tl trace SR-0001
```

```
SR-0001  [system_requirement/approved] Enforce a minimum password length of 12 characters
├─(implements) UR-0001  [user_requirement/approved] Residents authenticate securely
│ └─(derives_from) INT-0001  [intent/approved] Residents manage their council-housing account securely online
└─(satisfies) asvs:SR-0003  [system_requirement/approved] Minimum password length
```

Every requirement can be walked back to the intent that justifies it — this is the
"why" axis. Notice the last line: the `satisfies` link resolves to the borrowed ASVS
clause, with its own title and status, because `trace` answers over the composed
union. The product control and the clause it cites sit in one tree, yet the clause
keeps its namespace: this project cites it, and never owns or renumbers it.

## 5. Trace the other way — from intent down to delivery

```sh
tl trace INT-0001 --direction in
```

```
INT-0001  [intent/approved] Residents manage their council-housing account securely online
└─(derives_from) UR-0001  [user_requirement/approved] Residents authenticate securely
  ├─(implements) SR-0001  [system_requirement/approved] Enforce a minimum password length of 12 characters
  ├─(implements) SR-0002  [system_requirement/approved] Accept long passphrases without truncation
  └─(implements) SR-0003  [system_requirement/approved] Reject credentials found in breach corpora
```

Downward coverage: does everything the intent promises actually get delivered? A
delivery root that nobody derives from is an `unserved-root` finding — the graph
tells you when a stated goal has no requirements under it.

## 6. Ask "what if I change this?" — the blast radius

```sh
tl blast UR-0001
```

```
UR-0001 — blast radius: 3 dependent item(s)
  SR-0001  [system_requirement/approved] Enforce a minimum password length of 12 characters
  SR-0002  [system_requirement/approved] Accept long passphrases without truncation
  SR-0003  [system_requirement/approved] Reject credentials found in breach corpora
```

`blast` is the impact analysis: everything that depends, transitively, on the item
you name. Before you reword or retire `UR-0001`, this is the list of requirements
that would need re-checking. (Run it on a leaf like `SR-0001` and you get `0
dependent item(s)` — nothing rests on it.)

throughline also has an *automatic* version of this. Every grounding link can carry
a `stamp` — a fingerprint of its target when the link was last confirmed. Change an
item's normative content and every dependent whose stamp no longer matches turns
**suspect** on the next `check`, so a change can never quietly invalidate the things
built on top of it. `blast` is the manual "show me the radius"; suspect-cascade is
the gate that makes you look.

## 7. See the shape of the graph

```sh
tl shape
```

```
  system_requirement  -[implements]->    user_requirement  x3
  system_requirement  -[satisfies]->     <external>        x3
  user_requirement    -[derives_from]->  intent            x1
```

Every observed `(from)-[link]->(to)` triple. The middle row is the composition
story in one line: three product requirements reach *outside* this project to
satisfy clauses of an external standard.

## 8. Publish a document and gate its freshness

```sh
tl docs          # regenerate docs/spec.md and the README badges from the graph
tl docs --check  # CI gate: fail if either is stale
```

[`docs/spec.md`](docs/spec.md) and the count badges in the README are **generated
from the graph**. The prose headings are hand-owned; the item blocks, the coverage
matrix, and the counts are injected. `docs --check` fails the build if the document
drifts from the graph, so the spec can never silently fall out of date.

## 9. Move to a different edition of the standard

Open [`throughline.toml`](throughline.toml) and change one line:

```toml
[[sources]]
namespace = "asvs"
url = "https://github.com/rhodium-org/throughline-asvs"
ref = "v4.0.3"     # ← change this to adopt a newer edition
```

Adopting a new upstream edition is a one-line change to the pin. The borrowed graph
is never edited in place; you move the `ref`, re-run `tl check --strict`, and
throughline tells you whether any clause you were relying on changed or disappeared.

---

## Using the graph to find what's *missing* (with an AI agent)

The strongest use of a composed IDD graph is not describing what exists — it is
exposing what *should* exist and doesn't. Because this project pulls the whole ASVS
catalogue in by reference, an agent has, in one graph, both **your requirements** and
**the standard's full set of controls**, joined by explicit `satisfies` links. That
makes several kinds of gap machine-findable rather than a matter of someone
remembering.

Point an agent at the repo and have it run `tl context` first (so it knows the
model and the non-goals), then reason over `tl check --format json`, `tl shape`, and
the item files. Ask it for each of these:

**1. Standard controls you adopt but don't cover.** The `asvs` source contains every
clause in the standard; your project cites only some of them with `satisfies` links.
The set difference is a list of candidate missing requirements. In this example the
product satisfies `asvs:SR-0003` (V2.1.1), `asvs:SR-0004` (V2.1.2) and `asvs:SR-0006`
(V2.1.7) — but the source also carries `asvs:SR-0005` (V2.1.3, *no password
truncation*) and the V1 architecture clauses, which **nothing** in the product
satisfies. An agent surfaces those as: *"you adopt ASVS but have no requirement
citing V2.1.3 or V1.1.x — either add one, or add a `satisfies` link if an existing
requirement already covers it."* A human then judges which it is — the tool proposes,
the person decides.

**2. Structural holes the validator already names.** `tl check` reports
`orphan` (an item that grounds to no root — scope with no justification),
`unserved-root` (a goal or intent nobody delivers — a promise with no requirements),
and `grounding-cycle` (circular justification). These are "missing link" gaps the
build already fails on; an agent reads them straight from the JSON output and
proposes the missing grounding.

**3. Under-decomposed intent.** An intent or user requirement with few or no
children is a "why" that has not yet been turned into "what". An agent can walk
`tl trace <root> --direction in`, find the thin branches, and draft candidate
user/system requirements that would flesh them out.

**4. Coverage rules you assert but haven't met.** If the project declares a
`[[rules.coverage]]` requirement — say, *every user requirement must have at least
one implementing system requirement* — `check` reports each unmet one. The agent
fills the named gaps.

### Why this is safe to let an agent do at scale

throughline is built for exactly this: a machine can generate a hundred plausible
requirements an hour. Any item an agent creates enters as `proposed` with an AI
`origin`, and stays **unratified** — failing `--strict` — until a human signs it off
with `tl ratify`. So an agent can propose every gap it finds and you are never
swamped: unbounded generation is bounded into a **ranked review queue** of candidate
missing items, each already grounded to the intent or standard clause that motivates
it. You accept the ones that are real and reject the rest, and the graph only ever
records scope a person took responsibility for.

That is the loop this example is built to demonstrate: compose the standards you care
about, let the graph make the gaps explicit, let an agent draft the fillers, and keep
a human at the ratify gate.
