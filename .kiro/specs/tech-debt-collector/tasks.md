# Implementation Plan: Tech Debt Collector ("Refactor")

## Overview

Implement the Tech Debt Collector as a new `openhands/refactor/` module following the same patterns as `openhands/resolver/`. Tasks build core data models first, then scanning/auditing, then prioritization, then fix generation/verification, then PR creation/reporting, and finally the GitHub Actions workflow. Python is the implementation language, consistent with the existing backend.

## Tasks

- [ ] 1. Set up module structure and data models
  - [ ] 1.1 Create `openhands/refactor/__init__.py` and `openhands/refactor/models.py`
    - Define `ItemType` enum, `ItemOutcome` enum, `DeclaredDependency`, `RegistryInfo`, `TechDebtItem`, `FixResult`, `VerificationResult`, `PRResult`, `ItemReport`, and `SummaryReport` Pydantic models
    - Implement `TechDebtItem.to_json()` and `TechDebtItem.from_json()` class methods
    - Implement `SummaryReport.to_json()` and `SummaryReport.from_json()` class methods
    - _Requirements: 1.6, 1.7, 7.6_

  - [ ] 1.2 Write property test for TechDebtItem JSON round-trip
    - **Property 1: TechDebtItem JSON round-trip**
    - **Validates: Requirements 1.6, 1.7**

  - [ ] 1.3 Write property test for SummaryReport completeness
    - **Property 13: Summary report completeness**
    - **Validates: Requirements 7.6**

- [ ] 2. Implement configuration loading
  - [ ] 2.1 Create `openhands/refactor/config.py`
    - Define `RefactorConfig` Pydantic model with all fields and defaults
    - Implement `RefactorConfig.load()` class method that reads `.refactor.yml`, validates, and falls back to defaults
    - Implement `RefactorConfig.to_yaml()` and `RefactorConfig.from_yaml()` methods
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6_

  - [ ] 2.2 Write property test for RefactorConfig YAML round-trip
    - **Property 14: RefactorConfig YAML round-trip**
    - **Validates: Requirements 8.5, 8.6**

  - [ ] 2.3 Write unit tests for config loading
    - Test valid YAML loading, missing file (defaults), invalid YAML (fallback), and all config fields present
    - _Requirements: 8.1, 8.2, 8.3, 8.4_

- [ ] 3. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement TODO scanning
  - [ ] 4.1 Create `openhands/refactor/todo_scanner.py`
    - Implement `TODOScanner` class with `scan()`, `parse_file()`, and `extract_todo()` methods
    - Support Python (`#`), JavaScript/TypeScript (`//`, `/* */`), and shell (`#`) comment syntaxes
    - Respect `.gitignore` and `exclude_paths` from config
    - Extract author tags from `TODO(author)` format
    - Extract up to 10 lines of surrounding context
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

  - [ ] 4.2 Write property test for TODO extraction correctness
    - **Property 2: TODO extraction correctness**
    - **Validates: Requirements 1.2, 1.3**

  - [ ] 4.3 Write property test for multi-syntax TODO detection
    - **Property 3: Multi-syntax TODO detection**
    - **Validates: Requirements 1.4**

  - [ ] 4.4 Write property test for exclusion pattern filtering
    - **Property 4: Exclusion pattern filtering**
    - **Validates: Requirements 1.1**

- [ ] 5. Implement dependency auditing
  - [ ] 5.1 Create `openhands/refactor/dependency_auditor.py`
    - Implement `DependencyAuditor` class with `audit()`, `parse_pyproject()`, `parse_package_json()`, `parse_requirements_txt()` methods
    - Implement `check_pypi()` and `check_npm()` async methods for registry queries
    - Implement `is_outdated()` method with threshold logic (>1 major or >3 minor versions behind)
    - Handle unreachable registries gracefully (log and skip)
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

  - [ ] 5.2 Write property test for dependency manifest parsing
    - **Property 5: Dependency manifest parsing**
    - **Validates: Requirements 2.1**

  - [ ] 5.3 Write property test for outdated version detection threshold
    - **Property 6: Outdated version detection threshold**
    - **Validates: Requirements 2.3**

