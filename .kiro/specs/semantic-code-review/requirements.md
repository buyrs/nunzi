# Requirements Document

## Introduction

Gatekeeper is a semantic code review agent for the Nunzi platform (built on OpenHands). When a pull request is opened or updated, Gatekeeper analyzes the changed code against the team's coding standards (loaded from a Cortex knowledge base), identifies style violations, security vulnerabilities, and complexity issues, and posts inline review comments on the PR before a human reviewer sees it. Gatekeeper integrates into existing GitHub workflows and leverages the OpenHands runtime, agent controller, and GitHub PR interfaces.

## Glossary

- **Gatekeeper_Agent**: The autonomous code review agent that orchestrates the full PR analysis, rule evaluation, and comment posting cycle.
- **PR_Diff_Fetcher**: The component responsible for retrieving the diff and changed file contents from a pull request using the GitHub API.
- **Cortex_Loader**: The component that loads team-specific coding standards, style rules, and security patterns from a Cortex knowledge base or a repository-level `.gatekeeper.yml` configuration file.
- **Rule_Set**: A structured collection of coding rules organized by category (style, security, complexity) that Gatekeeper evaluates code against.
- **Rule**: A single coding standard with a category, severity level, description, and pattern or heuristic for detection.
- **Code_Analyzer**: The component that uses the OpenHands agent controller to analyze changed code against the loaded Rule_Set and produce structured findings.
- **Finding**: A structured data object describing a single code issue, including the file path, line number, rule violated, severity, and a human-readable explanation with a suggested fix.
- **Finding_Set**: A collection of Finding objects produced by the Code_Analyzer for a single PR review.
- **Deduplicator**: The component that removes duplicate or overlapping findings and filters out findings on unchanged lines.
- **Comment_Poster**: The component that posts inline review comments on the pull request using the GitHub review API.
- **Gatekeeper_Config**: A repository-level configuration file (`.gatekeeper.yml`) that controls Gatekeeper behavior, including enabled rule categories, severity thresholds, ignored paths, and custom rules.
- **Gatekeeper_Workflow**: The GitHub Actions workflow YAML that triggers Gatekeeper on PR events.
- **Review_Summary**: A structured summary posted as a top-level PR comment describing the overall review outcome.

## Requirements

### Requirement 1: PR Event Detection and Triggering

**User Story:** As a developer, I want Gatekeeper to automatically review my PR when I open or update it, so that I get feedback before requesting human review.

#### Acceptance Criteria

1. WHEN a pull request is opened against the default branch, THE Gatekeeper_Workflow SHALL trigger the Gatekeeper_Agent with the PR number and repository context.
2. WHEN a pull request is synchronized (new commits pushed) against the default branch, THE Gatekeeper_Workflow SHALL trigger the Gatekeeper_Agent with the updated PR number and repository context.
3. WHEN the Gatekeeper_Agent is triggered, THE PR_Diff_Fetcher SHALL retrieve the full diff and list of changed files from the pull request using the GitHub API.
4. IF the GitHub API is unreachable or returns an error, THEN THE Gatekeeper_Agent SHALL log the error and terminate the review attempt gracefully.
5. WHEN the Gatekeeper_Agent is triggered, THE Gatekeeper_Agent SHALL check for a `.gatekeeper.yml` configuration file in the repository root and load it if present.
6. IF no `.gatekeeper.yml` file exists, THEN THE Gatekeeper_Agent SHALL use default configuration values and proceed with the review.

### Requirement 2: Coding Standards Loading

**User Story:** As a team lead, I want Gatekeeper to learn my team's coding standards from our Cortex knowledge base, so that reviews reflect our specific conventions.

#### Acceptance Criteria

1. WHEN the Gatekeeper_Agent starts processing, THE Cortex_Loader SHALL load the Rule_Set from the configured Cortex knowledge base identifier.
2. IF the Cortex knowledge base is unavailable, THEN THE Cortex_Loader SHALL fall back to built-in default rules and log a warning.
3. WHEN a `.gatekeeper.yml` file contains custom rules, THE Cortex_Loader SHALL merge custom rules with the loaded Rule_Set, with custom rules taking precedence on conflicts.
4. THE Cortex_Loader SHALL categorize each Rule in the Rule_Set as one of: style, security, or complexity.
5. WHEN a Rule_Set is loaded, THE Cortex_Loader SHALL serialize the Rule_Set to JSON for storage and downstream consumption.
6. FOR ALL valid Rule_Set objects, serializing then deserializing SHALL produce an equivalent Rule_Set object (round-trip property).

### Requirement 3: Code Analysis

**User Story:** As a developer, I want Gatekeeper to analyze my code changes against team standards, so that I can fix issues before human review.

#### Acceptance Criteria

