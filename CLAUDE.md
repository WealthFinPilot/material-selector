# CLAUDE.md — material-selector

## Project identity

Browser-based tool for matching a client's steel sample against a reference database.
Stack: HTML/CSS/JS vanilla — zero dependencies, zero build step.
Deployed on Netlify via GitHub CI/CD (publish directory = root).
Live: https://materials-selector-wealthfinpilot.netlify.app
Repo: https://github.com/WealthFinPilot/material-selector

---

## Current state — V2 in progress

V1 is stable and deployed. V2 adds:
1. Database expansion from `V2_projet.xlsx` (sheet `Add`)
2. Two new requirement datasets from `V2_projet.xlsx` (sheet `Requis`)
3. A new "Reverse" tab with two features: grade lookup and compliance check

Do not touch `src/engine.js` or the existing main tab unless a bug is found.
Run `node test/engine.test.mjs` after any change — 44/44 + 12/12 must pass.

---

## File structure

```
material-selector/
├── index.html
├── src/
│   ├── engine.js          # Similarity engine — do not modify
│   ├── main.js            # Main tab UI
│   └── style.css          # Design system — OKLCH palette, WCAG AA
├── data/
│   ├── fe01.json          # 44 carbon steels — composition + weights
│   ├── fe30.json          # 12 stainless steels — composition + weights
│   ├── fe01_requis.json   # [V2] compliance requirements for Fe01
│   └── fe30_requis.json   # [V2] compliance requirements for Fe30
├── scripts/
│   └── extract_v2.py      # [V2] extracts Add + Requis from xlsx → JSON
├── test/
│   └── engine.test.mjs
├── docs/
│   ├── source-analysis.md
│   └── architecture.md
├── assets/
│   ├── Atome.png
│   └── Materials-Identifier.jpeg
├── V2_projet.xlsx          # Source data for V2 — never commit sensitive source files
├── .gitignore
├── netlify.toml
├── README.md
└── CLAUDE.md
```

---

## Data schemas

### fe01.json / fe30.json (updated in V2)

```json
{
  "weights": { "C": 500, "Mn": 400, "Cr": 150, "..." : 0 },
  "alloys": [
    {
      "designation": "AISI 1008",
      "norm": "AISI",
      "grade": "1008",
      "shape": ["Bars", "Pipe"],
      "type": "Nonresulfurized Carbon Steel",
      "composition": { "C": 0.10, "Mn": 0.40, "..." : 0 }
    }
  ]
}
```

`shape` and `type` are new in V2. Backfill `"shape": []` and `"type": ""` on all V1 entries.

### fe01_requis.json / fe30_requis.json (new in V2)

```json
[
  {
    "designation": "AISI 1008",
    "norm": "AISI",
    "grade": "1008",
    "shape": ["Bars"],
    "type": "Nonresulfurized Carbon Steel",
    "nb_requis": 3,
    "requirements": {
      "C": "0.10",
      "Mn": "0.30-0.50",
      "Si": "",
      "P": "0.040"
    }
  }
]
```

Requirement cell values are stored as raw strings. Parsing is done at runtime
(`parseRequirement` in `src/compliance.js`).
- Empty string → no constraint
- `"0.10"` → max constraint (parseFloat)
- `"0.30–0.50"` → range constraint. **Separator is an EN DASH (U+2013)** in the
  source, not a hyphen; the parser accepts both. Index 0 = min, index 1 = max.
- `"0.15 min"` → min constraint · `"0.13 max"` → max constraint

---

## V1 similarity algorithm (engine.js — do not modify)

```
sim(x, s) = min(x, s) / max(x, s)   ∈ [0, 1]
score(input, alloy) = Σ(w_i · sim_i) / Σw_i
```

Input = client values in % (same scale as database). No normalization.

---

## V2 compliance algorithm (implemented in `src/compliance.js`)

**Weighted boolean conformity.** Compliance is boolean per constraint: a value
either meets the spec or it does not. The original bonus/penalty model (empty →
`+0.5·w`, failures penalised by the value `max·w`) was rejected — it rewarded
data-entry volume, mixed dimensions (value-scaled penalties), let `%match` exceed
100 %, and produced a verdict that contradicted the displayed score. See the
session decision and `[[algo-decision-vba-ratio]]` rationale.

For each grade G, over every element `e` that carries a requirement (weight `w_e`):

```
pass_e  = 1 if the client value satisfies the constraint, else 0
          (a missing client value counts as 0 — not validated)
validated = Σ pass_e
verdict   = PASS  iff  validated === G.nb_requis     (every applicable constraint met)
            FAIL  otherwise
match     = Σ(w_e · pass_e) / Σ(w_e)      over applicable constraints   ∈ [0, 1]
```

A range counts as 1 requirement (not 2). `nb_requis` equals the number of
non-empty requirement cells (verified across all 330 grades).

Why weighting: a grade that only misses a trace element (Pb, `w=10`) ranks above
one that misses C (`w=500`). `match` is naturally bounded in `[0, 1]` — no cap.

**Display — top 10:**
- Keep PASS grades always; keep FAIL grades only if `match > 0`
- Sort: PASS before FAIL, then by `match` desc, then by `nb_requis` desc (tighter
  spec wins ties — all PASS grades share `match === 1`)

---

## Constraints — never violate

- Zero new dependencies. No npm, no CDN additions.
- Do not modify `src/engine.js`.
- New tab must use the same CSS variables as the existing design system.
- `scripts/extract_v2.py` is the only entry point for xlsx → JSON extraction.
- Never commit: `*.xlsm`, `*.pdf`, source databases, client data.
- All composition values must come from the source xlsx — never invent values.

---

## Execution order for V2

1. `python3 scripts/extract_v2.py` → verify JSON outputs before any UI work
2. Update `fe01.json` + `fe30.json` (backfill shape/type on existing entries)
3. Build Reverse tab — Section A (grade lookup) first
4. Build Reverse tab — Section B (compliance check)
5. Manual test: run compliance check on at least 2 known grades, verify PASS/FAIL logic
6. `node test/engine.test.mjs` (identity 203/203 + 183/183 after expansion — co-top
   ties allowed for chemically indistinguishable grades) and
   `node test/compliance.test.mjs` (compliance engine)
7. Commit with logical messages, update README if needed
