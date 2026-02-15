# Implementation Plan: Project Sentinel (Autonomous CI/CD Repair)

## Overview

Implement the Sentinel CI/CD repair agent as a new `openhands/sentinel/` module following the same patterns as `openhands/resolver/`. Tasks are ordered to build core data models first, then parsing/classification, then sandbox integration, then fix generation/verification, then commit/reporting, and finally the GitHub Actions workflow.

## Tasks

- [ ] 1. Set up module structure and data models
  - [ ] 1.1 Create `openhands/sentinel/__init__.py` and `openhands/sentinel/models.py`
    - Define `FailureType` enum, `FailureReport`, `ReproductionResult`, `FixResult`, `VerificationResult`, `PushResult`, `RepairAttemptStatus`, `RepairAttempt`, `SentinelOutput`, and `SentinelConfig` Pydantic models
    - Implement `FailureReport.to_json()` and `FailureReport.from_json()` methods
    - _Requirements: 2.1, 2.5, 2.6, 8.1, 8.7_

  - [ ] 1.2 Write property test for FailureReport round-trip serialization
    - **Property 3: FailureReport round-trip serialization**
    - **Validates: Requirements 2.5, 2.6**

  - [ ] 1.3 Write property test for Classifier returns valid FailureType
    - **Property 2: Classifier returns valid FailureType**
    - **Validates: Requirements 2.2**

- [ ] 2. Implement configuration loading
  - [ ] 2.1 Create `openhands/sentinel/config.py`
    - Implement `SentinelConfig.load()` class method that reads `.sentinel.yml`, validates it, and falls back to defaults on missing/invalid files
    - _Requirements: 8.1, 8.2, 8.8_

  - [ ] 2.2 Write unit tests for config loading
    - Test valid YAML, missing file (defaults), invalid YAML (fallback), and each config field
    - _Requirements: 8.1, 8.2, 8.8_

  - [ ] 2.3 Write property test for allowed failure types filtering
    - **Property 10: Allowed failure types filtering**
    - **Validates: Requirements 8.4**

  - [ ] 2.4 Write property test for excluded paths filtering
    - **Property 11: Excluded paths filtering**
    - **Validates: Requirements 8.5**

  - [ ] 2.5 Write property test for max file change limit
    - **Property 12: Max file change limit**
    - **Validates: Requirements 8.7**

- [ ] 3. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement CI log fetching and parsing
  - [ ] 4.1 Create `openhands/sentinel/log_fetcher.py`
    - Implement `CILogFetcher` class with `fetch_logs()` and `fetch_failed_jobs()` methods using the GitHub Actions API
    - _Requirements: 1.2, 1.3, 1.4_

  - [ ] 4.2 Create `openhands/sentinel/log_parser.py`
    - Implement `CILogParser.parse()` with regex patterns for pytest tracebacks, ruff/mypy output, npm/tsc errors, and dependency resolution errors
    - Return a list of `FailureReport` objects, one per distinct error
    - _Requirements: 2.1, 2.4_

  - [ ] 4.3 Create `openhands/sentinel/failure_classifier.py`
    - Implement `FailureClassifier.classify()` that categorizes a `FailureReport` into a `FailureType` based on error patterns and the failed command
    - _Requirements: 2.2, 2.3_

  - [ ] 4.4 Write property test for failed job identification
    - **Property 1: Failed job identification**
    - **Validates: Requirements 1.4**

  - [ ] 4.5 Write unit tests for log parser
    - Test with sample pytest traceback, ruff output, tsc error, and dependency conflict logs
    - _Requirements: 2.1, 2.4_

- [ ] 5. Implement sandbox reproduction
  - [ ] 5.1 Create `openhands/sentinel/reproducer.py`
    - Implement `SandboxReproducer` that uses the OpenHands `Runtime` to checkout the failing commit and execute the failed command
    - Map exit codes to `ReproductionResult` (non-zero → reproduced=True, zero → reproduced=False)
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ] 5.2 Write property test for reproduction exit code mapping
    - **Property 4: Reproduction exit code mapping**
    - **Validates: Requirements 3.3, 3.4**

- [ ] 6. Implement fix generation
  - [ ] 6.1 Create `openhands/sentinel/fix_generator.py`
    - Implement `FixGenerator` that constructs a prompt from the `FailureReport` and reproduction output, invokes the `AgentController`, and extracts the patch
    - Enforce file modification constraints (only affected files, excluded paths, max file changes)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 8.3, 8.5, 8.7_

  - [ ] 6.2 Write property test for prompt completeness
    - **Property 5: Prompt completeness**
    - **Validates: Requirements 4.2**

  - [ ] 6.3 Write property test for modified files constraint
    - **Property 6: Modified files constraint**
    - **Validates: Requirements 4.5**

- [ ] 7. Implement fix verification
  - [ ] 7.1 Create `openhands/sentinel/verifier.py`
    - Implement `FixVerifier` that applies the patch in the sandbox, re-runs the failed command, and runs the full test suite
    - Map exit codes to `VerificationResult` (zero → verified=True, non-zero → verified=False)
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [ ] 7.2 Write property test for verification exit code mapping
    - **Property 7: Verification exit code mapping**
    - **Validates: Requirements 5.3, 5.4**

- [ ] 8. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Implement commit pushing
  - [ ] 9.1 Create `openhands/sentinel/commit_pusher.py`
    - Implement `CommitPusher` that reuses `openhands.resolver.send_pull_request.make_commit` to create and push the fix commit
    - Format commit message as `fix: auto-repair ci failure in <failed_step>`
    - Attribute commit to Sentinel bot user
    - _Requirements: 6.1, 6.2, 6.3, 6.4_

  - [ ] 9.2 Write property test for commit message format
    - **Property 8: Commit message format**
    - **Validates: Requirements 6.1**

- [ ] 10. Implement reporting
  - [ ] 10.1 Create `openhands/sentinel/reporter.py`
    - Implement `ReportGenerator` that produces `SentinelOutput` from `RepairAttempt` objects
    - Include diff on success, failure reason on failure
    - Implement `post_pr_comment()` for posting reports to PRs
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_

  - [ ] 10.2 Write property test for report completeness
    - **Property 9: Report completeness**
    - **Validates: Requirements 7.1, 7.2, 7.3**

- [ ] 11. Implement the Sentinel Agent orchestrator
  - [ ] 11.1 Create `openhands/sentinel/resolve_ci_failure.py`
    - Implement `SentinelAgent` class that orchestrates the full pipeline: fetch logs → parse → classify → check config → reproduce → generate fix → verify → commit → report
    - Implement CLI entry point with argparse (analogous to `resolve_issue.py`)
    - Handle all error/skip paths (unknown classification, non-reproducible, fix failed, etc.)
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.3, 3.4, 3.5, 4.4, 5.4, 6.4, 6.5, 8.2, 8.3, 8.4, 8.6_

- [ ] 12. Create the GitHub Actions workflow
  - [ ] 12.1 Create `.github/workflows/sentinel-ci-repair.yml`
    - Define reusable workflow triggered by `workflow_run` with `conclusion: failure`
    - Install OpenHands, invoke `resolve_ci_failure.py`, upload artifacts, post PR comments
    - Follow the same patterns as `openhands-resolver.yml`
    - _Requirements: 1.1, 6.5, 7.4, 7.5_

- [ ] 13. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- The module follows the same patterns as `openhands/resolver/` for consistency
- Property tests use `hypothesis` with minimum 100 iterations per test
- Unit tests use `pytest` with mocked external dependencies (GitHub API, Runtime)
