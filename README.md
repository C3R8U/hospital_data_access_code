# hospital_data_access_code
# Hospital Data Access

Multi-authority attribute-based access control for encrypted clinical records, with disposable online/offline encryption tuples, LSSS access policies, blinded edge transformation, and signed metadata for three Hyperledger Fabric channels.

## Dataset

The input is **Diabetes 130-US Hospitals for Years 1999–2008** (UCI Machine Learning Repository, dataset 296). The CSV contains **101,766 hospital encounters**, **71,518 patient identifiers**, and **50 columns**. The `IDS_mapping.csv` file provides descriptions for coded admission and discharge fields.

The access-control workload is built from categorical clinical information and administrative authorization tokens. Patient encounter and patient identifiers are excluded from the encrypted example payloads. Readmission values are used as categorical fields, not as labels for predictive model training.

Dataset source: https://archive.ics.uci.edu/dataset/296/diabetes+130-us+hospitals+for+years+1999-2008

## Model and experiment configuration

The canonical configuration is `config/study.json`.

| Setting | Value |
| --- | --- |
| Random seed | `20261109` |
| Attribute universe | `120` distinct attributes |
| Clinical / administrative attributes | `108 / 12` |
| Attribute authorities | `4` |
| Access-policy row counts | `20, 40, 60, 80, 100` |
| Policy form | AND of OR clauses; unique attribute labels |
| Maximum leaf attributes per OR clause | `20` |
| LSSS matrix coefficients | `-1, 0, +1` |
| Maximum matrix columns / offline vector dimension | `16` |
| Offline cache depth | `8` one-use slots |
| Cache refill threshold | `20%` |
| Configured cache workers | `16` |
| Pairing family | PBC Type A1, symmetric, composite order |
| Composite-order group | `N = p1 * p2 * p3` |
| Subgroup prime lengths | `1024, 1024, 1024` bits |
| Pairing embedding degree | `2` |
| Target security strength | `128` bits |
| Content encryption | AES-128-GCM, 96-bit nonce |
| Key derivation | HKDF-SHA256, 128-bit output |
| Integrity and ownership | SHA-256 and Ed25519 signatures |
| Encrypted clinical payload | `2048` bytes per selected record |
| Warm-up / timed trials per policy size | `20 / 100` |
| Local metadata pipeline workload | `80` requests |
| Fabric channels | `clinical`, `laboratory`, `administrative` |
| Fabric topology parameters | `6` peers, `3` orderers, Raft |
| Block maximum transaction count | `20` |
| Block batch timeout | `100 ms` |
| Preferred block size | `524288` bytes |
| Fabric request rates | `500, 1000, 2000, 3000, 4000` requests/s |
| Fabric trial duration / repetitions | `60 s / 10` |

### Attribute construction and policy generation

The program ranks observed categorical values in 18 selected clinical fields and retains a fixed quota from each field, totaling **108 clinical attribute tokens**. It appends **12 authorization tokens** describing roles, access purposes, channels, and organizations. Every token is assigned to one of four authorities by its position in the attribute catalog.

An access policy contains unique attributes organized as a conjunction of OR clauses. Each clause has at least one attribute held by the selected authorized user. The policy tree is converted into a linear secret-sharing scheme (LSSS) matrix `M`, where row `i` is labeled by `rho(i)`. An authorized row set has reconstruction coefficients `gamma` such that:

```text
sum(gamma[i] * M[i] for i in authorized_rows) = (1, 0, ..., 0)
```

### Encryption and decryption

Each attribute `x` has authority-generated private exponents `alpha_x` and `y_x`, a public key `(e(g,g)^alpha_x, g^y_x)`, and a user-bound key `g^alpha_x * H(GID)^y_x`.

A one-use offline slot stores independent vectors `v = (s, v2, ..., vD)` and `w = (0, w2, ..., wD)`, the corresponding group elements `E[j] = e(g,g)^v[j]` and `W[j] = g^w[j]`, and attribute-specific randomized public-key components `(X[x], Y[x], Z[x])`.

