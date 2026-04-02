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

### Dataset-Specific Sampling Justification

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

Define total retained attack count as `A = sum_{c != B} n_c`, and set benign-prior target `alpha = 1/3`.

The design objective is to preserve all attack observations while setting benign prevalence near `alpha`:

`m_c = n_c, for all c != B`

`m_B = round((alpha / (1 - alpha)) * A)`

With `alpha = 1/3`, this reduces to `m_B = round(A / 2)` and thus `N = A + m_B ~ 1.5A`. This formulation explicitly maximizes attack retention under a controlled benign prior.

#### NF-CICIDS2018-v3

For CICIDS2018, selection was formulated as constrained class allocation with fixed total size and family-balance pressure.

Primary constraints:

- `sum_c m_c = 300,000` (fixed computational budget),
- `m_B >= 0.33 * 300,000` (minimum benign support),
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

## Shared Processing Protocol

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

## Final State: NF-UNSW-NB15-v3

### Input and Output

| Item | Value |
| --- | --- |
| Input file | `NF-UNSW-NB15-v3/data/NF-UNSW-NB15-v3.csv` |
| Input shape | `2,365,424 × 55` |
| Output file | `NF-UNSW-NB15-v3/NF-UNSW-NB15-v3.csv` |
| Output shape | `191,423 × 51` |

### Non-zero cleaning effects

| Metric | Count |
| --- | ---: |
| `SRC_TO_DST_SECOND_BYTES_inf_to_nan` | 59,068 |
| `DST_TO_SRC_SECOND_BYTES_inf_to_nan` | 122,493 |
| `SRC_TO_DST_SECOND_BYTES_zero_duration_nonfinite_fixed` | 122,493 |
| `DST_TO_SRC_SECOND_BYTES_zero_duration_nonfinite_fixed` | 122,493 |

### Deduplication

| Metric | Count |
| --- | ---: |
| Exact duplicates detected | 14,815 |
| Exact duplicates removed | 14,815 |
| Rows after clean + dedup | 2,350,609 |

### Class-control step

Benign traffic was downsampled to target approximately **33.3%** of final rows while preserving all attack rows after cleaning and deduplication.

| Metric | Count |
| --- | ---: |
| Benign before downsampling | 2,222,930 |
| Attack before downsampling | 127,679 |
| Benign removed | 2,159,186 |
| Benign after downsampling | 63,744 |
| Attack after downsampling | 127,679 |
| Final benign share | 33.3001% |

### Initial vs Final Class Counts

| Class | Initial Count | Final Count | Description |
| --- | ---: | ---: | --- |
| Benign | 2,222,930 | 63,744 | Normal unmalicious flows |
| Fuzzers | 33,816 | 33,816 | An attack in which the attacker sends large amounts of random data which cause a system to crash and also aim to discover security vulnerabilities in a system. |
| Analysis | 1,226 | 1,226 | A group that presents a variety of threats that target web applications through ports, emails and scripts. |
| Backdoor | 4,658 | 4,658 | A technique that aims to bypass security mechanisms by replying to specific constructed client applications. |
| DoS | 5,971 | 5,971 | Denial of Service is an attempt to overload a computer system's resources with the aim of preventing access to or availability of its data. |
| Exploits | 42,744 | 42,744 | Are sequences of commands controlling the behaviour of a host through a known vulnerability. |
| Generic | 19,651 | 19,651 | A method that targets cryptography and causes a collision with each block-cipher. |
| Reconnaissance | 17,074 | 17,074 | A technique for gathering information about a network host and is also known as a probe. |
| Shellcode | 2,381 | 2,381 | A malware that penetrates a code to control a victim's host. |
| Worms | 158 | 158 | Attacks that replicate themselves and spread to other computers. |

### Final attack distribution

| Attack class | Count | Share |
| --- | ---: | ---: |
| Benign | 63,744 | 33.3001% |
| Exploits | 42,744 | 22.3296% |
| Fuzzers | 33,816 | 17.6656% |
| Generic | 19,651 | 10.2657% |
| Reconnaissance | 17,074 | 8.9195% |
| DoS | 5,971 | 3.1193% |
| Backdoor | 4,658 | 2.4334% |
| Shellcode | 2,381 | 1.2438% |
| Analysis | 1,226 | 0.6405% |
| Worms | 158 | 0.0825% |

### Final label distribution

| Label | Count |
| --- | ---: |
| `0` | 63,744 |
| `1` | 127,679 |

### Mapping file

File: `NF-UNSW-NB15-v3/NF-UNSW-NB15-v3_map.json`