- [ ] 6. Implement prioritization
  - [ ] 6.1 Create `openhands/refactor/prioritizer.py`
    - Implement `Prioritizer` class with `prioritize()`, `score_dependency()`, and `score_todo()` methods
    - Security advisory weighting: +100 for advisory, +20 per major version lag, +5 per minor version lag
    - TODO scoring: +0.1 per day age (capped at 50), +0-20 for code complexity heuristic
    - Sort items by Priority_Score in descending order
    - Respect `max_items_per_run` from config
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ] 6.2 Write property test for security advisory priority boost
    - **Property 7: Security advisory priority boost**
    - **Validates: Requirements 2.4, 3.2**

  - [ ] 6.3 Write property test for priority score assignment and descending sort
    - **Property 8: Priority score assignment and descending sort**
    - **Validates: Requirements 3.1, 3.4**

  - [ ] 6.4 Write property test for max items per run limit
    - **Property 9: Max items per run limit**
    - **Validates: Requirements 3.5**

- [ ] 7. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Implement fix generation
  - [ ] 8.1 Create `openhands/refactor/fix_generator.py`
    - Implement `FixGenerator` class with `generate_fix()` and `build_prompt()` methods
    - Use OpenHands `AgentController` and `Runtime` for TODO fix generation
    - For dependency items, update version in manifest and regenerate lock files
    - Generate branch names matching pattern `refactor/<item-type>/<short-description>`
    - Handle max iteration failures gracefully
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [ ] 8.2 Write property test for branch name format
    - **Property 10: Branch name format**
    - **Validates: Requirements 5.3**

- [ ] 9. Implement fix verification
  - [ ] 9.1 Create `openhands/refactor/fix_verifier.py`
    - Implement `FixVerifier` class with `verify()` and `detect_test_command()` methods
    - Apply patch in OpenHands runtime sandbox
    - Execute test suite and map exit codes to `VerificationResult`
    - Detect test command from project structure when not configured
    - Handle sandbox initialization failures gracefully
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [ ] 9.2 Write property test for exit code to verification mapping
    - **Property 11: Exit code to verification mapping**
    - **Validates: Requirements 6.3, 6.4**

- [ ] 10. Implement PR creation and reporting
  - [ ] 10.1 Create `openhands/refactor/pr_creator.py`
    - Implement `PRCreator` class with `create_pr()`, `generate_branch_name()`, `generate_pr_title()`, and `generate_pr_body()` methods
    - Use existing `send_pull_request` utility from `openhands/resolver/`
    - Apply configurable labels and assign reviewers from config
    - Generate descriptive PR titles and bodies with original debt info, changes, and test results
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_

  - [ ] 10.2 Write property test for PR content completeness
    - **Property 12: PR content completeness**
    - **Validates: Requirements 7.2, 7.3**

- [ ] 11. Implement agent orchestrator and entry point
  - [ ] 11.1 Create `openhands/refactor/resolve_tech_debt.py`
    - Implement `RefactorAgent` class with `run()` and `process_item()` methods
    - Wire together: config loading → scanning → auditing → prioritization → fix generation → verification → PR creation
    - Implement lock file mechanism to prevent concurrent runs
    - Produce `SummaryReport` at the end of each run
    - Implement CLI entry point with argparse (analogous to `openhands/resolver/resolve_issue.py`)
    - _Requirements: 4.4, 4.5, 7.6_

  - [ ] 11.2 Write unit tests for lock file mechanism
    - Test lock file creation, detection, and cleanup
    - _Requirements: 4.4, 4.5_

- [ ] 12. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Create GitHub Actions workflow
  - [ ] 13.1 Create `.github/workflows/refactor-tech-debt.yml`
    - Define cron-triggered workflow (default: `0 2 * * 6`)
    - Install OpenHands dependencies
    - Invoke `openhands/refactor/resolve_tech_debt.py` with appropriate arguments
    - Upload summary report as workflow artifact
    - Support configurable inputs: `LLM_MODEL`, `max_items_per_run`
    - Use secrets: `LLM_API_KEY`, `PAT_TOKEN`
    - _Requirements: 4.1, 4.2, 4.3_

- [ ] 14. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties using Hypothesis
- Unit tests validate specific examples and edge cases using pytest
- The module follows the same patterns as `openhands/resolver/` for consistency
