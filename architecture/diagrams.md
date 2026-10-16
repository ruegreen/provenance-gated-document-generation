# Architecture Diagrams

Detailed diagrams for the provenance-gated document generation platform. All
diagrams describe design and control flow; none describe implementation
internals.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Client["Single-Page Client"]
        LEDGER["Ledger Editor"]
        ADVISOR["Collection Advisor"]
        REVIEW["Readiness Review"]
        EXPORTS["Export History"]
    end

    subgraph API["Application Layer"]
        subgraph SpineBox["Spine — domain-agnostic"]
            RANK["Provenance Ranking"]
            READY["Readiness Derivation"]
            SESS["Session Record"]
            VERS["Export Versioning"]
        end
        subgraph PackBox["Domain Pack — pluggable"]
            REG["Field Registry"]
            UNITS["Units and Formatting"]
            MODEL["Benefit Model"]
        end
        PLAN["Render Plan"]
        GATE["Finalisation Gate"]
    end

    subgraph RenderSvc["Render Service"]
        CONTRACT["Slot Contract"]
        APPLY["Operation Applier"]
        VALID["Post-Render Validator"]
    end

    subgraph Store["Persistence"]
        ACCT[("Account Ledger")]
        EXP[("Export Versions")]
        TPL[("Hash-Addressed Templates")]
    end

    LEDGER & ADVISOR & REVIEW & EXPORTS --> API
    RANK --> READY --> PLAN
    REG -.->|"passed as argument,<br/>never imported"| RANK
    MODEL --> PLAN --> GATE
    GATE -->|refused| ADVISOR
    GATE -->|passed| CONTRACT --> APPLY --> VALID --> EXP
    SpineBox <--> ACCT
    CONTRACT <--> TPL
```

---

## 2. Provenance Resolution — Weakest Link

How a derived value acquires its provenance grade. The rule is mechanical: the
derived value is never better-evidenced than its worst input.

```mermaid
flowchart TD
    START["Derived value requested"] --> INPUTS["Collect declared inputs"]
    INPUTS --> LOOP{"For each input:<br/>record exists?"}
    LOOP -->|no| MISSING["source = missing"]
    LOOP -->|yes| READ["source = record.provenance"]
    MISSING --> COLLECT["Accumulate sources"]
    READ --> COLLECT
    COLLECT --> WORST["worstOf(sources)<br/>client &lt; public &lt; modelled &lt; missing"]
    WORST --> CHECK{"worst == missing?"}
    CHECK -->|yes| BLOCK["Value blocked.<br/>Field added to collection plan.<br/>Owner recorded."]
    CHECK -->|no| GRADE["Value computed.<br/>Grade = worst source."]
    GRADE --> STAMP{"Grade == modelled?"}
    STAMP -->|yes| MARK["Slides fed by this value<br/>require an illustrative stamp"]
    STAMP -->|no| CLEAR["Slides fed by this value<br/>require an empty stamp"]
```

> **Design note.** The `missing` branch is a first-class source value, not the
> absence of a record. While absence was represented as undefined, new fields
> were simultaneously invisible to the gap list, the collection plan, the advisor
> and the stamping logic — four bugs with one cause.

---

## 3. The Finalisation Gate

The gate refuses on a named set of fields and returns the collection plan as its
refusal payload, so the answer to *why was this refused* and the answer to *what
do I do next* are the same object.

```mermaid
flowchart TD
    REQ["Finalise requested"] --> VAC{"Pack declares<br/>any value inputs?"}
    VAC -->|no| VACUOUS["REFUSE: vacuous pack.<br/>A gate with nothing to check<br/>must not emit a pass."]
    VAC -->|yes| SET["Assemble named set:<br/>• every benefit-model input<br/>• avoidable shares<br/>• realisation factor<br/>• investment profile"]
    SET --> EVAL{"Any field in the set<br/>resolving to missing?"}
    EVAL -->|yes| REFUSE["REFUSE with named fields"]
    REFUSE --> PLANOUT["Return collection plan:<br/>grouped by owner,<br/>ranked by blast radius"]
    EVAL -->|no| PASS["PASS"]
    PASS --> SNAP["Capture input snapshot"]
    SNAP --> HASH["SHA-256 over<br/>canonicalised snapshot"]
    HASH --> VERSION["Allocate version n+1"]
    VERSION --> RENDER["Hand to render service"]