For row `i`, online encryption uses `lambda_i = M[i] * v` and `omega_i = M[i] * w`, with:

```text
C1[i] = X[rho(i)] * product(E[j] ** M[i,j])
C2[i] = Y[rho(i)]
C3[i] = Z[rho(i)] * product(W[j] ** M[i,j], j >= 2)
C0    = R * E[1]
```

The serialized ciphertext includes the policy, row components, encapsulated target-group element, AES-GCM ciphertext, nonce, authenticated context, and a key-confirmation digest. Its owner signature protects the serialized package; SHA-256 supplies the package and policy digests.

A user blinds the attribute keys with an invertible scalar `z`. The edge transformation returns `P`, and the user computes `L`, `Z = L * P**z`, and `R = C0 / Z` before checking the key-confirmation digest and decrypting the AES-GCM payload.

## Requirements and execution

Run commands from the project root. Python dependencies are listed in `requirements.txt`.

```bash
python -m pip install -r requirements.txt
python -m pytest -q tests/
python scripts/run_experiment.py --backend algebra-check
python scripts/statistical_tests.py
python scripts/make_figures.py
python scripts/summarize_results.py
```

The `algebra-check` group backend evaluates the scheme's algebraic identities and access-control operations; the `pbc-a1` backend uses native composite-order bilinear pairings.

To build the native pairing tools, install PBC 0.5.14, GMP, and a C compiler, then run:

```bash
bash scripts/build_native.sh
python scripts/create_pairing_parameters.py --bits 1024
python scripts/run_experiment.py --backend pbc-a1
```

To prepare encrypted metadata records for the three Fabric channels:

```bash
python scripts/build_fabric_workload.py --count 150 --backend pbc-a1
python scripts/ipfs_upload.py --endpoint http://127.0.0.1:5001
cd scripts
npm install
FABRIC_REQUIRE_IPFS=1 node fabric_load.js
```

The Fabric gateway reads its peer address, TLS certificate, client certificate, signing key, and MSP identifier from `FABRIC_PEER_ENDPOINT`, `FABRIC_TLS_CERT`, `FABRIC_CLIENT_CERT`, `FABRIC_CLIENT_KEY`, and `FABRIC_MSP_ID`. The `hospitalmeta` chaincode is deployed separately to the configured channels.

## Project files

