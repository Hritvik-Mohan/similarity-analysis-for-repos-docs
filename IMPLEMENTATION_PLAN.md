# Similarity Analysis System — Implementation Plan

## Context

This document tracks all work required to evolve the current codebase into a
production-grade plagiarism detection system. Tickets are grouped into three tracks:

- **Track A** — Core features needed to reach the target architecture
- **Track B** — Known gaps/bugs in the current modular `repo_similarity` package
- **Track C** — Future features not implemented in either branch

**Active branch:** `feat/stable-fix`
**Last updated:** 2026-02-20 (reflects local edits to `scripts/run_similarity.py` and `repo_similarity/similarity.py` after commit `63bda82`)

### Use-Case Scope

The system must handle **N student repository submissions for a single coding question**
(typically 30–100 repos per question). This is not a 2-repo diff tool.

Implications:
- **Input model**: a single `submissions/` directory containing N subdirectories (one per student), not two explicit repo paths.
- **Comparison scale**: N=100 → C(100,2) = 4,950 candidate pairs. Exhaustive pairwise fingerprint comparison without candidate filtering is unacceptable at this scale → MinHash + LSH is **critical**, not optional.
- **Output**: a ranked similarity matrix / report across all suspicious submission pairs, not a single-pair diff.
- **Correctness bar**: false negatives (missed plagiarism) are worse than false positives — thresholds should be tuned conservatively.

---

## Current State of `feat/stable-fix` (active branch)

> Last significant commits on this branch:
> - `63bda82` — Refactor normalization and similarity modules; enhance fingerprinting and reporting
> - `798548a` — Merge PR #3 from `feature/modularize-normalization`
> - `58a78e3` — Make normalization pipeline configurable and address review feedback
>
> **Uncommitted local edits (post `63bda82`):**
> - `similarity.py` — added `jaccard_cache` memoization in `compute_pairs`
> - `scripts/run_similarity.py` — extracted magic numbers to named constants; try/except import fallback; ⚠ introduced `MODERATE_SIMILARITY_THRESHOLD` NameError (see TICKET-051)

| File | Role | Status |
|---|---|---|
| `detector/normalization.py` | Monolithic tokenizer + normalizer + fingerprinter + similarity + CLI | Legacy monolith — to be phased out |
| `repo_similarity/constants.py` | Language keywords (JS/Java/Python/C++), 5 file extensions, default config values | Exists — missing `.ts`/`.tsx`/`.jsx` extensions and TS-specific keywords (see TICKET-035, TICKET-036) |
| `repo_similarity/tokenizer.py` | File reading (`read_text`), directory walking (`collect_code_files`), regex tokenizer | Exists, fixed — handles template literals, block/line comments, strings, operators |
| `repo_similarity/normalizer.py` | Identifier normalization to `ID1`, `ID2`, … | Exists — no type distinction (all identifiers use same `IDn` namespace, see TICKET-006) |
| `repo_similarity/fingerprint.py` | k-shingle SHA-1 fingerprinting with two modes: `fingerprints_from_norm` (set only) and `fingerprints_with_positions` (set + per-position list for block extraction) | Exists, enhanced |
| `repo_similarity/similarity.py` | `jaccard`, bidirectional weighted `aggregate`, `compute_pairs` (A→B + B→A best-match passes, deduped + `jaccard_cache` memoization), `_matched_blocks` (contiguous matching shingle-run extraction with token spans) | Exists, enhanced |
| `scripts/run_similarity.py` | Full CLI with named module-level threshold/display constants (`VERY_HIGH_THRESHOLD`, `HIGH_THRESHOLD`, `MODERATE_THRESHOLD`, `MAX_DISPLAYED_BLOCKS`, `FILE_COLUMN_WIDTH`, etc.), try/except import fallback, verdict table, matched block previews, JSON output | Exists, enhanced — **⚠ has `MODERATE_SIMILARITY_THRESHOLD` NameError bug (see TICKET-051)** |
| `README.md` | Project readme | Empty (just title) |

**What this branch can do today:**
- Tokenize JS/Java/Python/C++ source files (regex-based; handles comments, strings, template literals)
- Normalize identifiers to `ID1`, `ID2`, … (correctly detects Type-1 and Type-2 clones)
- Generate k-shingle (k=5) SHA-1 fingerprints with optional per-position tracking
- Compute bidirectional best-match file pairs (A→B + B→A passes, deduped, sorted by Jaccard); Jaccard results are memoized to avoid recomputation across both passes
- Extract matched code blocks — contiguous runs of matching shingle positions merged into token-span ranges
- Assign verdicts per pair: LOW / MODERATE / HIGH / VERY HIGH using named threshold constants (no magic numbers)
- Display matched block previews in console output (up to `MAX_DISPLAYED_BLOCKS=5` per pair)
- Write a JSON results file with per-file metadata and pair detail
- Import `repo_similarity` via try/except with `sys.path` fallback (no package install required)

**What this branch cannot do:**
- Accept a `submissions/` directory with N repos — hardcoded to exactly 2 paths
- Scalable N-way candidate retrieval (no MinHash, no LSH, no Winnowing — O(n²) breaks above ~20 repos)
- Type-3 / Type-4 clone detection (no AST, no structural analysis)
- Dead code preprocessing
- Boilerplate filtering
- Typed identifier normalization (VAR / FUNC / CLASS distinction)
- Full pipeline orchestration with stage boundaries
- N-way similarity report (ranked pair matrix across all submissions)
- Side-by-side HTML diffs

---

## Recommended Project Structure

`repo_similarity/` (the working detection engine) is kept in place. New packages
are added around it — nothing is renamed or moved until replacements are stable.

