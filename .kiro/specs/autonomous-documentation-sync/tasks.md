# Implementation Plan: Scribe (Autonomous Documentation Sync)

## Overview

Implement the Scribe documentation sync agent as a new `openhands/scribe/` module following the same patterns as `openhands/resolver/`. Tasks are ordered to build core data models first, then config/parsing, then drift detection, then update generation/verification, then PR creation, and finally the GitHub Actions workflow.

## Tasks

- [ ] 1. Set up module structure and data models
  - [ ] 1.1 Create `openhands/scribe/__init__.py` and `openhands/scribe/models.py`
    - Define `ChangeType` enum, `DocType` enum, `ChangeEntry`, `ChangeSummary`, `DocFile`, `DriftEntry`, `DriftReport`, `DocUpdate`, `VerificationResult`, `PRResult`, and `ScribeOutput` Pydantic models
    - Implement `to_json()` / `from_json()` (or use Pydantic's `model_dump_json()` / `model_validate_json()`) on `ChangeSummary`, `DriftReport`, and `ScribeOutput`
    - _Requirements: 2.1, 2.3, 2.4, 3.3, 3.4, 3.5, 8.2, 8.4_

  - [ ] 1.2 Write property test for ChangeSummary round-trip serialization
    - **Property 1: Change_Summary round-trip serialization**
    - **Validates: Requirements 2.3, 2.4**

  - [ ] 1.3 Write property test for DriftReport round-trip serialization
    - **Property 2: Drift_Report round-trip serialization**
    - **Validates: Requirements 3.4, 3.5**

  - [ ] 1.4 Write property test for ScribeOutput round-trip serialization
    - **Property 3: Scribe_Output round-trip serialization**
    - **Validates: Requirements 8.4**

- [ ] 2. Implement configuration loading
  - [ ] 2.1 Create `openhands/scribe/config.py`
    - Implement `ScribeConfig` dataclass with fields: `enabled`, `doc_paths`, `ignore_paths`, `max_iterations`, `doc_mappings`
    - Implement `ScribeConfig.load(repo_dir)` that reads `.scribe.yml`, validates fields, falls back to defaults on missing/invalid file
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.6, 1.4, 1.5_

  - [ ] 2.2 Write property test for config fields parsed correctly with defaults
    - **Property 15: Config fields parsed correctly with defaults**
    - **Validates: Requirements 7.1, 7.2, 7.3, 7.4, 7.6**

  - [ ] 2.3 Write unit tests for config loading edge cases
    - Test missing file (defaults), invalid YAML (fallback), partial config, `enabled=false` behavior
    - _Requirements: 1.4, 1.5, 7.6, 7.7_

- [ ] 3. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement PR change analysis
  - [ ] 4.1 Create `openhands/scribe/change_analyzer.py`
    - Implement `PRChangeAnalyzer` class that parses unified diffs into `ChangeSummary` objects
    - Use regex to detect function/class definitions, route decorators, and dependency file changes in diff hunks
    - Categorize each change into the appropriate `ChangeType`
    - Set `has_code_changes=False` when diff contains only non-code files
    - _Requirements: 2.1, 2.2, 2.5_

  - [ ] 4.2 Write property test for diff parsing completeness
    - **Property 17: Diff parsing produces complete Change_Summary**
    - **Validates: Requirements 2.1**

  - [ ] 4.3 Write unit tests for change analyzer
    - Test with sample diffs: Python function addition, endpoint decorator change, dependency update, non-code-only diff
    - _Requirements: 2.1, 2.2, 2.5_

- [ ] 5. Implement documentation inventory and drift detection
  - [ ] 5.1 Create `openhands/scribe/doc_inventory.py`
    - Implement `DocInventory` class that scans the repo file tree using glob patterns from `ScribeConfig`
    - Classify each matched file into a `DocType` based on file name and content
    - _Requirements: 3.1, 3.7_

  - [ ] 5.2 Write property test for doc inventory pattern matching
    - **Property 4: Doc_Inventory respects configured patterns**
    - **Validates: Requirements 3.1, 3.7**

  - [ ] 5.3 Create `openhands/scribe/drift_detector.py`
    - Implement `DocDriftDetector` class that compares `ChangeSummary` against `DocInventory` results
    - Read each doc file's content and check for references to changed element names
    - Use `doc_mappings` from config to directly map source paths to doc files
    - For OpenAPI specs, check if changed endpoints appear in the spec's paths
    - Produce a `DriftReport` with entries for each stale doc file
    - _Requirements: 3.2, 3.3, 3.6, 7.5_

  - [ ] 5.4 Write property test for drift detection correctness
    - **Property 5: Drift detection identifies docs referencing changed elements**
    - **Validates: Requirements 3.2**

  - [ ] 5.5 Write property test for doc_mappings prioritization
    - **Property 6: Doc_mappings prioritize drift detection**
    - **Validates: Requirements 7.5**

- [ ] 6. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Implement update verification
  - [ ] 7.1 Create `openhands/scribe/verifier.py`
    - Implement `UpdateVerifier` with methods: `verify_openapi()`, `verify_mermaid()`, `verify_markdown()`, `verify_all()`
    - OpenAPI: validate against OpenAPI 3.x schema using `openapi-spec-validator` or `jsonschema`
    - Mermaid: extract fenced mermaid blocks, validate diagram type declarations and balanced delimiters
    - Markdown: parse and check that all internal heading links (`[text](#slug)`) resolve to actual headings
    - `verify_all()` returns only the updates that pass verification
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [ ] 7.2 Write property test for OpenAPI verification
    - **Property 9: OpenAPI verification rejects invalid specs**
    - **Validates: Requirements 5.1**

  - [ ] 7.3 Write property test for Mermaid verification
    - **Property 10: Mermaid verification validates syntax**
    - **Validates: Requirements 5.2**

  - [ ] 7.4 Write property test for Markdown internal link verification
    - **Property 11: Markdown internal link verification**
    - **Validates: Requirements 5.3**

  - [ ] 7.5 Write property test for verified updates subset
    - **Property 12: Verified updates are subset of input updates**
    - **Validates: Requirements 5.5**

- [ ] 8. Implement documentation updater
  - [ ] 8.1 Create `openhands/scribe/doc_updater.py`
    - Implement `DocUpdater` class that uses the OpenHands `AgentController` to generate documentation updates
    - Construct prompts containing the Drift_Report, current doc content, relevant source files, and Change_Summary
    - Include doc-type-specific instructions (OpenAPI schema rules, Mermaid syntax, Markdown structure preservation)
    - Constrain the agent to modify only files listed in the Drift_Report
    - Handle agent timeout by skipping the file and continuing
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7_

  - [ ] 8.2 Write property test for update context completeness
    - **Property 7: Update context includes all required elements**
    - **Validates: Requirements 4.2**

  - [ ] 8.3 Write property test for updates constrained to Drift_Report files
    - **Property 8: Updates constrained to Drift_Report files**
    - **Validates: Requirements 4.7**

  - [ ] 8.4 Write property test for README structure preservation
    - **Property 16: README structure preservation**
    - **Validates: Requirements 4.5**

- [ ] 9. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Implement PR creation
  - [ ] 10.1 Create `openhands/scribe/pr_creator.py`
    - Implement `PRCreator` class that reuses `send_pull_request` utilities from `openhands/resolver/`
    - Create branch `docs/scribe-sync-{pr_number}`, commit verified updates, open PR
    - Generate PR body listing each updated file with description and reference to original PR
    - Apply `documentation` label if it exists in the repository
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [ ] 10.2 Write property test for PR naming conventions
    - **Property 13: PR naming conventions**
    - **Validates: Requirements 6.1, 6.2, 6.3**

  - [ ] 10.3 Write property test for PR body content
    - **Property 14: PR body contains all updated files and original PR reference**
    - **Validates: Requirements 6.4**

- [ ] 11. Wire components together in the Scribe Agent entry point
  - [ ] 11.1 Create `openhands/scribe/resolve_docs.py`
    - Implement `ScribeAgent` class that orchestrates the full pipeline: config loading → diff fetching → change analysis → inventory scan → drift detection → update generation → verification → PR creation
    - Implement CLI argument parsing analogous to `openhands/resolver/resolve_issue.py`
    - Write `ScribeOutput` to the output directory as `scribe_output.json`
    - Handle all early-exit conditions (disabled, no code changes, no drift, no valid updates)
    - _Requirements: 1.1, 1.2, 1.3, 2.5, 3.6, 6.5, 7.7, 8.1, 8.2, 8.3_

  - [ ] 11.2 Write unit tests for ScribeAgent orchestration
    - Test early-exit paths: disabled config, non-code PR, no drift, no valid updates
    - Test happy path with mocked components
    - _Requirements: 1.3, 2.5, 3.6, 6.5, 7.7, 8.1_

- [ ] 12. Create GitHub Actions workflow
  - [ ] 12.1 Create `.github/workflows/scribe-doc-sync.yml`
    - Configure `pull_request` trigger with `types: [closed]` and merge condition
    - Install OpenHands, invoke `openhands/scribe/resolve_docs.py` with appropriate arguments
    - Upload `scribe_output.json` as a workflow artifact
    - _Requirements: 1.1, 1.2_

- [ ] 13. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties using `hypothesis`
- Unit tests validate specific examples and edge cases using `pytest`
- The implementation reuses existing OpenHands infrastructure (`Runtime`, `AgentController`, `send_pull_request`, `GithubIssueHandler`) rather than reimplementing
