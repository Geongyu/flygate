---
name: flygate-evidence-critic
description: >
  Use this skill when a claim is made from molecular docking, predicted affinity, or
  pharmacovigilance disproportionality output and that claim must be checked before it
  reaches a person. Invoke whenever the user mentions docking scores, AutoDock Vina,
  DiffDock confidence, Boltz-2 predicted affinity, OpenFold3 structures, PRR, ROR,
  FAERS signal detection, WHO-UMC or Naranjo causality, or asks whether a result
  supports a conclusion. The skill does not produce results; it decides what the
  results are allowed to say.
license: Apache-2.0
compatibility: "python>=3.11, requests>=2.28"
allowed-tools: Bash, Read, Write
---

# FlyGate Evidence Critic

Three gates run in order on every claim. Gates 1 and 2 use no model. Gate 3 uses one.
A claim whose evidence ids and numbers are both correct can still be rejected at gate 3.

```
claim -> [1] evidence id attached? -> [2] numbers match the log? -> [3] does the
reasoning stay inside what the evidence allows? -> PASS / REJECT / HUMAN
```

## Gate 1 — evidence id

Every quantitative claim carries an id pointing at a stored run (`vina:<target>/<compound>`,
`diffdock:<pdb>/<compound>`, `boltz2:<target>/<compound>`, `faers:<drug>/<event>`,
`dailymed:<brand>#<section>`, `chembl:<molecule>/<target>`). No id, reject here. Do not
run gates 2 and 3.

## Gate 2 — number check

Re-read each number from the stored run and compare character by character. Vina scores
come from `REMARK VINA RESULT` lines, not from stdout, which rounds. Report any mismatch
with both values. Do not repair the claim.

## Gate 3 — reasoning check

Give the model the evidence log and the rule list below, and ask for one verdict line.
Listing or ranking values that are in the log is always PASS. Reject only when the
reasoning exceeds the evidence.

| Rule | Reject when the claim … |
|---|---|
| `cross-target` | compares docking scores across different target proteins, or infers selectivity from them |
| `confidence≠affinity` | reads DiffDock confidence as binding strength |
| `score≠efficacy` | treats a docking score as measured affinity or clinical effect |
| `redock≠prospective` | uses redocking of a co-crystal ligand as evidence of prospective prediction |
| `cross-dock` | treats an exploratory cross-docking pose as evidence of real binding |
| `predicted≠measured` | calls a Boltz-2 pIC50 a measurement |
| `plddt≠affinity` | reads a structure confidence score as binding strength |
| `n<8` | computes a correlation over fewer than eight paired points |
| `species` | carries a non-human protein result to humans without support |
| `prr≠causal` | reads PRR or ROR as causality rather than reporting disproportionality |
| `indication` | counts the treated disease, or its symptoms, as an adverse event |
| `progression` | counts underlying disease worsening as a drug effect |
| `pharmacodynamic` | counts a monitoring lab value of the drug's intended action as an adverse event |
| `artifact` | counts a reporting-form term (off label use, drug ineffective, hospitalisation, fall) as an adverse event |
| `label-match` | concludes "not in the label" from a plain string search without spelling and synonym normalisation |

## Verdicts

- `PASS` — inside the evidence.
- `REJECT` — name the rule and the missing evidence. Never rewrite the claim.
- `HUMAN` — evidence is correct but the judgment belongs to a qualified reviewer:
  death or other serious outcome, causality scales disagreeing, two tools disagreeing.

## Causality claims

When a claim assigns causality to a drug and an adverse event, report WHO-UMC and Naranjo
side by side and state which Naranjo items are unanswerable. Public FAERS has no narrative,
so rechallenge, drug level, and dose-response cannot be scored. A Naranjo total computed
over the remaining items is not comparable to one computed over all ten. If the two scales
land in different categories, the verdict is `HUMAN`, not a choice between them.

## Reference model

Gate 3 was measured with `nvidia/nemotron-3-super-120b-a12b` at temperature 0. Reasoning
models need room: below roughly 800 output tokens they exhaust the budget before emitting
the verdict line and the run reads as a failure that is really a truncation. See
`BENCHMARK.md`.

## Output

```json
{"claim": "...", "gate1": "pass|reject", "gate2": "pass|reject",
 "gate3": "pass|reject|human", "verdict": "PASS|REJECT|HUMAN",
 "rule": "cross-target", "evidence": ["vina:parp1-4r6e/niraparib"], "reason": "<=25 words"}
```