```
ns-plag-sys/
│
├── config/
│   ├── __init__.py
│   └── constants.py              # TICKET-001 — all thresholds, flags, output dirs
│
├── ingestion/
│   ├── __init__.py
│   └── repository_collector.py  # TICKET-004 — walk submissions/ → {student: {rel: path}}
│
├── repo_similarity/              # KEEP — working detection engine (do not move yet)
│   ├── __init__.py
│   ├── constants.py
│   ├── tokenizer.py
│   ├── normalizer.py
│   ├── fingerprint.py
│   └── similarity.py
│
├── indexing/                     # TICKET-007–010 — scalable N-way candidate retrieval
│   ├── __init__.py
│   ├── minhash.py                # MinHash signatures (128 permutations)
│   ├── winnowing.py              # Winnowing fingerprint selection
│   ├── boilerplate_filter.py     # Entropy + pattern-based boilerplate detection
│   └── candidate_retrieval.py   # LSH bucketing → candidate pairs for N repos
│
├── comparison/                   # TICKET-016 — unified Type-1/2/3/4 scoring
│   ├── __init__.py
│   └── similarity_calculator.py
│
├── reporting/                    # TICKET-017–018 — N-way report generation
│   ├── __init__.py
│   ├── similarity_report.py      # Ranked pair list JSON + text summary
│   └── diff_generator.py         # Per-pair HTML diff
│
├── runner/                       # TICKET-019–020 — orchestration + CLI
│   ├── __init__.py
│   ├── orchestrator.py           # PlagiarismDetectionPipeline (N repos)
│   └── run_all.py                # python -m runner.run_all <submissions_dir>
│
├── tests/                        # TICKET-022–026
│   ├── test_normalization.py
│   ├── test_indexing.py
│   ├── test_structural_analysis.py
│   ├── test_dead_code.py
│   └── test_integration.py
│
├── scripts/
│   └── run_similarity.py         # KEEP — 2-repo CLI for quick dev/debug use
│
├── detector/
│   └── normalization.py          # KEEP for now — legacy monolith, phase out last
│
├── output/                       # gitignored — generated artefacts
│   ├── reports/
│   └── diffs/
│
├── demo_submissions/             # TICKET-021 — N-repo smoke-test fixture
│   ├── student_A/
│   ├── student_B/
│   └── student_C/
│
├── requirements.txt              # TICKET-002
└── README.md                     # TICKET-003
```

**Input model change (current → target):**
```bash
# Current — exactly 2 repo paths:
python scripts/run_similarity.py <repo1_dir> <repo2_dir>

# Target — submissions directory with N subdirs (one per student):
python -m runner.run_all <submissions_dir>/
```

---

## Track A — Parity Tickets

Milestone ordering reflects dependencies. Each ticket must be completed before
tickets that depend on it are started.

---

### MILESTONE 1 — Foundation & Config

---

#### TICKET-001 · Centralized Configuration Package

**Priority:** Critical
**Depends on:** nothing
**Replaces:** scattered constants in `repo_similarity/constants.py` and `detector/normalization.py`

Create `config/` as the single source of truth for all system-wide constants and
feature flags. All other modules must import from here — no magic numbers inline.

**Deliverables:**
- `config/__init__.py` — exports all public constants
- `config/constants.py` — full constant definitions:

```
Winnowing:     WINNOW_WINDOW_SIZE=4, WINNOW_HASH_THRESHOLD=0.5
MinHash:       MINHASH_NUM_PERMUTATIONS=128, MINHASH_THRESHOLD=0.3
Comparison:    TOKEN_MATCH_THRESHOLD=0.70, STRUCTURAL_MATCH_THRESHOLD=0.65,
               CONFIDENCE_MIN=0.5
Dead code:     DETECT_DEAD_CODE=True, FILTER_DEAD_CODE=True,
               DEAD_CODE_MIN_LINES=2, REPORT_DEAD_CODE_STATS=True
Boilerplate:   BOILERPLATE_ENTROPY_THRESHOLD=1.5
Output dirs:   AST_OUTPUT_DIR, REPORTS_OUTPUT_DIR, DIFFS_OUTPUT_DIR
Clone flags:   DETECT_TYPE_1=True, DETECT_TYPE_2=True,
               DETECT_TYPE_3=True, DETECT_TYPE_4=False
Performance:   BATCH_SIZE=10, MAX_COMPARISONS=10000,
               USE_TOKENIZATION_CACHE=True
```

---

#### TICKET-002 · Requirements File

**Priority:** Critical
**Depends on:** nothing

Add `requirements.txt` at repo root so the environment can be reproduced.

```
pytest>=7.0.0
pytest-cov>=3.0.0
black>=22.0.0
flake8>=4.0.0
mypy>=0.950
jsonschema>=4.0.0
```

Note: core detection logic uses only the Python standard library. Node.js + Esprima
are required for JavaScript AST generation (see TICKET-013).

---

#### TICKET-003 · README Rewrite

**Priority:** High
**Depends on:** TICKET-001, TICKET-002

Rewrite `README.md` to document:
- Project purpose and clone type definitions (Type-1 through Type-4)
- Quick-start (`pip install -r requirements.txt`, CLI invocation)
- Python API example (`create_pipeline().run_full_pipeline([...])`)
- Configuration reference (key constants with defaults)
- Performance profile (< 50 files = <5s, 50-500 = 5-30s, 500+ = 30-300s)
- High-level architecture diagram (layer list)
- Links to `ARCHITECTURE.md`, `DEAD_CODE_DETECTION.md`, `DEVELOPMENT.md`

---

### MILESTONE 2 — Ingestion Layer

---

#### TICKET-004 · Repository Ingestion Package

**Priority:** Critical
**Depends on:** TICKET-001

Create `ingestion/` to replace ad-hoc file walking in `detector/normalization.py`
and `repo_similarity/tokenizer.py`.

**Deliverables:**
- `ingestion/__init__.py` — exports `RepositoryCollector`, `FileLoader`
- `ingestion/repository_collector.py`:
  - `RepositoryCollector.collect_repository(path)` → `{rel_path: full_path}`
  - `RepositoryCollector.collect_multiple_repositories(paths)` → `{repo_name: {rel_path: full_path}}`
  - Skips: `node_modules`, `.git`, `__pycache__`, `dist`, `build`, `target`, `.metadata`
  - Supports extensions from `config/constants.py` (`.js`, `.ts`, `.py`, `.java`, `.c`, `.cpp`)
  - `FileLoader.load_file(path)` with UTF-8 + error-ignore fallback
  - In-memory cache keyed on absolute path