```

---

## 4. Render Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API
    participant P as Render Plan
    participant G as Gate
    participant R as Render Service
    participant S as Store

    C->>A: Request export (draft or final)
    A->>A: Recompute derived values from primitives
    A->>P: Build render plan
    P->>P: Resolve all slide numbering (single owner)
    P->>P: Resolve stamp set per slide
    alt final
        A->>G: Evaluate named-set gate
        G-->>C: Refusal + collection plan (if blocked)
    end
    A->>R: Plan + template hash
    R->>R: Verify contract hash == template hash
    alt mismatch
        R-->>A: REFUSE (not a warning)
    end
    R->>R: Apply operations (value slots + furniture + stamps)
    R->>R: Validate: every planned op applied?
    R-->>A: Rendered artifact + operation report
    A->>S: Store artifact, snapshot hash, version n+1
    S-->>C: Export record (supersession derived, not flagged)
```

---

## 5. Export Versioning and Supersession

```mermaid
graph LR
    subgraph Account["One Account"]
        V1["Export v1<br/>hash: a1b2…<br/>finalised"]
        V2["Export v2<br/>hash: c3d4…<br/>finalised"]
        V3["Export v3<br/>hash: e5f6…<br/>finalised"]
    end
    LEDGER1["Ledger change"] --> V2
    LEDGER2["Ledger change"] --> V3
    V1 -.->|"superseded = derived<br/>from existence of v2"| V2
    V2 -.->|"superseded = derived<br/>from existence of v3"| V3
    V3 --> CURRENT["current = highest version"]
```

> A finalised export is never regenerated in place. `superseded` is derived from
> the version sequence rather than written onto the older record, because a flag
> that must be set is a flag that will be missed.

---

## 6. Spine and Pack Boundary

```mermaid
graph TB
    subgraph Spine["Spine — knows how, never what"]
        S1["Provenance ranking"]
        S2["Readiness derivation"]
        S3["Collection plan assembly"]
        S4["Export versioning"]
        S5["Session recording"]
    end

    subgraph PackA["Domain Pack A — production"]
        A1["~24 gated fields"]
        A2["Benefit model levers"]
        A3["39-slide template contract"]
    end

    subgraph PackB["Domain Pack B — minimal, test only"]
        B1["Deliberately thin registry"]
        B2["Exists to keep the<br/>boundary honest"]
    end

    PackA -->|"registry passed<br/>as argument"| Spine
    PackB -->|"registry passed<br/>as argument"| Spine
    Spine -.->|"MUST NOT import"| PackA
    Spine -.->|"MUST NOT import"| PackB
    B2 ==>|"exposed the<br/>vacuous-gate bug"| S3
```

---

## 7. Slot Contract Coverage

Scale of the render target, and the deliberate limits of what the contract
addresses.

```mermaid
pie showData
    title "Text-bearing shapes and cells in the 39-slide template"
    "Addressed — value slots" : 45
    "Addressed — furniture strings" : 96
    "Not addressed (renders as authored)" : 1126
```

| Measure | Count |
|---|---|
| Slides in template | 39 |
| Slides carrying any generated value | 16 |
| Text-bearing shapes and cells | 1,267 |
| Contract-addressed entries | 141 |
| — value slots | 45 |
| — furniture strings | 96 |
| Fields in the finalisation gate | 24 |

> The 1,126 unaddressed shapes are the reason the template is described as
> domain-shaped rather than branded. Much of that text carries industry-specific
> vocabulary, so a different industry needs a different pack and template, not
> more holes in this one.
