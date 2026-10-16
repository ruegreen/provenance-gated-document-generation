# Provenance Model Pattern

Provenance is a property of the value, not an annotation beside it. Every design
consequence in this system follows from that choice.

---

## The Four Sources

```
CLIENT   = 0   # customer supplied or confirmed
PUBLIC   = 1   # published filing, report, disclosure
MODELLED = 2   # derived by the account team from assumptions
MISSING  = 3   # no record exists
```

Ordered worst-last so that "worst" is a maximum and the comparison is total. A
partial order would require a tie-break rule, and any tie-break rule here is an
invitation to argue the grade upward.

### Why only four

Earlier drafts had `verified`, `estimated`, `assumed`, `derived` and
`placeholder`. Every added state produced a boundary case that two reasonable
engineers graded differently, and a grade that depends on the grader is not a
grade. Four states survive the test *"could a reviewer dispute this label?"*
— the answer is no for all four, because each names a **source**, not a
confidence level.

---

## Weakest Link

```
worstOf(sources):
    return max(sources)        # ordering is worst-last

gradeOf(derivedValue, registry, ledger):
    inputs  = registry[derivedValue].declaredInputs
    sources = [ resolve(i, ledger, registry).provenance for i in inputs ]
    if sources is empty:
        error "derived value declares no inputs"   # see vacuous check
    return worstOf(sources)
```

A roll-up built from nine `client` figures and one `modelled` figure grades
`modelled`, and the slides it feeds are stamped accordingly. This is the single
rule that stops derivation from laundering confidence.

### It must be mechanical

The alternative — asking the author to judge whether the modelled input "really
affects" the total — fails every time, because the author is the person least
able to see it and the most motivated to say no.

---

## Absence Is a Source

`MISSING` is a value in the same enum, not the absence of one. Consequences:

- A derived value with a missing input grades `MISSING` and is **blocked**, not
  computed with a zero or a default.
- A missing field appears on the collection plan automatically, with an owner,
  because the resolver returns a well-formed record.
- Stamping logic sees missing fields and can report a slide that would print a
  blocked value.
- The gate can name the absent fields, because they exist as records to name.

```
isBlocked(derivedValue):
    return gradeOf(derivedValue) == MISSING
```

---

## Stamping

A stamp is a rendered consequence of a grade, computed per slide, on every
render.

```
stampsFor(renderPlan, registry, ledger):
    stamps = {}                           # slideNumber -> text | ""
    for slide in renderPlan.slides:
        grades = [ gradeOf(v) for v in valuesOn(slide) ]
        if grades is empty:
            stamps[slide] = ""            # authored slide, no claim made
        elif worstOf(grades) >= MODELLED:
            stamps[slide] = ILLUSTRATIVE_TEXT
        else:
            stamps[slide] = ""            # evidenced: stamp cleared, not skipped
    return stamps
```

**Every stamp shape is written on every render**, including the empty case.
Originally stamps were pre-printed on 22 of 39 slides and nothing ever removed
them, so a fully evidenced deck carried an illustrative stamp on slides whose
every number was confirmed — and the stamp count read identically for every
account, which made it meaningless. A stamp that is always present is not a
signal.

If a slide needs a stamp and the contract has no stamp shape for it, that is
**reported in the operation report**, not silently skipped.

---

## Colour as a Second Carrier

Because provenance is displayed as colour, colour is data and is validated as
data. See [accessibility-validation.md](accessibility-validation.md). Two
additional carriers are always present:

1. Every provenance chip prints its own text label.
2. The two states carrying the most decision weight carry a texture fill.

Colour is never the sole channel for a provenance distinction.