---

### MILESTONE 3 — Enhanced Normalization

---

#### TICKET-005 · Upgrade Tokenizer for Full JS/TS Support

**Priority:** High
**Depends on:** TICKET-001

The current `repo_similarity/tokenizer.py` tokenizer is a direct port of the
monolith. Upgrade it to a proper class-based `JavaScriptTokenizer`:

- Handle template literals (backticks) as a single `STR` token
- Capture and drop block comments (`/* */`) and line comments (`//`)
- Expand keyword set to full JS/TS: `interface`, `implements`, `type`, `as`,
  `any`, `never`, `unknown`, `readonly`, `declare`, `namespace`, `module`
- Add `PythonTokenizer` wrapping Python's built-in `tokenize` module with
  regex fallback
- Factory function `get_tokenizer(language)` dispatching by language key

**Deliverables:** Updated `normalization/tokenizer.py` (new location in
the layered package structure — see TICKET-007)

---

#### TICKET-006 · Upgrade Normalizer with Typed Abstraction

**Priority:** High
**Depends on:** TICKET-005

The current normalizer maps all identifiers to `ID1`, `ID2`, …
Upgrade to distinguish semantic token categories:

| Input | Output |
|---|---|
| Variable name | `VAR_0`, `VAR_1`, … |
| Function name | `FUNC_0`, `FUNC_1`, … |
| Class name | `CLASS_0`, `CLASS_1`, … |
| String literal | `STR` |
| Number literal | `NUM` |
| Boolean | `BOOL` |
| null / undefined | `NULL` |
| Keyword | kept as-is |
| Operator / symbol | kept as-is |

**Deliverables:**
- `normalization/__init__.py`
- `normalization/normalizer.py` — `Normalizer`, `CodeNormalizer`, `NormalizedToken`
- `normalization/tokenizer.py` — (from TICKET-005)

---

### MILESTONE 4 — Indexing (Scalable Candidate Retrieval)

---

#### TICKET-007 · Winnowing Algorithm

**Priority:** High
**Depends on:** TICKET-006

Replace the current O(n²) exhaustive pairwise comparison with Winnowing for O(n)
fingerprint generation.

**Algorithm (Schleimer, Wilkerson, Aiken):**
1. Generate k-grams (k=4) from normalized token sequence
2. Hash each k-gram
3. Slide a window of size `w=4` over the hash sequence
4. Select the minimum hash in each window as a fingerprint
5. Jaccard similarity over fingerprint sets = token overlap estimate

**Deliverables:**
- `indexing/winnowing.py`:
  - `Winnower.generate_kgrams(tokens, k)` → list of k-gram strings
  - `Winnower.winnow(tokens)` → set of selected fingerprint hashes
  - `Winnower.jaccard_similarity(set_a, set_b)` → float
  - `Winnower.find_similar_pairs(files_dict, threshold)` → list of (file_a, file_b, score)
- `indexing/__init__.py`

---

#### TICKET-008 · MinHash + LSH

**Priority:** Critical (N-way scale makes this non-optional)
**Depends on:** TICKET-007

Add probabilistic Jaccard estimation using MinHash signatures and Locality
Sensitive Hashing so candidate pairs can be found in near-O(n) time instead of O(n²).
For N=100 submissions this reduces 4,950 full pairwise comparisons to a small
candidate set, making the system practical at classroom scale.

**Algorithm:**
- MinHash: for each of 128 permutations `h_i(x) = (a*x + b) mod p`, take
  the minimum hash over the token shingle set → 128-length signature vector
- LSH: split signature into `b=16` bands of `r=8` rows each; files sharing
  any band bucket are candidate pairs
- Approximate Jaccard from MinHash = `fraction of matching signature positions`

**Deliverables:**
- `indexing/minhash.py`:
  - `MinHash.get_signature(tokens)` → list[int] of length 128
  - `MinHashSignature.jaccard_similarity(sig_a, sig_b)` → float
  - `LSH.add_signature(file_id, signature)`
  - `LSH.find_candidates()` → set of (file_a, file_b) pairs

---

#### TICKET-009 · Boilerplate Filter

**Priority:** Medium
**Depends on:** TICKET-006

Prevent common template/library code from inflating similarity scores.

**Approach:**
- Shannon entropy of normalized token sequence: `H = -Σ p(t) log₂ p(t)`
- Low entropy (< `BOILERPLATE_ENTROPY_THRESHOLD=1.5`) → mark as boilerplate
- Pattern matching: `console.log(`, `module.exports =`, `require(`, `import `,
  `def __init__`, `if __name__ == "__main__":`
- `CommonLibraryFilter`: regex-based import/require detection with score penalties

**Deliverables:**
- `indexing/boilerplate_filter.py`:
  - `BoilerplateDetector.calculate_entropy(tokens)` → float
  - `BoilerplateFilter.analyze_file(file_path, code, tokens)` → bool
  - `BoilerplateFilter.get_boilerplate_files()` → set[str]
  - `BoilerplateFilter.filter_candidates(pairs)` → filtered list
  - `CommonLibraryFilter`

---

#### TICKET-010 · Candidate Retrieval (Hybrid Strategy)

**Priority:** High
**Depends on:** TICKET-007, TICKET-008, TICKET-009

Wire all indexing components into a unified retrieval API that operates across
**all N submissions at once** (not just 2):
1. Builds MinHash signatures for every file across all N repos
2. Uses LSH to bucket files → candidate (file_a, file_b) pairs across all submissions
3. Verifies candidates with Winnowing (more precise Jaccard)
4. Removes boilerplate-flagged files
5. Returns only cross-submission pairs above threshold (same submission pairs are ignored)

