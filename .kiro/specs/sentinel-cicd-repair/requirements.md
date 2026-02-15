# Requirements Document

## Introduction

Project Sentinel is an autonomous CI/CD repair agent that monitors CI pipeline failures, analyzes build logs and stack traces, reproduces failures in a sandbox environment, generates fixes, and commits them back to the repository. It integrates into existing GitHub Actions workflows and leverages the OpenHands runtime, agent controller, and resolver patterns to provide zero-touch resolution for common build failures such as linting errors, test failures from typos, and dependency conflicts.

## Glossary

- **Sentinel_Agent**: The autonomous CI/CD repair agent that orchestrates the full failure analysis, reproduction, fix, and commit cycle.
- **CI_Log_Parser**: The component responsible for extracting structured failure information from raw CI build logs and stack traces.
- **Failure_Classifier**: The component that categorizes parsed CI failures into known failure types (lint, test, dependency, build) and determines repairability.
- **Sandbox_Reproducer**: The component that spins up an isolated OpenHands runtime sandbox to reproduce a CI failure locally before attempting a fix.
- **Fix_Generator**: The component that uses the OpenHands agent controller to generate code patches that resolve the identified failure.
- **Fix_Verifier**: The component that re-runs the failed CI step inside the sandbox to confirm the generated fix resolves the failure.
- **Commit_Pusher**: The component that commits and pushes verified fixes back to the repository branch.
- **Sentinel_Workflow**: The GitHub Actions workflow YAML that triggers Sentinel on CI failure events.
- **Failure_Report**: A structured data object containing parsed failure type, affected files, error messages, and stack traces extracted from CI logs.
- **Repair_Attempt**: A single end-to-end cycle of analysis, reproduction, fix generation, verification, and commit.

## Requirements

### Requirement 1: CI Failure Detection and Triggering

**User Story:** As a developer, I want Sentinel to automatically detect when my CI pipeline fails, so that repair can begin without manual intervention.

#### Acceptance Criteria

1. WHEN a GitHub Actions workflow run completes with a failure status, THE Sentinel_Workflow SHALL trigger the Sentinel_Agent with the failed run ID and repository context.
2. WHEN the Sentinel_Workflow is triggered, THE Sentinel_Agent SHALL retrieve the full build logs for the failed workflow run using the GitHub API.
3. IF the GitHub API is unreachable or returns an error, THEN THE Sentinel_Agent SHALL log the error and terminate the repair attempt gracefully.
4. WHEN the Sentinel_Workflow is triggered, THE Sentinel_Agent SHALL identify the specific failed job and step within the workflow run.

### Requirement 2: Build Log Parsing and Failure Classification

**User Story:** As a developer, I want Sentinel to understand why my CI failed, so that it can attempt an appropriate fix.

#### Acceptance Criteria

1. WHEN build logs are retrieved, THE CI_Log_Parser SHALL extract structured Failure_Report objects containing error messages, stack traces, affected file paths, and line numbers.
2. WHEN a Failure_Report is produced, THE Failure_Classifier SHALL categorize the failure into one of the supported types: lint_error, test_failure, dependency_conflict, build_error, or unknown.
3. WHEN the Failure_Classifier categorizes a failure as "unknown", THE Sentinel_Agent SHALL skip the repair attempt and report the failure as unclassifiable.
4. THE CI_Log_Parser SHALL produce a Failure_Report for each distinct error found in the build logs.
5. WHEN a Failure_Report is produced, THE CI_Log_Parser SHALL serialize the Failure_Report to JSON for storage and downstream consumption.
6. FOR ALL valid Failure_Report JSON strings, parsing then serializing SHALL produce an equivalent JSON string (round-trip property).

### Requirement 3: Failure Reproduction in Sandbox

**User Story:** As a developer, I want Sentinel to reproduce the CI failure before attempting a fix, so that fixes are validated against the actual failure.

#### Acceptance Criteria

1. WHEN a classifiable Failure_Report is produced, THE Sandbox_Reproducer SHALL spin up an OpenHands runtime sandbox with the repository checked out at the failing commit.
2. WHEN the sandbox is ready, THE Sandbox_Reproducer SHALL execute the failed CI command (e.g., `pytest tests/test_auth.py`, `ruff check`) inside the sandbox.
3. WHEN the failed CI command exits with a non-zero status in the sandbox, THE Sandbox_Reproducer SHALL mark the failure as successfully reproduced.
4. IF the failed CI command exits with a zero status in the sandbox, THEN THE Sentinel_Agent SHALL skip the repair attempt and report the failure as non-reproducible.
5. IF the sandbox fails to initialize, THEN THE Sentinel_Agent SHALL log the error and terminate the repair attempt gracefully.

