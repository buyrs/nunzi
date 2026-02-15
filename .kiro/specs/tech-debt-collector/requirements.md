# Requirements Document

## Introduction

The Tech Debt Collector ("Refactor") is a background agent for the Nunzi platform that passively improves codebases over time. It scans repositories for TODO comments and outdated dependencies, then autonomously creates branches, implements fixes, runs tests in the OpenHands sandbox, and opens pull requests. It is designed to run during off-peak hours (nights and weekends) so that the codebase improves without developer intervention.

## Glossary

- **Refactor_Agent**: The top-level background agent that orchestrates scanning, fix generation, verification, and PR creation for tech debt items.
- **TODO_Scanner**: The component that parses source files to extract TODO comments along with their file path, line number, and surrounding context.
- **Dependency_Auditor**: The component that inspects dependency manifests (e.g., `pyproject.toml`, `package.json`) and compares installed versions against the latest available versions to identify outdated or vulnerable packages.
- **Tech_Debt_Item**: A structured data object representing a single unit of tech debt, either a TODO comment or an outdated dependency, including its location, description, and priority.
- **Scheduler**: The component that triggers Refactor_Agent runs based on configurable cron schedules (e.g., nights and weekends).
- **Fix_Generator**: The component that uses the OpenHands AgentController to generate code changes that resolve a given Tech_Debt_Item.
- **Fix_Verifier**: The component that runs the project test suite inside an OpenHands runtime sandbox to confirm a generated fix does not introduce regressions.
- **PR_Creator**: The component that creates a Git branch, commits verified changes, and opens a pull request with a descriptive summary of the resolved tech debt.
- **Refactor_Config**: A YAML configuration file (`.refactor.yml`) in the repository root that controls scanning scope, scheduling, dependency policies, and PR behavior.
- **Scan_Report**: A structured data object containing all Tech_Debt_Items discovered in a single scan run, with metadata about the scan timestamp and repository state.
- **Priority_Score**: A numeric value assigned to each Tech_Debt_Item indicating its urgency, computed from factors like dependency version lag, security advisories, and TODO age.

## Requirements

### Requirement 1: TODO Comment Scanning

**User Story:** As a developer, I want the agent to find all TODO comments in my codebase, so that forgotten tasks are surfaced and addressed automatically.

#### Acceptance Criteria

1. WHEN a scan is initiated, THE TODO_Scanner SHALL recursively traverse all source files in the repository, respecting `.gitignore` and Refactor_Config exclusion patterns.
2. WHEN a TODO comment is found, THE TODO_Scanner SHALL extract the file path, line number, the full comment text, and up to 10 lines of surrounding context.
3. WHEN a TODO comment contains an author tag (e.g., `# TODO(alice): fix this`), THE TODO_Scanner SHALL extract the author name into the Tech_Debt_Item.
4. THE TODO_Scanner SHALL support comment syntaxes for Python (`#`), JavaScript/TypeScript (`//`, `/* */`), and shell scripts (`#`).
5. WHEN scanning is complete, THE TODO_Scanner SHALL produce a Scan_Report containing all discovered Tech_Debt_Items.
6. WHEN a TODO comment is found, THE TODO_Scanner SHALL serialize the resulting Tech_Debt_Item to JSON.
7. FOR ALL valid Tech_Debt_Item JSON strings, parsing then serializing SHALL produce an equivalent JSON string (round-trip property).

### Requirement 2: Dependency Auditing

**User Story:** As a developer, I want the agent to detect outdated and vulnerable dependencies, so that my project stays current and secure without manual checking.

#### Acceptance Criteria

1. WHEN a scan is initiated, THE Dependency_Auditor SHALL parse dependency manifests (`pyproject.toml`, `package.json`, `requirements.txt`) to extract declared dependencies and their version constraints.
2. WHEN dependencies are extracted, THE Dependency_Auditor SHALL query package registries (PyPI, npm) to determine the latest available version for each dependency.
3. WHEN a dependency is more than one major version or three minor versions behind the latest, THE Dependency_Auditor SHALL flag the dependency as outdated and create a Tech_Debt_Item.
4. WHEN a dependency has a known security advisory (via GitHub Advisory Database or equivalent), THE Dependency_Auditor SHALL flag the dependency as vulnerable and assign a higher Priority_Score.
5. IF a package registry is unreachable, THEN THE Dependency_Auditor SHALL log the error, skip the unreachable registry, and continue auditing remaining dependencies.
6. WHEN dependency auditing is complete, THE Dependency_Auditor SHALL add all discovered dependency Tech_Debt_Items to the Scan_Report.

### Requirement 3: Tech Debt Prioritization

**User Story:** As a developer, I want the agent to prioritize which tech debt items to fix first, so that the most impactful improvements are made during limited off-peak windows.

#### Acceptance Criteria