**Deliverables:**
- `indexing/candidate_retrieval.py`:
  - `CandidateRetriever.retrieve_candidates_hybrid(all_repo_tokens: dict[str, dict], use_lsh)` → list of (repo_a, file_a, repo_b, file_b, score)
  - Input: `{repo_name: {rel_path: norm_tokens}}` for all N repos simultaneously
  - `InvertedIndex`: token → set of (repo_name, file_id) for fast intersection lookup

---

### MILESTONE 5 — AST Generation Toolchain

---

#### TICKET-011 · Node.js AST Generation (JavaScript / TypeScript)

**Priority:** High
**Depends on:** nothing (independent toolchain)

Set up the Node.js toolchain that generates Esprima `.ast.json` files from JS/TS
source files. These JSON files are consumed by the Python structural analysis layer.

**Deliverables:**
- `ast_tools/package.json` — declares `esprima@^4.0.1` dependency
- `ast_tools/parse.js`:
  - Accepts repo paths as CLI args
  - Recursively walks each repo, processes `.js` and `.jsx` files
  - Generates `output/ast_tree/<repoName>_tree_ast/<filename>.js.ast.json`
  - AST options: `{ loc: true, range: true, comment: true, tolerant: true }`
- `ast_tools/tests/sample.js` + `ast_tools/tests/sample.js.ast.json` — smoke test fixture

**Usage:**
```bash
cd ast_tools && npm install
node parse.js /path/to/repo1 /path/to/repo2
```

---

### MILESTONE 6 — Structural Analysis

---

#### TICKET-012 · AST Processor

**Priority:** High
**Depends on:** TICKET-011

Load and normalize the Esprima JSON ASTs for use in structural comparison.

**Deliverables:**
- `structural_analysis/ast_processor.py`:
  - `ASTNormalizer.normalize_ast(ast)` — strips `loc`, `range`, `comments`;
    replaces all literal values with `'LIT'`; keeps node types and operators
  - `ASTLoader.load_ast_from_file(path)` → dict
  - `ASTLoader.load_asts_from_directory(dir, pattern)` → `{rel_path: ast}`
  - `ASTAnalyzer.get_node_count(ast)` → int
  - `ASTAnalyzer.get_ast_depth(ast)` → int
  - `ASTAnalyzer.get_node_type_distribution(ast)` → `{type: count}`
  - `ASTAnalyzer.get_function_count(ast)` → int
  - `ProcessedAST` dataclass: `path`, `raw`, `normalized`, `node_count`, `depth`
- `structural_analysis/__init__.py`

---

#### TICKET-013 · Tree Hashing

**Priority:** High
**Depends on:** TICKET-012

Produce structural fingerprints from ASTs so tree similarity can be computed
without full sub-tree comparison.

**Algorithm:**
- `TreeHasher.hash_node(node, depth)` — recursively combines node type + sorted child hashes using MD5, depth-limited to `TREE_HASH_DEPTH=3`
- `StructuralSignature` — flat list of all node hashes; supports Jaccard and cosine similarity
- `ASTPairMatcher.compute_structure_similarity(ast_a, ast_b)` → float using `1 - normalized edit distance` on hash sequences

**Deliverables:**
- `structural_analysis/tree_hash.py`:
  - `TreeHasher`, `StructuralSignature`, `ASTPairMatcher`

---

#### TICKET-014 · Function Fingerprinter

**Priority:** Medium
**Depends on:** TICKET-012

Enable function-level comparison rather than just file-level comparison.
This is the precursor to Type-4 (semantic) clone detection.

**Deliverables:**
- `structural_analysis/function_fingerprinter.py`:
  - `FunctionExtractor.extract_functions(ast)` — collects `FunctionDeclaration`,
    `FunctionExpression`, `ArrowFunctionExpression`, `MethodDefinition` nodes
  - `FunctionFingerprint` dataclass: `name`, `hash`, `node_count`,
    `cyclomatic_complexity`, `param_count`
  - `FunctionFingerprinter.fingerprint_file(ast)` → list[FunctionFingerprint]
  - `FunctionFingerprinter.compare_functions(fp_a, fp_b)` → confidence score
    - exact hash match → 0.95
    - similar size/complexity → 0.5–0.7

---

### MILESTONE 7 — Dead Code Preprocessing

---

#### TICKET-015 · Dead Code Detection & Filtering

**Priority:** High
**Depends on:** TICKET-011 (needs ASTs), TICKET-001 (feature flags)

Implement an optional Stage 2.5 that detects and removes unreachable code before
normalization. This reduces false positive similarity scores caused by copied dead
code patterns.

**Detection rules (conservative — never mark code dead unless ALL paths terminate):**
- Statement immediately after `return`, `throw`, `break`, `continue` → dead
- `if/else` where BOTH branches always terminate → code after the block is dead
- `try/catch` — analyzed conservatively (not assumed to terminate)
- Loops — not assumed to terminate (conservative)

**Deliverables:**
- `preprocessing/dead_code_detector.py`:
  - `DeadCodeRange` dataclass: `start_line`, `end_line`, `reason`, `context_line`
  - `DeadCodeDetector.always_terminates(statement)` → bool
  - `DeadCodeDetector.detect_dead_code_in_block(statements)` → list[DeadCodeRange]
  - `DeadCodeFilter.process_file(file_path, source_code, ast)` → (filtered_code, dead_ranges)
  - `DeadCodeFilter.get_summary()` → stats dict
- `preprocessing/__init__.py`

---

### MILESTONE 8 — Comparison Engine

---

#### TICKET-016 · Unified Similarity Calculator

**Priority:** Critical
**Depends on:** TICKET-006, TICKET-007, TICKET-013

Combine all detection signals into a single weighted similarity score per file pair.

**Scoring formula:**
```
overall = type_1 * 0.4 + type_2 * 0.3 + type_3 * 0.2 + type_4 * 0.1
```

| Score | Method |
|---|---|
| type_1 | Direct token sequence comparison (raw tokens) |
| type_2 | Normalized token sequence comparison (VAR_* abstracted) |
| type_3 | AST tree hash Jaccard similarity |
| type_4 | Function-level heuristics (disabled by default) |

