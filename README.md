# NF-UNSW-NB15-v3 and NF-CICIDS2018-v3 Preparation Journal

This document records the full preprocessing, cleaning, balancing, and mapping workflow used to produce the current final CSV artifacts and map files.

## 1) Processing flow used in both pipelines

1. Load data with strict schema validation.
- Expected columns: 55 NetFlow v3 fields (`FLOW_START_MILLISECONDS` ... `Label`, `Attack`).
- Any missing or unexpected columns fail the run.

2. Normalize text fields before any numeric processing.
- IP columns are string-normalized by trimming whitespace and replacing empty/null-like values with canonical placeholders.
- `Attack` text is canonicalized (trim/lowercase first, then normalized to canonical class names).

3. Coerce every numeric field to numeric and standardize non-finite values.
- `pd.to_numeric(..., errors="coerce")` behavior is applied to numeric features.
- `+inf/-inf` are converted to `NaN`.
- Per-column counters are tracked for:
  - non-numeric to `NaN`
  - `inf` to `NaN`

4. Rebuild binary label semantics from attack label.
- `Label = 0` iff `Attack == Benign`
- `Label = 1` otherwise.
- Any source label mismatch is counted and corrected.

5. Enforce temporal integrity.
- If `FLOW_END_MILLISECONDS < FLOW_START_MILLISECONDS`, swap them.
- Recompute/impute `FLOW_DURATION_MILLISECONDS` from timestamps when missing or inconsistent.

6. Enforce min/max pair ordering constraints.
- Swap if violated:
  - `MIN_TTL` and `MAX_TTL`
  - `SHORTEST_FLOW_PKT` and `LONGEST_FLOW_PKT`
  - `MIN_IP_PKT_LEN` and `MAX_IP_PKT_LEN`

7. Repair zero-duration throughput edge cases.
- For zero-duration flows, non-finite directional throughput/rate fields are forced to `0.0`.

8. Apply validity bounds and clipping rules.
- Non-negative clipping on count/size/duration-like numeric fields.
- Range clipping on bounded protocol fields, including:
  - ports: `[0, 65535]`
  - protocol/TTL/TCP flags: `[0, 255]` where applicable
  - DNS ID/type: `[0, 65535]`
  - DNS TTL answer: `[0, 4294967295]`
  - binary label: `[0, 1]`

9. Fill remaining `NaN` values deterministically.
- Float-designated columns use float median fill.
- Integer-designated columns use rounded integer median fill.

10. Remove exact duplicate rows.
- Row-level exact duplicates are removed.
- Duplicate accounting is recorded.

11. Remove high-leakage identifier/time columns from final export.
- Dropped columns:
  - `IPV4_SRC_ADDR`
  - `IPV4_DST_ADDR`
  - `FLOW_START_MILLISECONDS`
  - `FLOW_END_MILLISECONDS`

12. Enforce deterministic output.
- Fixed random seed (`random_state=0`) for all randomized downsampling/shuffling operations.

13. Export final CSV and JSON summary.
- Persist final row/column shape, class distributions, label distributions, cleaning counters, and mapping metadata.

---

## 2) NF-UNSW-NB15-v3 (final state)

### 2.1 Input and structural checks
- Input file: `NF-UNSW-NB15-v3/data/NF-UNSW-NB15-v3.csv`
- Input shape: `2,365,424 x 55`

### 2.2 Cleaning actions with non-zero impact
- `SRC_TO_DST_SECOND_BYTES_inf_to_nan`: `59,068`
- `DST_TO_SRC_SECOND_BYTES_inf_to_nan`: `122,493`
- `SRC_TO_DST_SECOND_BYTES_zero_duration_nonfinite_fixed`: `122,493`
- `DST_TO_SRC_SECOND_BYTES_zero_duration_nonfinite_fixed`: `122,493`

### 2.3 Deduplication
- Exact duplicate rows detected: `14,815`
- Exact duplicate rows removed: `14,815`
- Shape after clean+dedup: `2,350,609 x 55`

### 2.4 Label balancing
- Objective used: benign/attack `50:50` via benign downsampling only.
- Before balancing:
  - benign: `2,222,930`
  - attack: `127,679`
- Benign rows removed for balance: `2,095,251`
- After balancing:
  - benign: `127,679`
  - attack: `127,679`
  - rows: `255,358`

### 2.5 Column dropping
- Dropped:
  - `IPV4_SRC_ADDR`
  - `IPV4_DST_ADDR`
  - `FLOW_START_MILLISECONDS`
  - `FLOW_END_MILLISECONDS`
- Final shape: `255,358 x 51`

