# Data Preparation Report for NF-UNSW-NB15-v3 and NF-CICIDS2018-v3

## Scope

This report documents the completed data preparation process for two NetFlow-based intrusion datasets:

- `NF-UNSW-NB15-v3`
- `NF-CICIDS2018-v3`

The objective of the process was to produce reproducible, cleaned, deduplicated, and class-controlled datasets suitable for downstream machine learning experiments.

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

## References

- UQ NIDS dataset portal: `https://staff.itee.uq.edu.au/marius/NIDS_datasets/`
- NF-UNSW-NB15-v3 metadata record: `https://researchdata.edu.au/nf-unsw-nb15-v3/3485172`
- Temporal analysis of NetFlow datasets (NFv3 context): `https://arxiv.org/abs/2503.04404`
- IANA protocol numbers registry: `https://www.iana.org/assignments/protocol-numbers`
- RFC 791 (IPv4 header fields): `https://www.rfc-editor.org/rfc/rfc791`
- RFC 6335 (port namespace and ranges): `https://www.rfc-editor.org/rfc/rfc6335`
