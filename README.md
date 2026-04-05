# Data Preparation Report for NF*v3 datasets

## Scope

This report documents the completed data preparation process for two NetFlow-based intrusion datasets:

- `NF-UNSW-NB15-v3`
- `NF-CICIDS2018-v3`

The objective of the process was to produce reproducible, cleaned, deduplicated, and class-controlled datasets suitable for downstream machine learning experiments.

---

## Methodological Justification

### Rationale for Downsampling

Downsampling was applied to address extreme class-prior imbalance, particularly the dominance of benign traffic and high-volume attack families in CICIDS2018. In heavily imbalanced intrusion datasets, empirical risk minimization can be biased toward majority classes, yielding high aggregate accuracy while under-detecting minority attack modes. This effect is especially problematic when the operational objective is attack sensitivity rather than majority-class fidelity.

The procedure is strictly downsample-only (no oversampling or synthetic generation). In UNSW, benign is downsampled while attack observations are retained; in CICIDS2018, benign and oversubscribed attack classes are downsampled under explicit constraints. This design preserves empirical observations and improves computational tractability while maintaining low-frequency attack evidence.

### Dataset Sampling Justification

Sampling was specified separately for each dataset because the empirical class structures and operational constraints differ.

### Formal Problem Formulation

Let:

- `C` denote the set of classes.
- `F` denote the set of families.
- `n_c` denote the available (cleaned, deduplicated) count for class `c`.
- `m_c` denote the selected count for class `c` in the final dataset.
- `N = sum_{c in C} m_c` denote final dataset size.
- `B` denote the benign class.
- `S` denote the set of fixed-minority classes with requested lower bounds `r_c`.

All selections satisfy integer and feasibility constraints:

- `m_c in Z_{>=0}`
- `0 <= m_c <= n_c` (downsample-only; no oversampling)

#### NF-UNSW-NB15-v3

Define total retained attack count as `A = sum_{c != B} n_c`, and set target binary parity `alpha = 0.5`.

The design objective is to preserve all attack observations while enforcing exact binary balance:

`m_c = n_c, for all c != B`

`m_B = round((alpha / (1 - alpha)) * A)` with `alpha = 0.5`

This reduces to `m_B = A` and thus `N = 2A`. The resulting dataset is 255,358 rows with exact 50:50 benign-to-attack parity. This formulation maximizes attack retention while balancing class representation.

#### NF-CICIDS2018-v3

For CICIDS2018, selection was formulated as constrained class allocation with binary parity and family-balance pressure.

Primary constraints:

- `sum_{c in B} m_c = sum_{c in A} m_c` (binary parity),
- `m_B >= 0.5 * N_total` (minimum benign support),
- `m_c >= min(r_c, n_c), for c in S` (minority preservation),
- `0 <= m_c <= n_c, for all c in C`.

Define attack-family mass:

- `M_f = sum_{c in C_f} m_c`, for attack families `f in F_attack`.

Allocation rule in implementation:

- initialize each family with fixed-minority mass and feasible class caps;
- iteratively distribute remaining attack budget as evenly as possible across unsaturated families;
- when a family reaches its cap, remove it from further allocation and continue;
- split indivisible remainder deterministically.

This is a capped progressive-filling (max-min fairness) allocation under hard feasibility constraints. It yields near-equal family support without violating class availability, and directly suppresses dominance by high-volume families (especially DoS/DDoS) while preserving low-frequency evidence.

### Rationale for Class-to-Family Mapping

Class mapping was retained to support semantic comparability and analytical coherence across datasets with different label granularity and naming conventions. Family-level mapping serves three methodological functions:

1. It reduces nomenclature variance (e.g., class-name heterogeneity for conceptually related attacks).
2. It increases effective sample support per target category when individual fine-grained classes are sparse.
3. It provides a defensible cross-dataset abstraction layer for transfer and comparative evaluation.

Mappings were constrained to source-defined categories and explicit project taxonomy decisions documented in repository map files. This ensures traceability and prevents undocumented post hoc relabeling.

### Validity, Scope, and Reproducibility