1. WHEN a PR diff is retrieved, THE Code_Analyzer SHALL analyze each changed file against the active Rule_Set and produce a Finding_Set.
2. WHEN analyzing code, THE Code_Analyzer SHALL evaluate style rules by comparing code patterns against the Rule_Set style entries.
3. WHEN analyzing code, THE Code_Analyzer SHALL evaluate security rules by checking for known vulnerability patterns (SQL injection, hardcoded credentials, unsafe deserialization, command injection).
4. WHEN analyzing code, THE Code_Analyzer SHALL evaluate complexity rules by measuring function length, cyclomatic complexity indicators, and nesting depth.
5. WHEN a Finding is produced, THE Code_Analyzer SHALL include the file path, line number, violated rule identifier, severity level, a human-readable explanation, and a suggested fix.
6. WHEN a Finding_Set is produced, THE Code_Analyzer SHALL serialize the Finding_Set to JSON for storage and downstream consumption.
7. FOR ALL valid Finding_Set objects, serializing then deserializing SHALL produce an equivalent Finding_Set object (round-trip property).
8. IF a changed file is in a language not supported by the Code_Analyzer, THEN THE Code_Analyzer SHALL skip that file and log a notice.

### Requirement 4: Finding Deduplication and Filtering

**User Story:** As a developer, I want Gatekeeper to post only relevant, non-redundant comments, so that the review is useful and not noisy.

#### Acceptance Criteria

1. WHEN a Finding_Set is produced, THE Deduplicator SHALL remove findings that reference the same file, line, and rule.
2. WHEN a Finding_Set is produced, THE Deduplicator SHALL remove findings on lines that were not changed in the PR diff.
3. WHEN the Gatekeeper_Config specifies a minimum severity threshold, THE Deduplicator SHALL remove findings below that threshold.
4. WHEN the Gatekeeper_Config specifies ignored file paths, THE Deduplicator SHALL remove findings for files matching those patterns.
5. WHEN deduplication is complete, THE Deduplicator SHALL produce a filtered Finding_Set containing only actionable findings.

### Requirement 5: Review Comment Posting

**User Story:** As a developer, I want Gatekeeper to post inline comments on my PR with clear explanations and suggestions, so that I can quickly understand and fix issues.

#### Acceptance Criteria

1. WHEN a filtered Finding_Set contains one or more findings, THE Comment_Poster SHALL post an inline review comment on the PR for each finding at the corresponding file and line.
2. WHEN posting an inline comment, THE Comment_Poster SHALL format the comment with the rule category, severity, explanation, and suggested fix.
3. WHEN all inline comments are posted, THE Comment_Poster SHALL post a Review_Summary as a top-level PR comment listing the total number of findings by category and severity.
4. IF the filtered Finding_Set is empty, THEN THE Comment_Poster SHALL post a Review_Summary indicating no issues were found.
5. WHEN posting comments, THE Comment_Poster SHALL use the GitHub pull request review API to batch all inline comments into a single review.
6. IF posting a review comment fails, THEN THE Gatekeeper_Agent SHALL log the error and continue with remaining comments.

### Requirement 6: Configuration and Customization

**User Story:** As a repository maintainer, I want to configure Gatekeeper's behavior, so that it matches my project's needs and does not flag false positives.

#### Acceptance Criteria

1. THE Gatekeeper_Config SHALL support an `enabled` field that allows disabling Gatekeeper entirely (default: true).
2. WHEN the `enabled` field is set to false, THE Gatekeeper_Agent SHALL skip all processing and terminate immediately.
3. THE Gatekeeper_Config SHALL support a `cortex_id` field that specifies the Cortex knowledge base identifier to load rules from.
4. THE Gatekeeper_Config SHALL support a `categories` field that specifies which rule categories to enable (default: all of style, security, complexity).
5. THE Gatekeeper_Config SHALL support a `min_severity` field that specifies the minimum severity level for reported findings (default: info).
6. THE Gatekeeper_Config SHALL support an `ignore_paths` field that specifies glob patterns for files to exclude from analysis.
7. THE Gatekeeper_Config SHALL support a `custom_rules` field that allows defining additional rules inline.
8. THE Gatekeeper_Config SHALL support a `max_comments` field that limits the number of inline comments per review (default: 50).
9. WHEN the number of findings exceeds `max_comments`, THE Comment_Poster SHALL post only the highest-severity findings up to the limit and note the truncation in the Review_Summary.

### Requirement 7: Error Handling and Observability

**User Story:** As a developer, I want Gatekeeper to handle errors gracefully and provide clear logs, so that I can understand what happened when something goes wrong.

#### Acceptance Criteria

1. WHEN any component encounters an unrecoverable error, THE Gatekeeper_Agent SHALL log the error with full context (component name, input data summary, error message) and terminate gracefully.
2. WHEN the review attempt completes (success or failure), THE Gatekeeper_Agent SHALL produce a structured Gatekeeper_Output JSON object summarizing the run: PR number reviewed, files analyzed, findings produced, findings after filtering, comments posted, and any errors.
3. WHEN the Gatekeeper_Agent produces a Gatekeeper_Output, THE Gatekeeper_Agent SHALL write the output to the configured output directory as `gatekeeper_output.json`.
4. FOR ALL valid Gatekeeper_Output objects, serializing then deserializing SHALL produce an equivalent Gatekeeper_Output object (round-trip property).