**Deliverables:**
- `comparison/similarity_calculator.py`:
  - `SimilarityScore` dataclass: `file_a`, `file_b`, `type_1`, `type_2`, `type_3`, `type_4`, `overall_similarity`, `confidence`
  - `UnifiedSimilarityCalculator.compute_similarity(file_a, file_b, tokens_a, tokens_b, normalized_tokens_a, normalized_tokens_b, ast_a, ast_b)` → SimilarityScore
  - `RepositorySimilarity`: aggregates file-pair scores to repo level, tracks clone type distribution
- `comparison/__init__.py`

---

### MILESTONE 9 — Reporting

---

#### TICKET-017 · N-Way Similarity Report

**Priority:** High
**Depends on:** TICKET-016

Generate a structured JSON report summarizing detected similarities across **all N
submissions**. The report covers every suspicious submission-pair found, not just
a single pair.

**Report structure:**
```json
{
  "metadata": {
    "timestamp", "question_id", "total_submissions", "total_files",
    "candidate_pairs_checked", "suspicious_pairs_found"
  },
  "summary": {
    "total_suspicious_pairs", "very_high_count", "high_count", "medium_count",
    "avg_similarity_of_suspicious"
  },
  "configuration": { "all threshold values" },
  "dead_code_stats": {
    "files_processed", "files_with_dead_code",
    "total_dead_lines_removed", "dead_code_percentage"
  },
  "suspicious_pairs": [
    {
      "submission_a", "submission_b", "overall_similarity", "verdict",
      "clone_type_distribution": { "type_1", "type_2", "type_3", "type_4" },
      "file_pairs": [
        { "file_a", "file_b", "type_1", "type_2", "type_3", "type_4",
          "overall", "confidence" }
      ]
    }
  ]
}
```

Note: `suspicious_pairs` is sorted by `overall_similarity` descending so the
most likely plagiarism appears first.

**Deliverables:**
- `reporting/similarity_report.py`:
  - `SimilarityReport.to_dict()` / `to_json()`
  - `ReportGenerator.add_pair_result(submission_a, submission_b, scores)`
  - `ReportGenerator.set_configuration(config_dict)`
  - `ReportGenerator.set_dead_code_statistics(stats)`
  - `ReportGenerator.save_report(path)`
  - `TextReport.generate_summary()` — human-readable ranked list
- `reporting/__init__.py`

---

#### TICKET-018 · Side-by-Side HTML Diff Generator

**Priority:** Medium
**Depends on:** TICKET-017

Generate per-pair HTML files showing matched code segments highlighted side-by-side.

**Deliverables:**
- `reporting/diff_generator.py`:
  - `DiffMatch` dataclass: `start_a`, `end_a`, `start_b`, `end_b`, `similarity`
  - `DiffGenerator.generate_diffs(code_a, code_b)` → list[DiffMatch] using `difflib.SequenceMatcher`
  - `DiffGenerator.generate_side_by_side_html(code_a, code_b, diffs)` → HTML string

---

### MILESTONE 10 — Pipeline Orchestration

---

#### TICKET-019 · Main Pipeline Orchestrator

**Priority:** Critical
**Depends on:** all previous milestones

Wire all stages into a single `PlagiarismDetectionPipeline` class that accepts
a **submissions directory** (N repos) and produces one report covering all
suspicious pairs found.

**8-stage pipeline:**
```
Stage 1  → load_submissions(submissions_dir)
             discovers all N subdirectories; each subdir = one student submission
Stage 2  → load_source_code()
             tokenizes all files across all N submissions
Stage 2.5→ preprocess_dead_code()     [optional, guarded by DETECT_DEAD_CODE]
Stage 3  → normalize_code()
Stage 4  → filter_boilerplate()
Stage 5  → find_candidates()
             MinHash + LSH across ALL N submissions simultaneously
             → candidate (submission_i, submission_j) pairs
             early return if no candidates found
Stage 6  → load_asts()               [skipped if already loaded]
Stage 7  → compute_similarities()
             full similarity score for each candidate pair only
             early return if no results above threshold
Stage 8  → generate_report()
             N-way JSON report + optional HTML diffs per suspicious pair
```

**Deliverables:**
- `runner/__init__.py`
- `runner/orchestrator.py` — `PlagiarismDetectionPipeline`, `create_pipeline()`

---

#### TICKET-020 · CLI Entry Points

**Priority:** High
**Depends on:** TICKET-019

**Deliverables:**
- `runner/run_all.py` — full pipeline CLI (N submissions):
  ```bash
  # submissions_dir/ contains one subdir per student
  python -m runner.run_all <submissions_dir> [--output report.json] [--threshold 0.5]
  ```
- `runner/run_dead_code.py` — standalone dead code analysis on a single repo:
  ```bash
  python -m runner.run_dead_code <repo_dir>
  ```

---

#### TICKET-021 · Demo Submissions & Sample Test Cases

**Priority:** Medium
**Depends on:** TICKET-011, TICKET-019

Create a `demo_submissions/` directory that mirrors real usage: N student directories,
some of which contain intentional plagiarism and dead code, so the full pipeline can
be smoke-tested end-to-end with a single command.

**Deliverables:**
- `demo_submissions/student_A/` — original JS/Python files with dead code sections
- `demo_submissions/student_B/` — plagiarised variant (renamed variables, reordered blocks)
- `demo_submissions/student_C/` — independent original submission (should score LOW vs A and B)
- Pre-generated ASTs in `output/ast_tree/`
- Expected output fixture `tests/fixtures/demo_report_expected.json` for integration assertion

---

### MILESTONE 11 — Testing

---

#### TICKET-022 · Unit Tests — Normalization

**Priority:** High
**Depends on:** TICKET-005, TICKET-006

**Test cases:**
- Tokenizer: comments stripped, strings → `STR`, numbers → `NUM`, keywords preserved
- Normalizer: identifiers consistently renamed, cross-file isolation (each file resets `VAR_0`)
- Edge cases: empty file, file with only comments, unicode identifiers