### Requirement 4: Automated Fix Generation

**User Story:** As a developer, I want Sentinel to generate a code fix for the reproduced failure, so that the CI can be repaired without my intervention.

#### Acceptance Criteria

1. WHEN a failure is successfully reproduced, THE Fix_Generator SHALL use the OpenHands agent controller to generate a code patch that addresses the Failure_Report.
2. WHEN generating a fix, THE Fix_Generator SHALL provide the agent with the Failure_Report, the relevant source files, and the failed command output as context.
3. WHEN the agent produces a code patch, THE Fix_Generator SHALL extract the patch as a unified diff.
4. IF the agent fails to produce a patch within the configured maximum iterations, THEN THE Sentinel_Agent SHALL terminate the repair attempt and report the failure.
5. WHEN generating a fix, THE Fix_Generator SHALL constrain the agent to modify only files referenced in the Failure_Report.

### Requirement 5: Fix Verification

**User Story:** As a developer, I want Sentinel to verify that its fix actually resolves the CI failure, so that only validated fixes are committed.

#### Acceptance Criteria

1. WHEN a code patch is generated, THE Fix_Verifier SHALL apply the patch inside the sandbox environment.
2. WHEN the patch is applied, THE Fix_Verifier SHALL re-run the originally failed CI command inside the sandbox.
3. WHEN the re-run CI command exits with a zero status, THE Fix_Verifier SHALL mark the fix as verified.
4. IF the re-run CI command exits with a non-zero status, THEN THE Sentinel_Agent SHALL discard the fix and report the repair attempt as unsuccessful.
5. WHEN verifying a fix, THE Fix_Verifier SHALL run the full test suite (not just the failed test) to ensure no regressions are introduced.

### Requirement 6: Fix Commit and Push

**User Story:** As a developer, I want Sentinel to commit verified fixes to my branch, so that the CI pipeline can re-run and pass.

#### Acceptance Criteria

1. WHEN a fix is verified, THE Commit_Pusher SHALL create a commit with the message format `fix: auto-repair ci failure in <failed_step>`.
2. WHEN creating a commit, THE Commit_Pusher SHALL attribute the commit to the Sentinel bot user.
3. WHEN the commit is created, THE Commit_Pusher SHALL push the commit to the same branch that triggered the CI failure.
4. IF the push fails due to a conflict or permission error, THEN THE Commit_Pusher SHALL log the error and report the repair attempt as unsuccessful.
5. WHEN a fix is committed, THE Sentinel_Agent SHALL post a comment on the associated pull request (if one exists) summarizing the repair.

### Requirement 7: Repair Attempt Reporting

**User Story:** As a developer, I want to see a summary of what Sentinel did, so that I can understand and review the automated repair.

#### Acceptance Criteria

1. WHEN a Repair_Attempt completes (success or failure), THE Sentinel_Agent SHALL produce a structured report containing the failure type, actions taken, and outcome.
2. WHEN a Repair_Attempt succeeds, THE Sentinel_Agent SHALL include the diff of the applied fix in the report.
3. WHEN a Repair_Attempt fails, THE Sentinel_Agent SHALL include the reason for failure in the report.
4. WHEN a pull request exists for the failing branch, THE Sentinel_Agent SHALL post the repair report as a PR comment.
5. THE Sentinel_Agent SHALL upload the full repair report as a GitHub Actions artifact for audit purposes.

### Requirement 8: Configuration and Safety Guards

**User Story:** As a repository maintainer, I want to configure Sentinel's behavior and set safety limits, so that automated repairs stay within acceptable bounds.

#### Acceptance Criteria

1. THE Sentinel_Agent SHALL read configuration from a `.sentinel.yml` file in the repository root.
2. WHEN no `.sentinel.yml` file exists, THE Sentinel_Agent SHALL use default configuration values.
3. WHERE the configuration specifies `max_iterations`, THE Sentinel_Agent SHALL limit the agent to that number of iterations during fix generation.
4. WHERE the configuration specifies `allowed_failure_types`, THE Sentinel_Agent SHALL only attempt repairs for the listed failure types.
5. WHERE the configuration specifies `excluded_paths`, THE Sentinel_Agent SHALL prevent the Fix_Generator from modifying files matching those path patterns.
6. WHERE the configuration specifies `auto_commit: false`, THE Sentinel_Agent SHALL generate the fix but skip the commit and push step.
7. THE Sentinel_Agent SHALL enforce a maximum file change limit of 10 files per repair attempt to prevent large-scale unintended modifications.
8. WHEN the configuration file contains invalid YAML, THE Sentinel_Agent SHALL log a warning and fall back to default configuration values.