```json
{
  "Analysis": "Analysis",
  "Benign": "BENIGN",
  "Backdoor": "Backdoor",
  "DoS": "DoS",
  "Exploits": "Exploits",
  "Fuzzers": "Fuzzers",
  "Generic": "Generic",
  "Reconnaissance": "Reconnaissance",
  "Shellcode": "Shellcode",
  "Worms": "Worms"
}
```

Coverage status: 10 dataset classes mapped, 0 missing keys, 0 extra keys.

---

## Final State: NF-CICIDS2018-v3

### Input and Output

| Item | Value |
| --- | --- |
| Input file | `NF-CICIDS2018-v3/data/NF-CICIDS2018-v3.csv` |
| Output file | `NF-CICIDS2018-v3/NF-CICIDS2018-v3.csv` |
| Output shape | `300,000 × 51` |
| Benign minimum fraction constraint | `0.33` |

### Streaming and deduplication statistics

| Metric | Count |
| --- | ---: |
| Chunks processed per pass | 81 |
| Raw rows scanned per pass | 20,115,529 |
| Rows after cleaning (pre-dedup) | 20,115,529 |
| Local duplicate-hash removals | 628,445 |
| Cross-chunk duplicate-hash removals | 29 |
| Rows after deduplication | 19,487,055 |

### Sampling constraints and fixed minority retention

Requested fixed keeps:

- `DDOS_attack-LOIC-UDP`: 3,450
- `Brute_Force_-Web`: 1,618
- `Brute_Force_-XSS`: 480
- `SQL_Injection`: 440

Assigned after availability checks:

- `DDOS_attack-LOIC-UDP`: 1,725
- `Brute_Force_-Web`: 1,618
- `Brute_Force_-XSS`: 460
- `SQL_Injection`: 440

Shortfall:

- `DDOS_attack-LOIC-UDP`: 1,725
- `Brute_Force_-XSS`: 20

### Initial vs Final Family Counts

| Class | Initial Count | Final Count | Description |
| --- | ---: | ---: | --- |
| Benign | 17,392,754 | 99,000 | Normal unmalicious flows |
| Brute Force | 287,597 | 39,352 | A technique that aims to obtain usernames and password credentials by accessing a list of predefined possibilities |
| Bot | 111,460 | 39,352 | An attack that enables an attacker to remotely control several hijacked computers to perform malicious activities. |
| DoS | 254,296 | 39,351 | An attempt to overload a computer system's resources with the aim of preventing access to or availability of its data. |
| DDoS | 1,322,625 | 41,076 | An attempt similar to DoS but has multiple different distributed sources. |
| Infiltration | 115,805 | 39,351 | An inside attack that sends a malicious file via an email to exploit an application and is followed by a backdoor that scans the network for other vulnerabilities |
| Web Attack | 2,518 | 2,518 | A group that includes SQL injections, command injections and unrestricted file uploads |

### Final class distribution

| Attack class | Count |
| --- | ---: |
| Benign | 99,000 |
| Bot | 39,352 |
| Infilteration | 39,351 |
| DDOS_attack-HOIC | 19,676 |
| FTP-BruteForce | 19,676 |
| SSH-Bruteforce | 19,676 |
| DDoS_attacks-LOIC-HTTP | 19,675 |
| DoS_attacks-GoldenEye | 9,838 |
| DoS_attacks-Hulk | 9,838 |
| DoS_attacks-SlowHTTPTest | 9,838 |
| DoS_attacks-Slowloris | 9,837 |
| DDOS_attack-LOIC-UDP | 1,725 |
| Brute_Force_-Web | 1,618 |
| Brute_Force_-XSS | 460 |
| SQL_Injection | 440 |

### Final label distribution

| Label | Count |
| --- | ---: |
| `0` | 99,000 |
| `1` | 201,000 |

### Final family distribution

| Family | Count |
| --- | ---: |
| BENIGN | 99,000 |
| DDoS | 41,076 |
| Bot | 39,352 |
| Brute Force | 39,352 |
| DoS | 39,351 |
| Infiltration | 39,351 |
| Web Attack | 2,518 |

### Mapping file

File: `NF-CICIDS2018-v3/NF-CICIDS2018-v3_map.json`

```json
{
  "Benign": "BENIGN",
  "DDOS_attack-HOIC": "DDoS",
  "DDoS_attacks-LOIC-HTTP": "DDoS",
  "DDOS_attack-LOIC-UDP": "DDoS",
  "DoS_attacks-Hulk": "DoS",
  "DoS_attacks-GoldenEye": "DoS",
  "DoS_attacks-Slowloris": "DoS",
  "DoS_attacks-SlowHTTPTest": "DoS",
  "FTP-BruteForce": "Brute Force",
  "SSH-Bruteforce": "Brute Force",
  "Bot": "Bot",
  "Infilteration": "Infiltration",
  "Brute_Force_-Web": "Web Attack",
  "Brute_Force_-XSS": "Web Attack",
  "SQL_Injection": "Web Attack"
}
```

