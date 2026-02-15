# Requirements Document

## Introduction

Nexus is an intelligent task routing layer for the Nunzi platform (built on OpenHands) that analyzes incoming user tasks and routes them to the optimal agent execution strategy. Instead of using a one-size-fits-all heavyweight approach for every task, Nexus classifies tasks by complexity and selects the appropriate model, iteration budget, and agent configuration. This reduces cost, improves latency for simple tasks, and allocates powerful resources only when needed.

## Glossary

- **Nexus_Router**: The core routing component that receives a user task, classifies it, and selects the optimal agent configuration.
- **Task_Classifier**: The sub-component of Nexus_Router responsible for analyzing task text and context to determine complexity tier.
- **Complexity_Tier**: One of four categories (Quick_Fix, Standard, Complex, Epic) representing the estimated scope and difficulty of a task.
- **Quick_Fix**: A complexity tier for trivial edits such as typo fixes, variable renames, or single-line changes.
- **Standard**: A complexity tier for moderate tasks such as single-file bug fixes, adding a small function, or updating configuration.
- **Complex**: A complexity tier for multi-file refactors, implementing new features, or significant logic changes.
- **Epic**: A complexity tier for architecture changes, large-scale migrations, or tasks requiring multi-agent orchestration.
- **Agent_Configuration**: A bundle of settings including LLM model, max iterations, condenser config, and agent parameters that define how a task is executed.
- **Routing_Decision**: The output of Nexus_Router containing the selected Complexity_Tier and corresponding Agent_Configuration.
- **Project_Context**: Metadata about the user's project including programming language, framework, and codebase size.
- **Feedback_Record**: A record of a completed task's routing decision, actual outcome, cost, and duration used to improve future routing accuracy.
- **Routing_Confidence**: A numeric score (0.0 to 1.0) representing the Task_Classifier's confidence in its classification.

## Requirements

### Requirement 1: Task Classification

**User Story:** As a user, I want my submitted tasks to be automatically classified by complexity, so that the system can allocate the right resources without manual intervention.

#### Acceptance Criteria

1. WHEN a user submits a task, THE Task_Classifier SHALL analyze the task description text and produce a Complexity_Tier classification within 2 seconds.
2. WHEN the Task_Classifier produces a classification, THE Task_Classifier SHALL also produce a Routing_Confidence score between 0.0 and 1.0.
3. WHEN a task description contains keywords indicating trivial changes (e.g., "rename", "typo", "fix spelling"), THE Task_Classifier SHALL classify the task as Quick_Fix.
4. WHEN a task description references multiple files or modules, THE Task_Classifier SHALL classify the task as Complex or Epic.
5. WHEN a task description references architecture, migration, or large-scale restructuring, THE Task_Classifier SHALL classify the task as Epic.
6. IF the Task_Classifier cannot determine a Complexity_Tier with Routing_Confidence above 0.5, THEN THE Task_Classifier SHALL default to the Standard tier.

### Requirement 2: Agent Configuration Selection

**User Story:** As a platform operator, I want each complexity tier to map to an optimized agent configuration, so that simple tasks use fewer resources and complex tasks get the power they need.

#### Acceptance Criteria

1. THE Nexus_Router SHALL maintain a mapping from each Complexity_Tier to a default Agent_Configuration.
2. WHEN a task is classified as Quick_Fix, THE Nexus_Router SHALL select an Agent_Configuration with a lightweight LLM model and a low max iteration limit.
3. WHEN a task is classified as Standard, THE Nexus_Router SHALL select an Agent_Configuration with a standard LLM model and a moderate max iteration limit.
4. WHEN a task is classified as Complex, THE Nexus_Router SHALL select an Agent_Configuration with a powerful LLM model and a high max iteration limit.
5. WHEN a task is classified as Epic, THE Nexus_Router SHALL select an Agent_Configuration with the most capable LLM model, the highest max iteration limit, and multi-agent delegation enabled.
6. THE Nexus_Router SHALL serialize and deserialize Agent_Configuration mappings to and from a JSON configuration format.

### Requirement 3: Project Context Integration

**User Story:** As a user, I want the routing to consider my project's language and framework, so that the agent configuration is fine-tuned for my specific codebase.

#### Acceptance Criteria

1. WHEN a task is submitted with an associated repository, THE Nexus_Router SHALL extract Project_Context including programming language, framework, and estimated codebase size.
2. WHEN Project_Context indicates a large codebase (over 100,000 lines), THE Nexus_Router SHALL increase the Complexity_Tier by one level (e.g., Standard becomes Complex), unless the tier is already Epic.
3. WHEN Project_Context is unavailable, THE Nexus_Router SHALL proceed with classification based solely on the task description.

### Requirement 4: Routing Decision Output

**User Story:** As a developer integrating Nexus, I want a structured routing decision output, so that downstream components can consume it reliably.

#### Acceptance Criteria

1. THE Nexus_Router SHALL produce a Routing_Decision containing the Complexity_Tier, Routing_Confidence, selected Agent_Configuration, and a human-readable explanation.
2. THE Nexus_Router SHALL serialize Routing_Decision objects to JSON format.
3. THE Nexus_Router SHALL deserialize JSON strings back into valid Routing_Decision objects (round-trip property).
4. WHEN a Routing_Decision is produced, THE Nexus_Router SHALL log the decision including task identifier, selected tier, confidence, and selected model name.

### Requirement 5: Feedback Loop

**User Story:** As a platform operator, I want the system to track task outcomes and use them to improve routing accuracy over time, so that classification gets better with usage.

#### Acceptance Criteria

1. WHEN a task completes, THE Nexus_Router SHALL create a Feedback_Record containing the original Routing_Decision, actual cost, actual duration, and a success indicator.
2. THE Nexus_Router SHALL persist Feedback_Record objects to storage.
3. WHEN the Nexus_Router classifies a new task, THE Nexus_Router SHALL consult historical Feedback_Records to adjust Routing_Confidence for similar tasks.
4. THE Nexus_Router SHALL serialize and deserialize Feedback_Record objects to and from JSON format (round-trip property).

### Requirement 6: Integration with Existing Router Infrastructure

**User Story:** As a developer, I want Nexus to integrate with the existing RouterLLM and LLMRegistry infrastructure, so that it works seamlessly within the current OpenHands architecture.

#### Acceptance Criteria

1. THE Nexus_Router SHALL extend the existing RouterLLM base class and register itself in the ROUTER_LLM_REGISTRY.
2. THE Nexus_Router SHALL implement the _select_llm method to route messages to the appropriate LLM based on the current Routing_Decision.
3. WHEN the Nexus_Router is configured via ModelRoutingConfig, THE Nexus_Router SHALL read tier-to-model mappings from the llms_for_routing dictionary.
4. WHEN no Nexus-specific configuration is provided, THE Nexus_Router SHALL fall back to the primary LLM for all tasks (noop behavior).

### Requirement 7: Error Handling and Resilience

**User Story:** As a user, I want the routing layer to handle errors gracefully, so that task execution is never blocked by a routing failure.

#### Acceptance Criteria

1. IF the Task_Classifier encounters an error during classification, THEN THE Nexus_Router SHALL fall back to the Standard tier and log the error.
2. IF a selected LLM model is unavailable, THEN THE Nexus_Router SHALL fall back to the primary LLM and log a warning.
3. IF Feedback_Record storage fails, THEN THE Nexus_Router SHALL continue task execution and log the storage failure.
