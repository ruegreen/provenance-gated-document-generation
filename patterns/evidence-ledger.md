# Evidence Ledger Pattern

The ledger is the only place a value may enter the system. A deck, report or
proposal is a render of the ledger — never a document somebody writes.

---

## The Record

Every field is one record. Four properties are mandatory; none is optional
"when known".

```
FieldRecord
  key          stable identifier, defined by the domain pack
  value        the figure or string, in the unit the registry declares
  provenance   one of: client | public | modelled | missing
  owner        the human accountable for obtaining or confirming it
  feedsTo      the set of output locations this value renders into
  capturedAt   when the value was last set or confirmed
  note         optional free text — never load-bearing
```

`feedsTo` is what makes blast radius computable. Without it the advisor cannot
rank gaps, and a reviewer cannot answer *"if this number is wrong, what else is
wrong?"*

---

## Absence Is a Record State

```
resolve(key, ledger, registry):
    if key not in registry:
        error "unknown field"          # not a missing value — a bug
    record = ledger.get(key)
    if record is null:
        return { value: null, provenance: MISSING, owner: registry[key].defaultOwner }
    return record
```

The resolver never returns undefined. A caller asking about a field that has
never been touched receives a well-formed record whose provenance is `missing`.

**Why this matters.** While absence was represented as undefined, a newly added
field was invisible to four consumers at once — the gap list, the collection
plan, the advisor and the stamping logic — because each had its own truthiness
check and each handled undefined differently. Modelling absence explicitly
collapsed four code paths into one.

---

## The Collection Plan

The collection plan is not a report generated for the user. It is the canonical
representation of *what is missing*, used by the advisor, by the readiness view,
and as the refusal payload of the finalisation gate.

```
collectionPlan(ledger, registry, gateSet):
    gaps = []
    for key in registry.keys():
        record = resolve(key, ledger, registry)
        if record.provenance != MISSING: continue
        gaps.append({
            key,
            owner:       record.owner,
            blastRadius: countDownstream(key, registry),
            blocksGate:  key in gateSet,
        })

    # Rank: gate blockers first, then by how much each unblocks,
    # then group by owner so one person gets one ask.
    gaps.sort(by: (blocksGate desc, blastRadius desc, key asc))
    return groupBy(gaps, owner)
```

`countDownstream` walks `feedsTo` plus every derived value that declares the
field as an input, so a single primitive feeding three levers and nine slides
outranks one feeding a single footnote.

### Grouping by owner is the point

An ungrouped gap list produces nine separate emails to the same person. Grouping
produces one ask per owner, ordered by what it unblocks. This is the difference
between a correct list and a usable one.

---

## Ownership

Owner is a property of the record, not of the field definition, because the same
field is owned by different people on different engagements. The registry
supplies a default owner so a never-touched field still appears on somebody's
plan rather than nobody's.

---

## What the Ledger Deliberately Does Not Store

| Not stored | Why |
|---|---|
| Readiness percentage | Derived on read; a stored score can disagree with its inputs |
| Benefit-model outputs | Derived on read from primitives |
| Stamp set | Derived per render from provenance |
| Collection plan | Derived on read |
| `superseded` flag on exports | Derived from the existence of a higher version |

The rule: **if it can be recomputed from primitives, it is not stored.** A stored
derivation is a number that can silently disagree with the values it came from.

---

## Concurrency Note

The account ledger is persisted as a single document and updated with a whole-
document read-modify-write. Two concurrent editors on one account will overwrite
each other. This is a known and accepted limitation at the intended scale of one
engineer per account, documented rather than discovered.

A field-level update path would resolve it, at the cost of making partial writes
— and therefore partially consistent derivations — possible. That trade was
declined deliberately.