**Deliverables:** `tests/test_normalization.py`

---

#### TICKET-023 · Unit Tests — Indexing

**Priority:** High
**Depends on:** TICKET-007, TICKET-008, TICKET-009

**Test cases:**
- Winnowing: known token sequences produce expected fingerprint sets
- MinHash: identical files → similarity ≈ 1.0, disjoint files → similarity ≈ 0.0
- LSH: all pairs sharing a bucket are returned as candidates
- Boilerplate: low-entropy token sequences flagged correctly

**Deliverables:** `tests/test_indexing.py`

---

#### TICKET-024 · Unit Tests — Structural Analysis

**Priority:** Medium
**Depends on:** TICKET-012, TICKET-013, TICKET-014

**Test cases:**
- AST normalization: location info stripped, literals replaced with `LIT`
- Tree hashing: structurally identical ASTs → same hash, different → different hash
- Function extraction: all function types (declaration, expression, arrow, method) found

**Deliverables:** `tests/test_structural_analysis.py`

---

#### TICKET-025 · Unit Tests — Dead Code Detection

**Priority:** Medium
**Depends on:** TICKET-015

**Test cases:**
- Code after bare `return` → dead
- Code after `throw` → dead
- `if/else` where both branches return → code after is dead
- `if` with no `else` → code after is NOT dead (conservative)
- Loop body after `return` → conservative (not flagged)

**Deliverables:** `tests/test_dead_code.py`

---

#### TICKET-026 · Integration Test — End-to-End Pipeline

**Priority:** High
**Depends on:** TICKET-019, TICKET-021

Run the full pipeline against `demo_submissions/` and assert:
- `student_A` ↔ `student_B` overall similarity > 0.7 (intentional plagiarism)
- `student_C` scores LOW vs both A and B (independent submission)
- Dead code lines removed match expected counts
- Output report JSON is valid, conforms to schema, and matches `tests/fixtures/demo_report_expected.json`

**Deliverables:** `tests/test_integration.py`

---

#### TICKET-027 · CI Configuration

**Priority:** Medium
**Depends on:** TICKET-022 – TICKET-026

**Deliverables:**
- `.github/workflows/ci.yml` — runs on push to `master` and PRs:
  - `pip install -r requirements.txt`
  - `flake8 .`
  - `mypy .`
  - `pytest --cov=. --cov-report=xml`

---

### MILESTONE 12 — Documentation

---

#### TICKET-028 · Architecture Document

**Priority:** Medium
**Depends on:** all milestones complete

Write `ARCHITECTURE.md` covering:
- Problem statement and clone type taxonomy
- Layer-by-layer system design with ASCII diagram
- Component descriptions (what each class does and why)
- Performance profile and complexity analysis per component
- Threshold justification table

---

#### TICKET-029 · Dead Code Detection Document

**Priority:** Low
**Depends on:** TICKET-015

Write `DEAD_CODE_DETECTION.md` covering:
- Why dead code filtering matters (prevents inflated similarity on copied templates)
- Detection algorithm with examples (guaranteed-termination analysis)
- Conservative cases (loops, try/catch)
- Configuration options
- Performance overhead

---

#### TICKET-030 · Developer Guide

**Priority:** Low
**Depends on:** all milestones complete

Write `DEVELOPMENT.md` covering:
- Local setup instructions
- Running tests + coverage
- Adding a new language parser
- Adding a new detection strategy
- Threshold tuning guide

---

## Track B — Improvement Tickets (Gaps in the Feature Branch)

These issues exist in `code-similarity-repo-improvements` and should be addressed
before or during the migration.

---

#### TICKET-031 · Fix `repo_similarity/tokenizer.py` — Double File Open

**Status:** ✅ Fixed in `master` (already done)

---

#### TICKET-032 · Fix `repo_similarity/fingerprint.py` — Duplicate Function

**Status:** ✅ Fixed in `master` (already done)

---

#### TICKET-033 · Fix `detector/normalization.py` — Stale TODO Comment

**Status:** ✅ Fixed in `master` (already done)

---

#### TICKET-034 · Fix `runner/orchestrator.py` — Dead Code Block After `return`

**Status:** ✅ Fixed in `code-similarity-repo-improvements` (already done)

---

#### TICKET-035 · `repo_similarity/constants.py` — Missing Extensions

**Priority:** Low

`CODE_EXTENSIONS` only includes `{".java", ".py", ".c", ".cpp", ".js"}`.
Missing: `.ts`, `.tsx`, `.jsx`, `.go`, `.rb`, `.php`, `.cs`.
Add the full set to match what `detector/normalization.py` already has.

---

#### TICKET-036 · `normalization/tokenizer.py` — JS Keyword Set Incomplete

**Priority:** Medium

`JS_KEYWORDS` in the feature branch is missing TypeScript-specific keywords:
`interface`, `implements`, `type`, `as`, `any`, `never`, `unknown`, `readonly`,
`global`, `namespace`, `declare`, `module`.

These are present in `detector/normalization.py` but not in the modular
`repo_similarity/constants.py`. Consolidate into `config/constants.py`.

---

#### TICKET-037 · `structural_analysis` — No Python / Java AST Support

**Priority:** Medium (see also TICKET-040)

`ast_tools/parse.js` only processes `.js` / `.jsx` files via Esprima.
Python files have no AST-based structural analysis path — `PythonTokenizer`
exists but there is no Python AST → JSON pipeline feeding
`structural_analysis/ast_processor.py`.

**Short-term fix:** Wire Python's built-in `ast` module to emit a JSON
representation compatible with `ASTLoader` for `.py` files.

---

#### TICKET-038 · `comparison/similarity_calculator.py` — Type-4 Not Implemented

**Priority:** Low (gated by `DETECT_TYPE_4=False`)

`UnifiedSimilarityCalculator` computes a Type-4 score as `0.0` with a
heuristic fallback. True semantic similarity requires embedding-based comparison.
Ticket covers designing the interface so Type-4 can be plugged in later
(see TICKET-043).

