# Requirements Document

## Introduction

Scribe is an autonomous documentation sync agent for the Nunzi platform (built on OpenHands). When a pull request is merged, Scribe analyzes the code changes, identifies documentation that has drifted out of sync, and automatically generates updates to affected documentation files (README.md, OpenAPI specs, Mermaid diagrams, inline doc comments). It then opens a new pull request with the documentation updates. Scribe integrates into existing GitHub workflows and leverages the OpenHands runtime, agent controller, and `send_pull_request` infrastructure.

## Glossary

- **Scribe_Agent**: The autonomous documentation sync agent that orchestrates the full PR analysis, change detection, documentation update, and PR creation cycle.
- **PR_Change_Analyzer**: The component responsible for extracting structured change summaries from a merged pull request's diff.
- **Change_Summary**: A structured data object describing what changed in a merged PR, including affected files, added/removed/modified functions, endpoints, classes, and configuration entries.
- **Doc_Inventory**: A catalog of documentation files in the repository and their associated content types (API spec, README, architecture diagram, inline docs).
- **Doc_Drift_Detector**: The component that compares a Change_Summary against the Doc_Inventory to identify documentation files that are potentially out of sync.
- **Drift_Report**: A structured data object listing documentation files that need updating, the reason each file is stale, and the specific sections affected.
- **Doc_Updater**: The component that uses the OpenHands agent controller to generate updated documentation content based on the Drift_Report and source code context.
- **Update_Verifier**: The component that validates generated documentation updates for structural correctness (valid JSON for OpenAPI, valid Mermaid syntax, valid Markdown).
- **PR_Creator**: The component that commits documentation updates and opens a pull request with a descriptive title and body.
- **Scribe_Workflow**: The GitHub Actions workflow YAML that triggers Scribe on PR merge events.
- **Scribe_Config**: A repository-level configuration file (`.scribe.yml`) that controls Scribe behavior, including watched paths, ignored paths, and documentation mappings.

## Requirements

### Requirement 1: PR Merge Detection and Triggering

**User Story:** As a developer, I want Scribe to automatically wake up when a PR is merged, so that documentation sync begins without manual intervention.

#### Acceptance Criteria

1. WHEN a pull request is merged into the default branch, THE Scribe_Workflow SHALL trigger the Scribe_Agent with the merged PR number and repository context.
2. WHEN the Scribe_Workflow is triggered, THE Scribe_Agent SHALL retrieve the full diff of the merged pull request using the GitHub API.
3. IF the GitHub API is unreachable or returns an error, THEN THE Scribe_Agent SHALL log the error and terminate the sync attempt gracefully.
4. WHEN the Scribe_Workflow is triggered, THE Scribe_Agent SHALL check for a `.scribe.yml` configuration file in the repository root and load it if present.
5. IF no `.scribe.yml` file exists, THEN THE Scribe_Agent SHALL use default configuration values and proceed with the sync attempt.

### Requirement 2: Code Change Analysis

**User Story:** As a developer, I want Scribe to understand what changed in my PR, so that it can determine which documentation needs updating.

#### Acceptance Criteria

1. WHEN a PR diff is retrieved, THE PR_Change_Analyzer SHALL parse the diff and produce a Change_Summary containing affected file paths, added functions, removed functions, modified function signatures, new API endpoints, modified API endpoints, removed API endpoints, and changed configuration entries.
2. WHEN producing a Change_Summary, THE PR_Change_Analyzer SHALL categorize each change as one of: endpoint_change, function_signature_change, class_change, config_change, dependency_change, or other.
3. WHEN a Change_Summary is produced, THE PR_Change_Analyzer SHALL serialize the Change_Summary to JSON for storage and downstream consumption.
4. FOR ALL valid Change_Summary objects, serializing then deserializing SHALL produce an equivalent Change_Summary object (round-trip property).
5. IF the PR diff contains only non-code files (e.g., images, binary assets), THEN THE Scribe_Agent SHALL skip the sync attempt and log that no documentation-relevant changes were detected.

### Requirement 3: Documentation Inventory and Drift Detection

**User Story:** As a developer, I want Scribe to know which documentation files exist and detect when they are out of sync, so that only relevant docs are updated.

#### Acceptance Criteria

1. WHEN the Scribe_Agent starts processing, THE Doc_Inventory SHALL scan the repository for documentation files matching configured patterns (default: `README.md`, `**/README.md`, `docs/**/*.md`, `**/openapi.json`, `**/openapi.yaml`, `**/swagger.json`).
2. WHEN the Doc_Inventory is built, THE Doc_Drift_Detector SHALL compare each documentation file against the Change_Summary to determine if the file references any changed code elements.
3. WHEN drift is detected, THE Doc_Drift_Detector SHALL produce a Drift_Report listing each affected documentation file, the specific sections that are stale, and the reason for staleness.
4. WHEN a Drift_Report is produced, THE Doc_Drift_Detector SHALL serialize the Drift_Report to JSON for storage and downstream consumption.
5. FOR ALL valid Drift_Report objects, serializing then deserializing SHALL produce an equivalent Drift_Report object (round-trip property).
6. IF no documentation drift is detected, THEN THE Scribe_Agent SHALL terminate the sync attempt and log that all documentation is up to date.
7. WHERE the Scribe_Config specifies custom documentation path patterns, THE Doc_Inventory SHALL use those patterns instead of the defaults.