The resulting datasets are intended for controlled model development and comparative evaluation, not for direct estimation of deployment prevalence. Because resampling modifies class priors, prevalence-sensitive metrics should be interpreted alongside prior-independent measures and class-conditional performance. To preserve auditability, this repository reports both initial and final counts, keeps mapping files explicit, and fixes stochastic operations via a deterministic random seed.

---

## Processing Protocol

The following procedure was applied in both pipelines.

1. **Schema validation**
   - All records were validated against the expected NetFlow v3 schema (55 columns).
   - Rows were processed only when schema conformity was satisfied.

2. **Text normalization**
   - Text fields were canonicalized by trimming whitespace and normalizing null-like tokens.
   - Attack labels were normalized to canonical class strings before any class-level operations.

3. **Numeric coercion and finite-value enforcement**
   - Numeric fields were coerced using strict numeric conversion.
   - Non-finite values (`±inf`) were converted to missing values (`NaN`) and tracked.

4. **Binary label consistency enforcement**
   - `Label` was enforced as: `0` for `Benign`, `1` for all attack classes.
   - Any mismatch between source `Label` and `Attack` semantics was corrected.

5. **Temporal consistency checks**
   - Start/end timestamps were repaired where ordering was invalid.
   - Flow duration was recomputed or imputed from timestamps when required.

6. **Pairwise ordering constraints**
   - The following lower/upper pairs were repaired when violated:
     - `MIN_TTL` / `MAX_TTL`
     - `SHORTEST_FLOW_PKT` / `LONGEST_FLOW_PKT`
     - `MIN_IP_PKT_LEN` / `MAX_IP_PKT_LEN`

7. **Zero-duration throughput repair**
   - For zero-duration flows, non-finite directional rate fields were set to `0.0`.

8. **Range and sign constraints**
   - Non-negative constraints were enforced on count/size/duration features.
   - Bounded fields were clipped to protocol-valid ranges (ports, protocol IDs, TTL, flags, DNS ranges, and binary label bounds).

9. **Missing-value completion**
   - Remaining missing values were imputed deterministically:
     - median imputation for float fields
     - rounded median imputation for integer fields

10. **Exact duplicate removal**
    - Exact row duplicates were identified and removed.
    - Duplicate statistics were retained in the summary reports.

11. **Identifier and timestamp drop policy**
    - The following columns were removed from final exports:
      - `IPV4_SRC_ADDR`
      - `IPV4_DST_ADDR`
      - `FLOW_START_MILLISECONDS`
      - `FLOW_END_MILLISECONDS`

12. **Deterministic sampling**
    - Randomized operations used `random_state = 0` for reproducibility.

---

## Data Products

- `NF-UNSW-NB15-v3/dataset.csv`
- `NF-UNSW-NB15-v3/preparation_report.json`
- `NF-UNSW-NB15-v3/map.json`
- `NF-CICIDS2018-v3/dataset.csv`
- `NF-CICIDS2018-v3/preparation_report.json`
- `NF-CICIDS2018-v3/map.json`

---

## References

- UQ NIDS dataset portal: `https://staff.itee.uq.edu.au/marius/NIDS_datasets/`
- NF-UNSW-NB15-v3 metadata record: `https://researchdata.edu.au/nf-unsw-nb15-v3/3485172`
- Temporal analysis of NetFlow datasets (NFv3 context): `https://arxiv.org/abs/2503.04404`
- IANA protocol numbers registry: `https://www.iana.org/assignments/protocol-numbers`
- RFC 791 (IPv4 header fields): `https://www.rfc-editor.org/rfc/rfc791`
- RFC 6335 (port namespace and ranges): `https://www.rfc-editor.org/rfc/rfc6335`

---

## Citation

For scholarly use of the NetFlow v3 temporal analysis reference, please cite:

```bibtex
@misc{luay2025NetFlowDatasetsV3,
  title = {Temporal Analysis of NetFlow Datasets for Network Intrusion Detection Systems},
  author = {Majed Luay and Siamak Layeghy and Seyedehfaezeh Hosseininoorbin and Mohanad Sarhan and Nour Moustafa and Marius Portmann},
  year = {2025},
  eprint = {2503.04404},
  archivePrefix = {arXiv},
  primaryClass = {cs.LG},
  url = {https://arxiv.org/abs/2503.04404}
}
```