---

#### TICKET-039 · `boilerplate_filter.py` — `COMMON_IDIOMS` Dict Uses `...` Literal

**Priority:** Low

`config/constants.py` on the feature branch has:
```python
'javascript': ['console.log(', 'module.exports =', 'require(', ...],
```
The `...` (Ellipsis) is a Python literal, not a placeholder — it will be
included in the list and never match any token string. Replace with the
full intended list.

---

#### TICKET-051 · `scripts/run_similarity.py` — `MODERATE_SIMILARITY_THRESHOLD` NameError

**Priority:** Critical (runtime crash)

`MODERATE_THRESHOLD = 0.3` is defined at module level but the code references the
undeclared name `MODERATE_SIMILARITY_THRESHOLD` in two places:

- Line 42: `if score >= MODERATE_SIMILARITY_THRESHOLD:` inside `_verdict()`
- Line 82: `flagged = [p for p in pairs if p["jaccard"] >= MODERATE_SIMILARITY_THRESHOLD]`

**Impact:** Any invocation of `print_report` triggers the list comprehension on line 82
unconditionally — `NameError: name 'MODERATE_SIMILARITY_THRESHOLD' is not defined` is
raised every time the CLI is run. The script is currently broken for all inputs.

**Fix:** Either rename the constant at the top from `MODERATE_THRESHOLD` to
`MODERATE_SIMILARITY_THRESHOLD`, or update both usages to `MODERATE_THRESHOLD`.

---

## Track C — Future Feature Tickets

Features not implemented in either branch, drawn from the roadmap docs.

---

#### TICKET-040 · Python AST Structural Analysis

**Priority:** High

Use Python's built-in `ast` module to generate JSON-serialisable AST
representations for `.py` files, feeding the existing `ASTLoader` →
`ASTNormalizer` → `TreeHasher` pipeline.

**Deliverables:**
- `ast_tools/python_parser.py` — serialize Python AST to Esprima-compatible JSON
- Update `config/constants.py` `SUPPORTED_LANGUAGES` to enable Python structural analysis

---

#### TICKET-041 · Java AST Support

**Priority:** Medium

Integrate a Java parser (tree-sitter-java or JavaParser via subprocess)
to generate AST JSON for `.java` files.

---

#### TICKET-042 · C / C++ AST Support

**Priority:** Low

Integrate libclang or tree-sitter-cpp for structural analysis of `.c` / `.cpp` files.

---

#### TICKET-043 · Type-4 Semantic Clone Detection (ML Embeddings)

**Priority:** Low

Implement true semantic similarity using code embeddings:
- Option A: Code2Vec — trains on AST paths, produces function-level vectors
- Option B: UniXcoder / CodeBERT — pre-trained transformer, no training needed
- Similarity: cosine distance between embedding vectors, threshold = 0.60

Gate behind `DETECT_TYPE_4=True` flag. Only runs after Type-1/2/3 pass
to avoid unnecessary inference cost.

---

#### TICKET-044 · Parallel / Concurrent Processing

**Priority:** Medium

For large repos (500+ files), candidate comparison is the bottleneck.
Add optional multiprocessing support:
- `multiprocessing.Pool` for `compute_similarities()` across candidate pairs
- Configurable worker count via `config/constants.py`
- Preserve correctness: results are merged after all workers complete

---

#### TICKET-045 · Incremental / Git-Diff-Based Analysis

**Priority:** Medium

Instead of re-scanning entire repos, only analyse files changed since a
given commit SHA:
- Accept optional `--since <commit>` CLI argument
- Use `git diff --name-only <commit>` to get changed files
- Only re-fingerprint and re-compare changed files; load previous results
  for unchanged files from cache

---

#### TICKET-046 · Fingerprint Cache Layer

**Priority:** Medium
**Depends on:** TICKET-001 (`CACHE_DIR`, `USE_TOKENIZATION_CACHE`)

Persist MinHash signatures and winnowing fingerprints to disk so repeated
runs against the same repo don't re-compute from scratch.

- Cache key: `sha256(file_path + last_modified_time + file_size)`
- Cache format: JSON or pickle under `.cache/`
- Invalidation: cache miss on key mismatch

---

#### TICKET-047 · REST API Server

**Priority:** Low

Expose the pipeline as an HTTP API for CI/CD integration:
- `POST /compare` — body: `{ repo_paths: [...] }` → returns report JSON
- `GET /report/{id}` — retrieve a previously generated report
- Framework: FastAPI (async, auto-docs via OpenAPI)

---

#### TICKET-048 · Web Dashboard

**Priority:** Low
**Depends on:** TICKET-047

Interactive UI for viewing reports:
- Repo-pair similarity heatmap
- Per-file-pair detail view with side-by-side diff (reuse HTML from TICKET-018)
- Filter by clone type, similarity threshold, repository

---

#### TICKET-049 · Distributed Processing (Apache Spark / Celery)

**Priority:** Low

For very large codebases (thousands of files across dozens of repos):
- Celery task queue for async pipeline execution
- Spark-based candidate retrieval for datasets that don't fit in memory

---

#### TICKET-050 · Real-Time Submission Monitoring

**Priority:** Low

Webhook integration for CI/CD: automatically trigger plagiarism analysis
when a new commit or PR is pushed to a monitored repository.

---

## Ticket Summary

