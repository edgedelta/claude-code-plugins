# Changelog - edgedelta-reference

All notable changes to this skill will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] - 2025-10-19

### Added
- Versioning in SKILL.md frontmatter
- SECURITY.md documentation
- Additional processor documentation
- Disparity tracking between docs and source code

### Features (Current)
- **23 sequence-compatible processors** covered in MASTER_INDEX
- **4 detailed processor references**:
  - `generic_mask.md` (~1000 lines) - PII masking, regex redaction
  - `extract_metric.md` - Log-to-metric conversion, OTTL conditions
  - `ottl_transform.md` - OTTL transformations, attribute manipulation
  - `json_unroll.md` - Array unrolling, batch processing

### Processor Categories
- Core Transformation & Filtering (3 processors)
- Masking & Privacy (2 processors)
- Sampling & Rate Limiting (4 processors)
- Metrics Extraction (4 processors)
- Data Manipulation (3 processors)
- Deduplication (1 processor)
- Enrichment (1 processor)
- Special (1 processor - deotel)

### Documentation
- MASTER_INDEX.md - Quick reference with copy-paste snippets
- documentation_disparities.md - Tracks differences between docs and source
- 6 workflow patterns documented

## [1.1.0] - 2025-10-10

### Added
- Detailed `json_unroll.md` reference
- Common pitfalls section for each processor
- Production examples from tested templates

### Changed
- Expanded `generic_mask.md` with more examples
- Improved OTTL transformation documentation
- Updated validation rules

## [1.0.0] - Initial

### Added
- First production release
- MASTER_INDEX with all processors
- Basic processor documentation
- Quick copy snippets