### Requirement 4: Documentation Update Generation

**User Story:** As a developer, I want Scribe to generate accurate documentation updates, so that my docs stay in sync with the code.

#### Acceptance Criteria

1. WHEN a Drift_Report is produced with one or more stale documentation files, THE Doc_Updater SHALL use the OpenHands agent controller to generate updated content for each stale file.
2. WHEN generating updates, THE Doc_Updater SHALL provide the agent with the Drift_Report, the current documentation file content, the relevant source code files, and the Change_Summary as context.
3. WHEN updating an OpenAPI specification file, THE Doc_Updater SHALL produce valid OpenAPI 3.x JSON or YAML that reflects the new, modified, or removed endpoints.
4. WHEN updating a Markdown file containing Mermaid diagrams, THE Doc_Updater SHALL produce valid Mermaid syntax within fenced code blocks.
5. WHEN updating a README file, THE Doc_Updater SHALL preserve the existing document structure and only modify sections relevant to the detected changes.
6. IF the agent fails to produce an update within the configured maximum iterations, THEN THE Scribe_Agent SHALL skip that documentation file and continue with remaining files.
7. WHEN generating updates, THE Doc_Updater SHALL constrain the agent to modify only documentation files listed in the Drift_Report.

### Requirement 5: Update Verification

**User Story:** As a developer, I want Scribe to verify that generated documentation updates are structurally valid, so that broken docs are not committed.

#### Acceptance Criteria

1. WHEN an OpenAPI specification update is generated, THE Update_Verifier SHALL validate the output against the OpenAPI 3.x schema and reject invalid output.
2. WHEN a Markdown update containing Mermaid diagrams is generated, THE Update_Verifier SHALL validate that all Mermaid code blocks contain parseable Mermaid syntax.
3. WHEN a Markdown update is generated, THE Update_Verifier SHALL validate that the output is well-formed Markdown with no broken links to headings within the same file.
4. IF a documentation update fails verification, THEN THE Scribe_Agent SHALL discard that update, log the validation errors, and continue with remaining files.
5. WHEN all updates for a Drift_Report have been verified, THE Update_Verifier SHALL produce a final list of verified updates ready for commit.

### Requirement 6: PR Creation and Reporting

**User Story:** As a developer, I want Scribe to open a clean PR with the documentation updates, so that I can review and merge them easily.

#### Acceptance Criteria

1. WHEN verified documentation updates are ready, THE PR_Creator SHALL create a new branch named `docs/scribe-sync-{pr_number}` from the default branch.
2. WHEN the branch is created, THE PR_Creator SHALL commit all verified documentation updates with a commit message following the format: `docs: sync documentation with PR #{pr_number}`.
3. WHEN the commit is pushed, THE PR_Creator SHALL open a pull request with the title `docs: sync documentation with recent changes (#{pr_number})`.
4. WHEN the pull request is opened, THE PR_Creator SHALL include a body that lists each updated file, a summary of what changed, and a reference to the original merged PR.
5. IF no documentation updates pass verification, THEN THE Scribe_Agent SHALL skip PR creation and log that no valid updates were produced.
6. WHEN the PR is created, THE PR_Creator SHALL apply the label `documentation` to the pull request if the label exists in the repository.

### Requirement 7: Configuration and Customization

**User Story:** As a repository maintainer, I want to configure Scribe's behavior, so that it works correctly for my project's documentation structure.

#### Acceptance Criteria

1. THE Scribe_Config SHALL support a `doc_paths` field that specifies glob patterns for documentation files to monitor.
2. THE Scribe_Config SHALL support an `ignore_paths` field that specifies glob patterns for files to exclude from change analysis.
3. THE Scribe_Config SHALL support a `max_iterations` field that controls the maximum number of agent iterations for generating updates (default: 30).
4. THE Scribe_Config SHALL support a `doc_mappings` field that maps source code path patterns to their associated documentation files.
5. WHEN the Scribe_Config contains a `doc_mappings` entry, THE Doc_Drift_Detector SHALL use the mapping to prioritize drift detection for the specified documentation files.
6. THE Scribe_Config SHALL support an `enabled` field that allows disabling Scribe entirely (default: true).
7. WHEN the `enabled` field is set to false, THE Scribe_Agent SHALL skip all processing and terminate immediately.

### Requirement 8: Error Handling and Observability

**User Story:** As a developer, I want Scribe to handle errors gracefully and provide clear logs, so that I can understand what happened when something goes wrong.

#### Acceptance Criteria

1. WHEN any component encounters an unrecoverable error, THE Scribe_Agent SHALL log the error with full context (component name, input data summary, error message) and terminate gracefully.
2. WHEN the sync attempt completes (success or failure), THE Scribe_Agent SHALL produce a structured Scribe_Output JSON object summarizing the run: PR number processed, files analyzed, drift detected, updates generated, updates verified, PR created (or reason for skipping).
3. WHEN the Scribe_Agent produces a Scribe_Output, THE Scribe_Agent SHALL write the output to the configured output directory as `scribe_output.json`.
4. FOR ALL valid Scribe_Output objects, serializing then deserializing SHALL produce an equivalent Scribe_Output object (round-trip property).