| Ticket | Title | Track | Priority | Status |
|---|---|---|---|---|
| TICKET-001 | Centralized Configuration Package | A | Critical | Open — `repo_similarity/constants.py` covers basic values; full `config/` package with all thresholds not yet created |
| TICKET-002 | Requirements File | A | Critical | Open — no `requirements.txt` exists |
| TICKET-003 | README Rewrite | A | High | Open — `README.md` is empty |
| TICKET-004 | Repository Ingestion Package | A | Critical | Open — `collect_code_files` in `tokenizer.py` covers basic walking; no `ingestion/` package |
| TICKET-005 | Upgrade Tokenizer for Full JS/TS Support | A | High | Partial — template literals and comments handled; TypeScript keyword set incomplete (blocked by TICKET-036) |
| TICKET-006 | Upgrade Normalizer with Typed Abstraction | A | High | Open — all identifiers still use same `IDn` namespace; no VAR/FUNC/CLASS distinction |
| TICKET-007 | Winnowing Algorithm | A | High | Open |
| TICKET-008 | MinHash + LSH | A | Critical | Open — mandatory for N=30–100 scale; without it C(N,2) exhaustive comparison is too slow |
| TICKET-009 | Boilerplate Filter | A | Medium | Open |
| TICKET-010 | Candidate Retrieval (Hybrid Strategy) | A | High | Open — must operate across all N submissions simultaneously, not just 2 |
| TICKET-011 | Node.js AST Generation (JS/TS) | A | High | Open |
| TICKET-012 | AST Processor | A | High | Open |
| TICKET-013 | Tree Hashing | A | High | Open |
| TICKET-014 | Function Fingerprinter | A | Medium | Open |
| TICKET-015 | Dead Code Detection & Filtering | A | High | Open |
| TICKET-016 | Unified Similarity Calculator | A | Critical | Open — `similarity.py` has Jaccard; no weighted Type-1/2/3/4 scoring |
| TICKET-017 | N-Way Similarity Report | A | High | Partial — 2-repo JSON output exists in CLI; N-way report with metadata/summary/ranked pairs not yet built |
| TICKET-018 | Side-by-Side HTML Diff Generator | A | Medium | Open |
| TICKET-019 | Main Pipeline Orchestrator | A | Critical | Open — no submissions-dir N-repo pipeline; `scripts/run_similarity.py` handles exactly 2 paths |
| TICKET-020 | CLI Entry Points | A | High | Partial — `scripts/run_similarity.py` (2-repo) works; `runner/run_all.py <submissions_dir>` (N-repo) not built |
| TICKET-021 | Demo Submissions & Sample Test Cases | A | Medium | Partial — `test_repos/student_A` and `student_B` exist (gitignored, 2 repos only); no `demo_submissions/` with ≥3 students and ASTs |
| TICKET-022 | Unit Tests — Normalization | A | High | Open — no `tests/` directory exists |
| TICKET-023 | Unit Tests — Indexing | A | High | Open |
| TICKET-024 | Unit Tests — Structural Analysis | A | Medium | Open |
| TICKET-025 | Unit Tests — Dead Code Detection | A | Medium | Open |
| TICKET-026 | Integration Test — End-to-End Pipeline | A | High | Open |
| TICKET-027 | CI Configuration | A | Medium | Open — no `.github/workflows/` directory |
| TICKET-028 | Architecture Document | A | Medium | Open |
| TICKET-029 | Dead Code Detection Document | A | Low | Open |
| TICKET-030 | Developer Guide | A | Low | Open |
| TICKET-031 | Fix tokenizer double file open | B | — | ✅ Done |
| TICKET-032 | Fix fingerprint duplicate function | B | — | ✅ Done |
| TICKET-033 | Fix stale TODO comment | B | — | ✅ Done |
| TICKET-034 | Fix orchestrator dead code block | B | — | ✅ Done |
| TICKET-035 | constants.py — Missing file extensions | B | Low | Open — `CODE_EXTENSIONS` has only 5 entries; `.ts`, `.tsx`, `.jsx`, `.go`, `.rb`, `.php`, `.cs` absent |
| TICKET-036 | Tokenizer — Incomplete JS keyword set | B | Medium | Open — TS keywords (`interface`, `declare`, `namespace`, `readonly`, `module`, etc.) missing from `repo_similarity/constants.py`; present only in legacy `detector/normalization.py` |
| TICKET-037 | No Python/Java AST structural support | B | Medium | Open |
| TICKET-038 | Type-4 not implemented | B | Low | Open |
| TICKET-039 | boilerplate_filter — Ellipsis literal bug | B | Low | Open |
| TICKET-051 | `run_similarity.py` — `MODERATE_SIMILARITY_THRESHOLD` NameError | B | Critical | Open — script crashes on every run; `MODERATE_THRESHOLD` defined but `MODERATE_SIMILARITY_THRESHOLD` used on lines 42 and 82 |
| TICKET-040 | Python AST Structural Analysis | C | High | Open |
| TICKET-041 | Java AST Support | C | Medium | Open |
| TICKET-042 | C/C++ AST Support | C | Low | Open |
| TICKET-043 | Type-4 Semantic Clone Detection (ML) | C | Low | Open |
| TICKET-044 | Parallel / Concurrent Processing | C | Medium | Open |
| TICKET-045 | Incremental / Git-Diff-Based Analysis | C | Medium | Open |
| TICKET-046 | Fingerprint Cache Layer | C | Medium | Open |
| TICKET-047 | REST API Server | C | Low | Open |
| TICKET-048 | Web Dashboard | C | Low | Open |
| TICKET-049 | Distributed Processing | C | Low | Open |
| TICKET-050 | Real-Time Submission Monitoring | C | Low | Open |

---

## Dependency Graph (Track A Critical Path)

```
TICKET-001 (config)
    └── TICKET-004 (ingestion)
    └── TICKET-005 (tokenizer)
            └── TICKET-006 (normalizer)
                    └── TICKET-007 (winnowing)
                            └── TICKET-010 (candidate retrieval)
                    └── TICKET-009 (boilerplate)
                            └── TICKET-010
                    └── TICKET-016 (similarity calculator)
    └── TICKET-008 (minhash+lsh)
            └── TICKET-010

TICKET-011 (node.js ast gen)
    └── TICKET-012 (ast processor)
            └── TICKET-013 (tree hash)
                    └── TICKET-016
            └── TICKET-014 (function fingerprinter)
                    └── TICKET-016
            └── TICKET-015 (dead code)

TICKET-016 (similarity calculator)
    └── TICKET-017 (report)
            └── TICKET-018 (html diff)
            └── TICKET-019 (orchestrator)
                    └── TICKET-020 (CLI)
```