Coverage status: 15 dataset classes mapped, 0 missing keys, 0 extra keys.

---

## Data Products

- `NF-UNSW-NB15-v3/NF-UNSW-NB15-v3.csv`
- `NF-UNSW-NB15-v3/NF-UNSW-NB15-v3_summary.json`
- `NF-UNSW-NB15-v3/NF-UNSW-NB15-v3_map.json`
- `NF-CICIDS2018-v3/NF-CICIDS2018-v3.csv`
- `NF-CICIDS2018-v3/NF-CICIDS2018-v3_summary.json`
- `NF-CICIDS2018-v3/NF-CICIDS2018-v3_map.json`

---

# Further Note: Secondary 50:50 Rebalancing Audit

This section documents an additional deterministic rebalancing stage applied after the previously prepared datasets. The resulting files are saved as `dataset.csv` in each dataset directory, with corresponding machine-readable audit records in `dataset_rebalance_report.json`.

Methodological characteristics of this stage:

- Binary target ratio: `Label 0 : Label 1 = 1 : 1`
- Downsample-only procedure (no oversampling)
- Minority preservation on the downsampled side (threshold defined in each audit JSON)
- Deterministic allocation with `random_state = 0`

## Rebalance Artifacts

- `NF-UNSW-NB15-v3/dataset.csv`
- `NF-UNSW-NB15-v3/dataset_rebalance_report.json`
- `NF-CICIDS2018-v3/dataset.csv`
- `NF-CICIDS2018-v3/dataset_rebalance_report.json`

## Aggregate Outcomes

| Dataset | Rows Before | Rows After | Label 0 After | Label 1 After | Max Abs Attack-Share Delta | Minority Preservation Rate |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| NF-UNSW-NB15-v3 | 191,423 | 127,488 | 63,744 | 63,744 | 0.018704 | 100.00% |
| NF-CICIDS2018-v3 | 300,000 | 198,000 | 99,000 | 99,000 | 0.008842 | 100.00% |

Interpretation: the second-stage rebalance achieved exact binary parity in both datasets while preserving all identified minority classes on the downsampled side.

## Distribution Change: NF-UNSW-NB15-v3

Mapping source: `NF-UNSW-NB15-v3/NF-UNSW-NB15-v3_map.json`

| Family | Initial Count | Final Count | Absolute Change | Percent Change |
| --- | ---: | ---: | ---: | ---: |
| BENIGN | 63,744 | 63,744 | 0 | 0.00% |
| Exploits | 42,744 | 20,690 | -22,054 | -51.60% |
| Fuzzers | 33,816 | 16,368 | -17,448 | -51.60% |
| Generic | 19,651 | 9,512 | -10,139 | -51.60% |
| Reconnaissance | 17,074 | 8,264 | -8,810 | -51.60% |
| DoS | 5,971 | 2,890 | -3,081 | -51.60% |
| Backdoor | 4,658 | 2,255 | -2,403 | -51.59% |
| Shellcode | 2,381 | 2,381 | 0 | 0.00% |
| Analysis | 1,226 | 1,226 | 0 | 0.00% |
| Worms | 158 | 158 | 0 | 0.00% |

Interpretation: most non-minority attack families were approximately halved, whereas identified minority families (`Shellcode`, `Analysis`, `Worms`) were preserved exactly.

## Distribution Change: NF-CICIDS2018-v3

Mapping source: `NF-CICIDS2018-v3/NF-CICIDS2018-v3_map.json`

| Family | Initial Count | Final Count | Absolute Change | Percent Change |
| --- | ---: | ---: | ---: | ---: |
| BENIGN | 99,000 | 99,000 | 0 | 0.00% |
| DDoS | 41,076 | 20,676 | -20,400 | -49.66% |
| Bot | 39,352 | 18,952 | -20,400 | -51.84% |
| Brute Force | 39,352 | 18,952 | -20,400 | -51.84% |
| DoS | 39,351 | 18,951 | -20,400 | -51.84% |
| Infiltration | 39,351 | 18,951 | -20,400 | -51.84% |
| Web Attack | 2,518 | 2,518 | 0 | 0.00% |

Interpretation: family-level attack mass was reduced nearly uniformly among non-minority families, while minority web-attack categories were preserved, maintaining rare-event evidence under exact binary balancing.

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