### 2.6 Final class distribution (`Attack`)
- `Benign`: `127,679`
- `Exploits`: `42,744`
- `Fuzzers`: `33,816`
- `Generic`: `19,651`
- `Reconnaissance`: `17,074`
- `DoS`: `5,971`
- `Backdoor`: `4,658`
- `Shellcode`: `2,381`
- `Analysis`: `1,226`
- `Worms`: `158`

### 2.7 Final binary label distribution (`Label`)
- `0`: `127,679`
- `1`: `127,679`

### 2.8 Final mapping state
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

Map coverage check:
- dataset attack labels: `10`
- map keys: `10`
- missing keys: none
- extra keys: none

---

## 3) NF-CICIDS2018-v3 (final state)

### 3.1 Input and streaming strategy
- Input file: `NF-CICIDS2018-v3/data/NF-CICIDS2018-v3.csv`
- Data is processed in streaming chunks (memory-safe), not as one in-memory frame.
- Raw rows scanned per pass: `20,115,529`
- Chunks per pass: `81`

### 3.2 Cleaning and dedup results (per full pass)
- Rows after clean (before dedup): `20,115,529`
- Local duplicate hash removals: `628,445`
- Cross-chunk duplicate hash removals: `29`
- Rows after dedup: `19,487,055`

### 3.3 Sampling objective and constraints
- Target final rows: `300,000`
- Benign minimum fraction: `0.33`
- Effective benign target: `99,000`
- Attack budget: `201,000`
- Fixed keep requests:
  - `DDOS_attack-LOIC-UDP`: `3,450`
  - `Brute_Force_-Web`: `1,618`
  - `Brute_Force_-XSS`: `480`
  - `SQL_Injection`: `440`
- Fixed keep assigned (capped by availability after clean+dedup):
  - `DDOS_attack-LOIC-UDP`: `1,725`
  - `Brute_Force_-Web`: `1,618`
  - `Brute_Force_-XSS`: `460`
  - `SQL_Injection`: `440`
- Fixed keep shortfall:
  - `DDOS_attack-LOIC-UDP`: `1,725`
  - `Brute_Force_-Web`: `0`
  - `Brute_Force_-XSS`: `20`
  - `SQL_Injection`: `0`

### 3.4 Target allocation mechanism
1. Build class counts and family counts from cleaned+deduped data.
2. Reserve fixed-kept class counts first.
3. Allocate remaining attack budget across families as evenly as possible with no oversampling.
4. Allocate each family target across classes as evenly as possible with no oversampling.
5. Run deterministic per-class reservoir sampling.
6. Write only selected rows to final CSV.

### 3.5 Final output shape
- Final shape: `300,000 x 51`

### 3.6 Final class distribution (`Attack`)
- `Benign`: `99,000`
- `Bot`: `39,352`
- `Infilteration`: `39,351`
- `DDOS_attack-HOIC`: `19,676`
- `FTP-BruteForce`: `19,676`
- `SSH-Bruteforce`: `19,676`
- `DDoS_attacks-LOIC-HTTP`: `19,675`
- `DoS_attacks-GoldenEye`: `9,838`
- `DoS_attacks-Hulk`: `9,838`
- `DoS_attacks-SlowHTTPTest`: `9,838`
- `DoS_attacks-Slowloris`: `9,837`
- `DDOS_attack-LOIC-UDP`: `1,725`
- `Brute_Force_-Web`: `1,618`
- `Brute_Force_-XSS`: `460`
- `SQL_Injection`: `440`

### 3.7 Final binary label distribution (`Label`)
- `0`: `99,000`
- `1`: `201,000`

### 3.8 Final family distribution (using current map)
- `BENIGN`: `99,000`
- `DDoS`: `41,076`
- `Bot`: `39,352`
- `Brute Force`: `39,352`
- `DoS`: `39,351`
- `Infiltration`: `39,351`
- `Web Attack`: `2,518`

### 3.9 Final mapping state
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

Map coverage check:
- dataset attack labels: `15`
- map keys: `15`
- missing keys: none
- extra keys: none

---

## 4) Authoritative references used during cleaning-rule design

- UQ NF dataset portal: `https://staff.itee.uq.edu.au/marius/NIDS_datasets/`
- NF-UNSW-NB15-v3 record: `https://researchdata.edu.au/nf-unsw-nb15-v3/3485172`
- Temporal analysis paper (NFv3 context): `https://arxiv.org/abs/2503.04404`
- IANA protocol numbers: `https://www.iana.org/assignments/protocol-numbers`
- RFC 791 (IPv4 header semantics): `https://www.rfc-editor.org/rfc/rfc791`
- RFC 6335 (port number ranges): `https://www.rfc-editor.org/rfc/rfc6335`

