# Implementation Plan: Gatekeeper (Semantic Code Review)

## Overview

Implement the Gatekeeper semantic code review agent as a new `openhands/gatekeeper/` module following the same patterns as `openhands/resolver/`. Tasks are ordered to build core data models first, then config/rule loading, then code analysis, then deduplication/filtering, then comment posting, and finally the GitHub Actions workflow and orchestration.

## Tasks

- [ ] 1. Set up module structure and data models
  - [ ] 1.1 Create `openhands/gatekeeper/__init__.py` and `openhands/gatekeeper/models.py`
    - Define `Severity` enum, `RuleCategory` enum, `Rule`, `RuleSet`, `ChangedFile`, `Finding`, `FindingSet`, `ReviewSummary`, and `GatekeeperOutput` Pydantic models
    - Implement JSON serialization/deserialization using Pydantic's `model_dump_json()` / `model_validate_json()`
    - _Requirements: 2.4, 2.5, 2.6, 3.5, 3.6, 3.7, 7.2, 7.4_

  - [ ] 1.2 Write property test for RuleSet round-trip serialization
    - **Property 1: Rule_Set round-trip serialization**
    - **Validates: Requirements 2.5, 2.6**

  - [ ] 1.3 Write property test for FindingSet round-trip serialization
    - **Property 2: Finding_Set round-trip serialization**
    - **Validates: Requirements 3.6, 3.7**

  - [ ] 1.4 Write property test for GatekeeperOutput round-trip serialization
    - **Property 3: Gatekeeper_Output round-trip serialization**
    - **Validates: Requirements 7.4**

- [ ] 2. Implement configuration loading
  - [ ] 2.1 Create `openhands/gatekeeper/config.py`
    - Implement `GatekeeperConfig` dataclass with fields: `enabled`, `cortex_id`, `categories`, `min_severity`, `ignore_paths`, `custom_rules`, `max_comments`
    - Implement `GatekeeperConfig.load(repo_dir)` that reads `.gatekeeper.yml`, validates fields, falls back to defaults on missing/invalid file
    - Implement `to_dict()` and `from_dict()` methods
    - _Requirements: 6.1, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8, 1.5, 1.6_

  - [ ] 2.2 Write property test for config parsing with defaults
    - **Property 12: Config parsing with defaults**
    - **Validates: Requirements 6.1, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8**

  - [ ] 2.3 Write unit tests for config loading edge cases
    - Test missing file (defaults), invalid YAML (fallback), partial config, `enabled=false` behavior
    - _Requirements: 1.5, 1.6, 6.1, 6.2_

- [ ] 3. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement Cortex loader and rule management
  - [ ] 4.1 Create `openhands/gatekeeper/cortex_loader.py`
    - Implement `CortexLoader` class with `load_rules()`, `merge_custom_rules()`, and `get_default_rules()` methods
    - Define built-in default rules for style (print usage, naming), security (SQL injection, hardcoded secrets, eval/exec, shell=True), and complexity (function length, nesting depth, parameter count)
    - Implement rule merging where custom rules override base rules with the same id
    - _Requirements: 2.1, 2.2, 2.3, 2.4_

  - [ ] 4.2 Write property test for custom rules precedence on merge
    - **Property 4: Custom rules take precedence on merge**
    - **Validates: Requirements 2.3**

  - [ ] 4.3 Write property test for rule category validation
    - **Property 5: All rules have valid categories**
    - **Validates: Requirements 2.4**

  - [ ] 4.4 Write unit tests for Cortex loader
    - Test Cortex unavailable fallback, custom rule merging with specific examples, default rules content
    - _Requirements: 2.1, 2.2, 2.3_

- [ ] 5. Implement PR diff fetcher
  - [ ] 5.1 Create `openhands/gatekeeper/diff_fetcher.py`
    - Implement `PRDiffFetcher` class that uses `GithubPRHandler` from `openhands/resolver/interfaces/github.py` to retrieve PR diffs
    - Implement `detect_language()` based on file extension
    - Parse diff to extract `ChangedFile` objects with path, patch, content, and language
    - _Requirements: 1.3_

  - [ ] 5.2 Write unit tests for diff fetcher
    - Test language detection, diff parsing with sample PR data, API error handling
    - _Requirements: 1.3, 1.4_

