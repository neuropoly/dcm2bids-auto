# dcm2bids-auto

Automatic extraction of BIDS fields from DICOM metadata — a plugin/extension for [dcm2bids](https://github.com/UNFmontreal/Dcm2Bids).

## Motivation

Converting DICOM images to [BIDS](https://bids.neuroimaging.io/) format currently requires manual configuration: the user must inspect DICOM headers and write heuristic rules (e.g. in a JSON config file for dcm2bids) to map each sequence to its correct BIDS label. This works well for a single, known dataset, but it does not scale to large heterogeneous collections, multi-site studies, or clinical data repositories where sequence naming conventions vary widely across vendors, sites, and operators.

The root cause is a fundamental mismatch in the DICOM standard itself: as shown by Pullens (ISMRM 2026, abstract 567-03-001), only about one-third of sequences in a typical neuro MRI protocol can be unambiguously identified from the small set of *required* DICOM attributes with defined terms (`ImageType`, `ScanningSequence`, `SequenceVariant`, `ScanOptions`, `MRAcquisitionType`). The remainder rely on the user-defined `SequenceName` tag, which is free-text and therefore subjective, site-dependent, and error-prone. The same problem propagates downstream: tools like dcm2niix carry the `SeriesDescription` string into NIfTI filenames and BIDS sidecars, embedding the inconsistency into the converted data.

This project aims to bridge that gap by automatically inferring BIDS fields from DICOM metadata — without requiring a hand-written configuration file.

## Goals

- Build and maintain a large, community-contributed `config.json` that covers as many clinical and research sites, vendors, and scanner models as possible, so that dcm2bids can run with little or no manual configuration on new datasets.
- Parse a set of standard, vendor-independent DICOM tags and derive BIDS-compliant labels (`datatype`, `suffix`, `acq`, `dir`, `part`, etc.) through deterministic rules where possible, and learned heuristics where not.
- Integrate cleanly as a plugin or preprocessing step for [dcm2bids](https://github.com/UNFmontreal/Dcm2Bids), reducing or eliminating the need for manual configuration files.
- Support the full range of sequences common in neuro MRI protocols (T1w, T2w, FLAIR, DWI, fMRI, fieldmaps, …).
- Remain vendor-agnostic and site-agnostic by relying only on standardized DICOM attributes wherever possible.
- Produce human-readable, auditable output so researchers can inspect and override decisions.

## Approach

### The community config.json

The central deliverable of this project is a large, shared `config.json` file in dcm2bids format. Normally, each lab or site creates their own config from scratch, encoding the specific sequence names and parameter ranges used at their scanner. The goal here is different: by aggregating contributions from many sites, vendors (Siemens, GE, Philips, Canon, …), field strengths, and protocol variants, a single config file can be built that works out of the box — or nearly so — for most new datasets.

This is the key distinction from other similar initiatives: rather than each group solving the same problem independently, the config is treated as a shared, versioned community resource, continuously improved as new sites contribute their metadata.

### Rule-based identification

A minimal, ordered set of rules derived from standard DICOM tags with defined terms (following Pullens 2026) provides high-confidence, fully deterministic classification for the subset of sequences that can be uniquely identified this way. Rules are expressed as regular expressions interpretable by PACS systems or Python matching logic, and can be extended by the community.

### Learned heuristics

For the remaining sequences, metadata features (TE, TR, TI, flip angle, b-value, echo train length, scanning sequence flags) are fed to a lightweight classifier. This is inspired by Bartnik et al. (2024, *Neuroinformatics*), who achieved >99% accuracy classifying nine common MRI types using an XGBoost model trained on DICOM header features from public datasets (ADNI, OASIS3, PPMI). The key insight is that relaxation times (TR, TE, TI) are the most discriminative features, which has direct physical justification.

The two layers are applied sequentially: deterministic rules first, learned classifier as fallback.

## Data Sharing and Privacy

### What contributors share

A key ingredient of this project is large-scale metadata sharing across centers. Contributors do **not** share DICOM image files — only a small, carefully stripped subset of DICOM header fields needed for sequence identification. This is what makes broad participation feasible: the data volume is tiny, and the privacy risk, when handled correctly, is minimal.

The fields that will be collected are purely technical acquisition parameters with no diagnostic or patient-level content:

| Field | Tags | Notes |
|---|---|---|
| Sequence type flags | `ScanningSequence` (0018,0020), `SequenceVariant` (0018,0021), `ScanOptions` (0018,0022), `MRAcquisitionType` (0018,0023), `ImageType` (0008,0008) | Fully standardized defined terms |
| Relaxation / timing | TE, TR, TI, flip angle, echo train length, b-value | Numeric; no patient link |
| Sequence name | `SequenceName` (0018,0024), `ProtocolName` (0018,1030), `SeriesDescription` (0008,103E) | Site-defined strings; the main target of harmonization |
| Scanner hardware | `Manufacturer`, `ManufacturerModelName`, `MagneticFieldStrength`, `SoftwareVersions` | Needed for vendor-specific rule coverage |
| Geometry | `MRAcquisitionType`, slice thickness, matrix size, pixel bandwidth | Helps distinguish 2D/3D and reformatted images |

### Fields that must be removed or pseudonymized before sharing

The following fields must be stripped or replaced before any metadata is contributed to this project:

| Field | Action |
|---|---|
| `PatientName`, `PatientID`, `PatientBirthDate`, `PatientAge`, `PatientSex` | **Remove entirely** |
| `AcquisitionDateTime`, `SeriesDate`, `StudyDate`, `ContentDate` | **Replace** with a random offset or dummy date (preserving relative ordering within a session if needed) |
| `InstitutionName`, `InstitutionAddress`, `StationName` | **Replace** with an anonymized site code (e.g. `SITE_042`) — the site identity is useful for tracking vendor/protocol clusters, but the real name should not be shared |
| `OperatorsName`, `ReferringPhysicianName`, `PerformingPhysicianName` | **Remove entirely** |
| `StudyDescription` | **Review before sharing** — at some sites this contains clinical indication text that may be identifiable |
| Private tags (odd group numbers) | **Remove all** — private tags are vendor-specific and may contain patient or device information |

### Ethics considerations

> **Important:** this section reflects current best understanding and is not legal or ethical advice. Contributors are responsible for compliance with the regulations and institutional policies that apply to their site.

Whether stripped acquisition metadata requires ethical approval varies by jurisdiction and institution. In general:

- **EU / GDPR:** pseudonymised data that retains any indirect re-identification potential (e.g. rare disease + known site + approximate date) may still be considered personal data. When in doubt, consult your Data Protection Officer.
- **US / HIPAA:** HIPAA's Safe Harbor method requires removing 18 specific identifiers from a "designated record set." Pure acquisition metadata with no patient fields is arguably outside that scope, but institutional review board (IRB) guidance should be sought for any prospective data collection.
- **General recommendation:** even where approval is not strictly required, contributors are encouraged to document their data-sharing basis (e.g. a note that metadata was stripped per this protocol) and to enter into a lightweight data use agreement (DUA) with the project maintainers.

A template data use agreement and a reference anonymization script will be provided in this repository.

## Related Work

Several tools tackle the DICOM-to-BIDS problem from different angles:

| Tool | Approach | Notes |
|---|---|---|
| [dcm2bids](https://github.com/UNFmontreal/Dcm2Bids) | Heuristic config files (JSON) | Mature, widely used; requires manual config per dataset |
| [HeuDiConv](https://github.com/nipy/heudiconv) | Heuristic Python scripts | Very flexible; requires neuroimaging expertise to configure |
| [BIDScoin](https://github.com/Donders-Institute/bidscoin) | GUI + heuristics | User-friendly interface; relies on SeriesDescription matching |
| [niix2bids](https://github.com/benoitberanger/niix2bids) | Automatic, NIfTI-based | No prior on filenames; Siemens-only; works on dcm2niix output |
| [DeepDicomSort](https://github.com/Svdvoort/DeepDicomSort) | CNN on image content | No metadata needed; does not cover fMRI |
| [scan_classifier](https://gitlab.com/abartnik/scan_classifier) (Bartnik et al. 2024) | XGBoost on DICOM metadata | >99% accuracy; full DICOM-to-BIDS pipeline; open model |

**niix2bids** (Béranger) is the closest in spirit to this project: it operates fully automatically with no assumptions about input file naming, making it well-suited for heterogeneous clinical and multi-cohort data. It works on dcm2niix output (NIfTIs + JSON sidecars) and infers BIDS labels from the sidecar metadata. Its current limitation is that it supports only Siemens scanners.

**Bartnik et al. (2024)** demonstrate that DICOM metadata alone — specifically ten features including TE, TR, TI, flip angle, and scanning sequence flags — is sufficient for highly accurate classification of common MRI types, and show a full end-to-end DICOM-to-BIDS transformation pipeline. Their open model is a candidate component for the learned-heuristic layer of this project.

**Pullens (ISMRM 2024)** provides the theoretical grounding: a systematic analysis of which standard DICOM tags have the discriminative power to identify sequences one-to-one, and a concrete call for standardization of required DICOM attributes. This motivates both the rule-based layer of this project and the broader need for it.

The key differentiator of this project relative to all of the above is the emphasis on **community data sharing at scale**: rather than each group training a model or writing rules on their own data, contributing stripped metadata from many sites and vendors into a shared resource allows the config to generalize far beyond what any single center could achieve.

## Get Involved

This project only works if people contribute — and contributing does not require writing any code. Every site that shares even a small sample of anonymized metadata makes the shared `config.json` more robust for everyone.

**Who should get involved?**

- MRI physicists and radiographers who manage scanner protocols at their site
- Neuroimaging researchers who regularly convert DICOM data to BIDS
- Research software engineers and data managers working with multi-site cohorts
- Clinical researchers dealing with heterogeneous imaging archives
- Anyone who has ever spent an afternoon writing a dcm2bids config file and thought: *there has to be a better way*

**Ways to contribute — from easiest to most involved:**

| What | How | Where |
|---|---|---|
| Tell us about your scanner / protocol setup | Open a [Discussion](../../discussions) | GitHub Discussions → *Show and tell* |
| Report a sequence that is wrongly classified or missing | Open an [Issue](../../issues) | GitHub Issues → *bug* or *missing sequence* label |
| Ask a question about anonymization, ethics, or the format | Open a [Discussion](../../discussions) | GitHub Discussions → *Q&A* |
| Share anonymized metadata from your site | Open a pull request with your metadata file | See *How to Contribute* below |
| Improve the rules or classifier for a specific vendor | Open a pull request with code + test case | See *How to Contribute* below |
| Spread the word | Share with your lab, your BIDS user group, or your scanner vendor rep | Anywhere |

**Not sure where to start?** Open a [Discussion](../../discussions) and describe your scanner setup (vendor, field strength, rough number of sequences in your neuro protocol). That alone is useful information and a good way to get oriented.

**Have an existing dcm2bids config.json** from your site? That is exactly what this project needs. Open an issue or discussion and we will walk you through the anonymization steps together before it gets merged.

The more sites contribute early, the faster the shared config reaches the coverage needed to be genuinely useful. If you are reading this and thinking *this would save us time*, please consider being one of the first contributors — early input has an outsized influence on the direction of the project.

## How to Contribute

> Contribution guidelines are under development. The following is the intended workflow.

1. Extract the relevant DICOM header fields from a representative sample of your sequences (one DICOM per unique sequence in your protocol is sufficient).
2. Apply the anonymization script provided in this repository to strip or pseudonymize all patient- and site-identifying fields.
3. Submit the resulting metadata file via a pull request, along with the vendor, field strength, and a mapping to the correct BIDS labels.

The maintainers will review the submission, integrate it into the shared `config.json`, and add automated tests to prevent regressions.

## Status

🚧 Early development / RFC stage.

Contributions, discussion, and use-case descriptions are welcome via Issues.

## References

- Pullens P. *A plea for objective DICOM based MRI decoding.* ISMRM 2026, abstract 567-03-001.
- Bartnik A, et al. *An automated tool to classify and transform unstructured MRI data into BIDS datasets.* Neuroinformatics 2024. https://doi.org/10.1007/s12021-024-09659-5
- Béranger B. *niix2bids: Automatic BIDS architecture.* https://github.com/benoitberanger/niix2bids
- Gorgolewski KJ, et al. *The brain imaging data structure.* Scientific Data 2016. https://doi.org/10.1038/sdata.2016.44
- Zwiers MP, et al. *BIDScoin: A User-Friendly Application to Convert Source Data to Brain Imaging Data Structure.* Front Neuroinform 2022. https://doi.org/10.3389/fninf.2021.770608

## License

MIT
