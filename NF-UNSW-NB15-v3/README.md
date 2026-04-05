# Data Preparation Report: NF-UNSW-NB15-v3

## Input and Output

| Item | Value |
| --- | --- |
| Input file | `NF-UNSW-NB15-v3/data/NF-UNSW-NB15-v3.csv` |
| Input shape | `2,365,424 × 55` |
| Output file | `NF-UNSW-NB15-v3/dataset.csv` |
| Output shape | `255,358 × 51` |

## Non-zero cleaning effects

| Metric | Count |
| --- | ---: |
| `SRC_TO_DST_SECOND_BYTES_inf_to_nan` | 59,068 |
| `DST_TO_SRC_SECOND_BYTES_inf_to_nan` | 122,493 |
| `SRC_TO_DST_SECOND_BYTES_zero_duration_nonfinite_fixed` | 122,493 |
| `DST_TO_SRC_SECOND_BYTES_zero_duration_nonfinite_fixed` | 122,493 |

## Deduplication

| Metric | Count |
| --- | ---: |
| Exact duplicates detected | 14,815 |
| Exact duplicates removed | 14,815 |

## Initial vs Final Class Counts

| Class | Initial Count | Final Count | Share | Description |
| --- | ---: | ---: | ---: | --- |
| Benign | 2,222,930 | 127,679 | 50.0000% | Normal unmalicious flows |
| Exploits | 42,744 | 42,744 | 16.7389% | Sequences of commands controlling the behaviour of a host through a known vulnerability |
| Fuzzers | 33,816 | 33,816 | 13.2426% | An attack in which the attacker sends large amounts of random data which cause a system to crash and also aim to discover security vulnerabilities in a system |
| Generic | 19,651 | 19,651 | 7.6955% | A method that targets cryptography and causes a collision with each block-cipher |
| Reconnaissance | 17,074 | 17,074 | 6.6863% | A technique for gathering information about a network host and is also known as a probe |
| DoS | 5,971 | 5,971 | 2.3383% | Denial of Service is an attempt to overload a computer system's resources with the aim of preventing access to or availability of its data |
| Backdoor | 4,658 | 4,658 | 1.8241% | A technique that aims to bypass security mechanisms by replying to specific constructed client applications |
| Shellcode | 2,381 | 2,381 | 0.9324% | A malware that penetrates a code to control a victim's host |
| Analysis | 1,226 | 1,226 | 0.4801% | A group that presents a variety of threats that target web applications through ports, emails and scripts |
| Worms | 158 | 158 | 0.0619% | Attacks that replicate themselves and spread to other computers |

## Final Family Distribution

| Family | Count | Share |
| --- | ---: | ---: |
| BENIGN | 127,679 | 50.0000% |
| Exploits | 42,744 | 16.7389% |
| Fuzzers | 33,816 | 13.2426% |
| Generic | 19,651 | 7.6955% |
| Reconnaissance | 17,074 | 6.6863% |
| DoS | 5,971 | 2.3383% |
| Backdoor | 4,658 | 1.8241% |
| Shellcode | 2,381 | 0.9324% |
| Analysis | 1,226 | 0.4801% |
| Worms | 158 | 0.0619% |

## Final Label Distribution

| Label | Count | Share |
| --- | ---: | ---: |
| `0` | 127,679 | 50.0000% |
| `1` | 127,679 | 50.0000% |

## Mapping File

File: `NF-UNSW-NB15-v3/map.json`

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

## Data Products

- `NF-UNSW-NB15-v3/dataset.csv`
- `NF-UNSW-NB15-v3/preparation_report.json`
- `NF-UNSW-NB15-v3/map.json`
