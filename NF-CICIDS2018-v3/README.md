# Data Preparation Report: NF-CICIDS2018-v3

## Input and Output

| Item | Value |
| --- | --- |
| Input file | `NF-CICIDS2018-v3/data/NF-CICIDS2018-v3.csv` |
| Input shape | `5,201,806 × 55` |
| Output file | `NF-CICIDS2018-v3/dataset.csv` |
| Output shape | `4,188,602 × 51` |

## Non-zero Cleaning Effects

| Metric | Count |
| --- | ---: |
| Exact duplicates detected | 509,222 |
| Exact duplicates removed | 509,222 |

## Initial vs Final Family Counts

| Family | Initial Count | Final Count | Share | Description |
| --- | ---: | ---: | ---: | --- |
| Benign | 17,392,754 | 2,094,301 | 50.0000% | Normal unmalicious flows |
| DDoS | 1,322,625 | 1,322,625 | 31.5768% | An attempt similar to DoS but has multiple different distributed sources |
| Brute Force | 287,597 | 287,597 | 6.8662% | A technique that aims to obtain usernames and password credentials by accessing a list of predefined possibilities |
| DoS | 254,296 | 254,296 | 6.0711% | An attempt to overload a computer system's resources with the aim of preventing access to or availability of its data |
| Infiltration | 115,805 | 115,805 | 2.7648% | An inside attack that sends a malicious file via an email to exploit an application and is followed by a backdoor that scans the network for other vulnerabilities |
| Bot | 111,460 | 111,460 | 2.6610% | An attack that enables an attacker to remotely control several hijacked computers to perform malicious activities |
| Web Attack | 2,518 | 2,518 | 0.0601% | A group that includes SQL injections, command injections and unrestricted file uploads |

## Final Label Distribution

| Label | Count | Share |
| --- | ---: | ---: |
| `0` | 2,094,301 | 50.0000% |
| `1` | 2,094,301 | 50.0000% |

## Final Family Distribution

| Family | Count | Share |
| --- | ---: | ---: |
| BENIGN | 2,094,301 | 50.0000% |
| DDoS | 1,322,625 | 31.5768% |
| Brute Force | 287,597 | 6.8662% |
| DoS | 254,296 | 6.0711% |
| Infiltration | 115,805 | 2.7648% |
| Bot | 111,460 | 2.6610% |
| Web Attack | 2,518 | 0.0601% |

## Mapping File

File: `NF-CICIDS2018-v3/map.json`

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

- `NF-CICIDS2018-v3/dataset.csv`
- `NF-CICIDS2018-v3/preparation_report.json`
- `NF-CICIDS2018-v3/map.json`