1. WHEN a Scan_Report is produced, THE Refactor_Agent SHALL assign a Priority_Score to each Tech_Debt_Item.
2. WHEN computing Priority_Score for a dependency item, THE Refactor_Agent SHALL weight security vulnerabilities higher than version staleness.
3. WHEN computing Priority_Score for a TODO item, THE Refactor_Agent SHALL consider the age of the TODO (based on git blame) and the complexity of the surrounding code.
4. THE Refactor_Agent SHALL sort Tech_Debt_Items by Priority_Score in descending order before processing.
5. WHERE a Refactor_Config specifies a `max_items_per_run` limit, THE Refactor_Agent SHALL process only the top N items by Priority_Score.

### Requirement 4: Scheduling and Triggering

**User Story:** As a developer, I want the agent to run during off-peak hours, so that automated maintenance does not interfere with active development.

#### Acceptance Criteria

1. THE Scheduler SHALL support cron-based scheduling via a GitHub Actions workflow or equivalent trigger mechanism.
2. WHEN no Refactor_Config schedule is specified, THE Scheduler SHALL default to running at 2:00 AM UTC on Saturdays.
3. WHERE a Refactor_Config specifies a custom cron schedule, THE Scheduler SHALL use the configured schedule instead of the default.
4. WHEN the Scheduler triggers a run, THE Refactor_Agent SHALL check for an active lock file to prevent concurrent runs on the same repository.
5. IF a lock file exists from a previous run, THEN THE Refactor_Agent SHALL skip the current run and log a warning.

### Requirement 5: Automated Fix Generation

**User Story:** As a developer, I want the agent to implement TODO items and update outdated dependencies automatically, so that tech debt is resolved without manual effort.

#### Acceptance Criteria

1. WHEN processing a TODO Tech_Debt_Item, THE Fix_Generator SHALL use the OpenHands AgentController to generate code changes that implement the TODO description.
2. WHEN processing a dependency Tech_Debt_Item, THE Fix_Generator SHALL update the dependency version in the manifest file and any lock files.
3. WHEN generating a fix, THE Fix_Generator SHALL create a dedicated Git branch named `refactor/<item-type>/<short-description>`.
4. IF the Fix_Generator fails to produce a valid patch after the configured maximum iterations, THEN THE Refactor_Agent SHALL skip the item and log the failure.
5. WHEN a fix is generated, THE Fix_Generator SHALL produce a diff patch representing the changes.

### Requirement 6: Fix Verification in Sandbox

**User Story:** As a developer, I want every automated fix to be tested before it is proposed, so that the agent does not introduce regressions.

#### Acceptance Criteria

1. WHEN a fix patch is produced, THE Fix_Verifier SHALL apply the patch inside an OpenHands runtime sandbox with the repository checked out at the current HEAD.
2. WHEN the patch is applied, THE Fix_Verifier SHALL execute the project test suite (detected from Refactor_Config or inferred from project structure).
3. WHEN the test suite passes with a zero exit code, THE Fix_Verifier SHALL mark the fix as verified.
4. IF the test suite fails after applying the patch, THEN THE Fix_Verifier SHALL mark the fix as failed and THE Refactor_Agent SHALL discard the patch.
5. IF the sandbox fails to initialize, THEN THE Refactor_Agent SHALL log the error and skip the current Tech_Debt_Item.

### Requirement 7: Pull Request Creation and Reporting

**User Story:** As a developer, I want the agent to open well-documented pull requests for each fix, so that I can review and merge improvements easily.

#### Acceptance Criteria

1. WHEN a fix is verified, THE PR_Creator SHALL commit the changes to the dedicated branch and push the branch to the remote repository.
2. WHEN creating a pull request, THE PR_Creator SHALL generate a descriptive title summarizing the resolved tech debt item.
3. WHEN creating a pull request, THE PR_Creator SHALL include in the PR body: the original TODO text or dependency version info, the changes made, and the test results.
4. WHEN creating a pull request, THE PR_Creator SHALL apply a configurable label (default: `tech-debt`) to the pull request.
5. WHERE Refactor_Config specifies reviewers, THE PR_Creator SHALL assign the configured reviewers to the pull request.
6. WHEN all items in a run are processed, THE Refactor_Agent SHALL produce a summary report listing all attempted items, their outcomes (fixed, skipped, failed), and links to any opened PRs.

### Requirement 8: Configuration Management

**User Story:** As a developer, I want to configure the agent's behavior per repository, so that scanning scope, scheduling, and policies match my project's needs.

#### Acceptance Criteria

1. THE Refactor_Agent SHALL read configuration from a `.refactor.yml` file in the repository root.
2. WHEN no `.refactor.yml` file exists, THE Refactor_Agent SHALL use sensible defaults for all configuration values.
3. THE Refactor_Config SHALL support the following settings: `schedule`, `exclude_paths`, `include_paths`, `max_items_per_run`, `allowed_item_types`, `test_command`, `reviewers`, `labels`, and `max_iterations`.
4. WHEN a `.refactor.yml` file contains invalid YAML, THE Refactor_Agent SHALL log a warning and fall back to default configuration.
5. THE Refactor_Config SHALL support serialization to and deserialization from YAML.
6. FOR ALL valid Refactor_Config objects, serializing to YAML then deserializing SHALL produce an equivalent Refactor_Config object (round-trip property).