- [ ] 6. Implement code analyzer
  - [ ] 6.1 Create `openhands/gatekeeper/code_analyzer.py`
    - Implement `CodeAnalyzer` class that uses the OpenHands `AgentController` to analyze changed files against the Rule_Set
    - Construct prompts containing the Rule_Set, changed file content, diff, and category-specific guidance
    - Parse agent output into structured `Finding` objects
    - Skip unsupported file languages
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.8_

  - [ ] 6.2 Write property test for finding field completeness
    - **Property 6: Findings contain all required fields**
    - **Validates: Requirements 3.5**

  - [ ] 6.3 Write unit tests for code analyzer
    - Test with sample code containing print() usage, SQL injection pattern, long function
    - Test unsupported language skip behavior
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.8_

- [ ] 7. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Implement deduplicator and filtering
  - [ ] 8.1 Create `openhands/gatekeeper/deduplicator.py`
    - Implement `Deduplicator` class with `deduplicate()` method
    - Implement `_remove_duplicates()` using `(file_path, line_number, rule_id)` as dedup key
    - Implement `_filter_unchanged_lines()` that parses unified diff to extract changed line numbers and removes findings on unchanged lines
    - Implement `_filter_severity()` that removes findings below the configured minimum severity
    - Implement `_filter_ignored_paths()` that removes findings matching ignore glob patterns
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

  - [ ] 8.2 Write property test for deduplication
    - **Property 7: Deduplication removes exact duplicates**
    - **Validates: Requirements 4.1**

  - [ ] 8.3 Write property test for filtering criteria
    - **Property 8: Filtering respects all criteria**
    - **Validates: Requirements 4.2, 4.3, 4.4**

- [ ] 9. Implement comment poster
  - [ ] 9.1 Create `openhands/gatekeeper/comment_poster.py`
    - Implement `CommentPoster` class with `post_review()`, `format_comment()`, `format_summary()`, and `prioritize_findings()` methods
    - Use GitHub pull request review API to batch inline comments into a single review
    - Implement truncation logic: sort by severity descending, take top `max_comments`, note truncation in summary
    - Format comments with category emoji, severity, explanation, suggested fix, and rule id
    - Generate `ReviewSummary` with correct counts by category and severity
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 6.9_

  - [ ] 9.2 Write property test for comment formatting
    - **Property 9: Comment formatting includes all required information**
    - **Validates: Requirements 5.1, 5.2**

  - [ ] 9.3 Write property test for review summary counts
    - **Property 10: Review_Summary has correct counts**
    - **Validates: Requirements 5.3**

  - [ ] 9.4 Write property test for truncation behavior
    - **Property 11: Truncation respects max_comments with severity ordering**
    - **Validates: Requirements 6.9**

  - [ ] 9.5 Write unit tests for comment poster
    - Test empty findings summary, comment formatting with specific examples, API error handling
    - _Requirements: 5.4, 5.6_

- [ ] 10. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 11. Wire components together in the Gatekeeper Agent entry point
  - [ ] 11.1 Create `openhands/gatekeeper/review_pr.py`
    - Implement `GatekeeperAgent` class that orchestrates the full pipeline: config loading → diff fetching → rule loading → code analysis → deduplication → comment posting
    - Implement CLI argument parsing analogous to `openhands/resolver/resolve_issue.py`
    - Write `GatekeeperOutput` to the output directory as `gatekeeper_output.json`
    - Handle all early-exit conditions (disabled, API error, no findings)
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 6.2, 7.1, 7.2, 7.3_

  - [ ] 11.2 Write unit tests for GatekeeperAgent orchestration
    - Test early-exit paths: disabled config, API error, no findings
    - Test happy path with mocked components
    - _Requirements: 1.4, 6.2, 7.1_

- [ ] 12. Create GitHub Actions workflow
  - [ ] 12.1 Create `.github/workflows/gatekeeper-review.yml`
    - Configure `pull_request` trigger with `types: [opened, synchronize]`
    - Install OpenHands, invoke `openhands/gatekeeper/review_pr.py` with appropriate arguments
    - Upload `gatekeeper_output.json` as a workflow artifact
    - _Requirements: 1.1, 1.2_

- [ ] 13. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties using `hypothesis`
- Unit tests validate specific examples and edge cases using `pytest`
- The implementation reuses existing OpenHands infrastructure (`Runtime`, `AgentController`, `GithubPRHandler`) rather than reimplementing
- Test files go in `tests/unit/test_gatekeeper/` following the existing test organization pattern