| Path | Purpose |
| --- | --- |
| `README.md` | Describes the data, model, configuration, commands, and project files. |
| `MODEL_SPEC.md` | Documents attribute keys, offline shares, ciphertext components, recovery equations, and metric definitions. |
| `EXPERIMENT_NOTES.md` | Records execution modes, input conventions, and workload commands. |
| `requirements.txt` | Lists Python package dependencies. |
| `config/study.json` | Stores the seed and cryptographic, policy, cache, Fabric, and benchmark settings. |
| `healthshare/__init__.py` | Defines the Python package. |
| `healthshare/data.py` | Loads clinical records, derives categorical attributes, and constructs encrypted data payloads. |
| `healthshare/policy.py` | Builds monotone policies, LSSS matrices, and reconstruction witnesses. |
| `healthshare/groups.py` | Implements the algebra-check and native PBC group interfaces. |
| `healthshare/abe.py` | Implements authority keys, offline slots, online encryption, verification, and outsourced recovery. |
| `healthshare/experiments.py` | Runs timed operations, checks ciphertext behavior, and writes measurement data. |
| `native/setup_a1.c` | Creates public A1 pairing parameters and a selected subgroup generator. |
| `native/pbc_bridge.c` | Exposes PBC group operations to the Python interface. |
| `scripts/build_native.sh` | Compiles the native pairing library and parameter generator. |
| `scripts/create_pairing_parameters.py` | Generates three subgroup primes and the public pairing parameter files. |
| `scripts/run_experiment.py` | Launches policy-size experiments using the selected group backend. |
| `scripts/statistical_tests.py` | Calculates paired hypothesis tests and Holm-adjusted p-values. |
| `scripts/make_figures.py` | Creates encryption, access, integrity, and ciphertext-size plots. |
| `scripts/summarize_results.py` | Displays aggregate execution measurements and dataset counts. |
| `scripts/build_fabric_workload.py` | Creates encrypted records and transaction templates for the three channels. |
| `scripts/ipfs_upload.py` | Uploads encrypted records and replaces file references with IPFS content addresses. |
| `scripts/fabric_load.js` | Submits metadata transactions through Fabric Gateway and records commit outcomes. |
| `scripts/package.json` | Declares Node.js dependencies for the Fabric load client. |
| `scripts/write_manifest.py` | Records package versions and SHA-256 hashes of the input configuration and data files. |
| `chaincode/metadata/main.go` | Implements the Fabric metadata storage contract. |
| `chaincode/metadata/go.mod` | Defines the Go module and dependencies for the chaincode. |
| `tests/test_engine.py` | Tests authorized recovery, policy enforcement, signature checks, and rejected tampering. |
| `data/raw/diabetic_data.csv` | Contains the source hospital encounter records. |
| `data/raw/IDS_mapping.csv` | Maps coded clinical administrative fields to their descriptions. |
| `data/derived/attribute_catalog.csv` | Lists the 120 authorization tokens, source fields, and occurrence counts. |
| `data/derived/code_descriptions.json` | Stores the parsed descriptions of coded hospital fields. |
| `data/derived/readmission_by_age.csv` | Summarizes readmission categories by age group. |
| `data/derived/policy_instances.jsonl` | Stores generated policy matrices and associated authorized attribute sets. |
| `data/derived/fabric_transactions.jsonl` | Stores signed metadata templates for encrypted channel submissions. |
| `data/derived/fabric_ciphertexts/*.bin` | Stores the encrypted objects referenced by Fabric transaction templates. |
| `results/run.log` | Records the experiment execution sequence and policy-size progress. |
| `results/training.log` | Records data preparation and attribute-policy generation activity. |
| `results/performance_raw.csv` | Stores individual operation timings and ciphertext sizes. |
| `results/performance_summary.csv` | Stores means, sample standard deviations, and 95% confidence intervals. |
| `results/paired_tests.csv` | Stores paired-test results and adjusted p-values. |
| `results/verification_batches.csv` | Stores the timings of 1-, 10-, and 50-object verification batches. |
| `results/dataset_profile.json` | Stores the source data counts, readmission classes, and missing-value counts. |
| `results/integrity_checks.json` | Stores ciphertext tampering, transformation, and channel-isolation checks. |
| `results/metrics.json` | Stores run configuration identifiers and workload-level measurements. |
| `results/hardware.json` | Records the operating system, processor, and Python runtime details. |
| `results/manifest.json` | Stores data and configuration hashes and software package versions. |
| `results/local_metadata.sqlite` | Stores local metadata records used by the encrypted storage pipeline. |
| `results/encrypted_objects/*.bin` | Stores encrypted objects produced by the local storage pipeline. |
| `results/figures/encryption.png` | Plots online and direct encryption timings by policy size. |
| `results/figures/access.png` | Plots direct and edge-assisted recovery timings by policy size. |
| `results/figures/integrity.png` | Plots ciphertext integrity-check timings by policy size. |
| `results/figures/ciphertext_size.png` | Plots serialized ciphertext sizes by policy size. |

## Data citation

Strack, B., DeShazo, J. P., Gennings, C., Olmo, J. L., Ventura, S., Cios, K. J., and Clore, J. N. (2014). *Impact of HbA1c Measurement on Hospital Readmission Rates: Analysis of 70,000 Clinical Database Patient Records.* BioMed Research International, 2014, 781670. https://doi.org/10.1155/2014/781670
