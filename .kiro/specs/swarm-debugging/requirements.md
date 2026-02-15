# Requirements Document

## Introduction

This feature introduces dynamic agent swarms for coordinated multi-agent debugging in OpenHands. When a user reports a complex bug that spans multiple layers (frontend, backend, database), a Manager Agent orchestrates specialized sub-agents that work concurrently to diagnose and resolve the issue. Sub-agents communicate findings through a shared channel, mimicking a real-world "War Room" debugging session.

## Glossary

- **Manager_Agent**: The orchestrating agent that receives the user's bug report, determines which specialized sub-agents to spawn, and coordinates their work toward a resolution.
- **Sub_Agent**: A specialized agent spawned by the Manager_Agent to investigate a specific layer or domain of the system (e.g., frontend, backend, database).
- **Swarm**: A collection of one Manager_Agent and one or more Sub_Agents working together on a single debugging task.
- **Shared_Channel**: A communication mechanism through which Sub_Agents and the Manager_Agent exchange findings, observations, and conclusions during a debugging session.
- **Swarm_Session**: A single debugging session encompassing the lifecycle of a Swarm from creation to resolution.
- **Agent_Role**: A named specialization assigned to a Sub_Agent that determines its tools, prompts, and investigation strategy (e.g., "frontend", "backend", "database").
- **Swarm_Event**: An event posted to the Shared_Channel by any agent in the Swarm, containing a finding, question, or conclusion.
- **Resolution_Summary**: A final report produced by the Manager_Agent that aggregates findings from all Sub_Agents and presents the root cause and recommended fix.

## Requirements

### Requirement 1: Swarm Session Lifecycle

**User Story:** As a developer, I want the system to create and manage a coordinated debugging session when I report a complex bug, so that multiple specialized agents can investigate the issue concurrently.

#### Acceptance Criteria

1. WHEN a user submits a bug report, THE Manager_Agent SHALL analyze the report and determine which Agent_Roles are needed for investigation.
2. WHEN the Manager_Agent determines the required Agent_Roles, THE Manager_Agent SHALL spawn one Sub_Agent for each identified Agent_Role.
3. WHILE a Swarm_Session is active, THE Manager_Agent SHALL monitor the progress of all Sub_Agents and coordinate their activities.
4. WHEN all Sub_Agents have completed their investigations or a timeout is reached, THE Manager_Agent SHALL produce a Resolution_Summary.
5. WHEN a Swarm_Session ends, THE system SHALL clean up all Sub_Agent resources and return to a single-agent conversation state.

### Requirement 2: Sub-Agent Specialization

**User Story:** As a developer, I want each sub-agent to have domain-specific tools and investigation strategies, so that each layer of the stack is investigated by an expert.

#### Acceptance Criteria

1. THE system SHALL support at least three Agent_Roles: "frontend", "backend", and "database".
2. WHEN a Sub_Agent with the "frontend" Agent_Role is spawned, THE Sub_Agent SHALL have access to browser interaction tools and network log capture capabilities.
3. WHEN a Sub_Agent with the "backend" Agent_Role is spawned, THE Sub_Agent SHALL have access to server log tailing and exception analysis tools.
4. WHEN a Sub_Agent with the "database" Agent_Role is spawned, THE Sub_Agent SHALL have access to query execution and lock inspection tools.
5. WHEN a new Agent_Role is registered, THE system SHALL make the role available for selection by the Manager_Agent without modifying existing roles.

### Requirement 3: Shared Channel Communication

**User Story:** As a developer, I want all agents in a swarm to communicate through a shared channel, so that findings from one agent can inform the investigation of another.

#### Acceptance Criteria

1. WHEN a Swarm_Session is created, THE system SHALL create a Shared_Channel associated with that session.
2. WHEN a Sub_Agent posts a Swarm_Event to the Shared_Channel, THE Shared_Channel SHALL make the event visible to the Manager_Agent and all other Sub_Agents in the same Swarm.
3. WHEN a Swarm_Event is posted, THE Shared_Channel SHALL include the originating Agent_Role, a timestamp, and the event content.
4. WHILE a Swarm_Session is active, THE Manager_Agent SHALL read Swarm_Events from the Shared_Channel and use them to direct Sub_Agent activities.
5. WHEN a Swarm_Event contains a finding that contradicts or correlates with another agent's finding, THE Manager_Agent SHALL flag the correlation and notify relevant Sub_Agents.

### Requirement 4: Resolution and Reporting

**User Story:** As a developer, I want a consolidated summary of the debugging session, so that I can understand the root cause and apply the fix.

#### Acceptance Criteria

1. WHEN the Manager_Agent produces a Resolution_Summary, THE Resolution_Summary SHALL include findings from each Sub_Agent organized by Agent_Role.
2. WHEN the Manager_Agent produces a Resolution_Summary, THE Resolution_Summary SHALL identify the root cause and recommend a fix.
3. WHEN the Manager_Agent produces a Resolution_Summary, THE Resolution_Summary SHALL include a timeline of Swarm_Events that led to the diagnosis.
4. THE system SHALL serialize the Resolution_Summary to a structured format for storage and retrieval.
5. THE system SHALL deserialize stored Resolution_Summaries back into their original structured form.

### Requirement 5: Frontend Visualization

**User Story:** As a developer, I want to see the swarm debugging session in the UI, so that I can follow the investigation in real time and understand what each agent is doing.

#### Acceptance Criteria

1. WHEN a Swarm_Session is active, THE frontend SHALL display a panel showing all active Sub_Agents and their current Agent_Roles.
2. WHEN a Swarm_Event is posted to the Shared_Channel, THE frontend SHALL render the event in a shared timeline view with the originating Agent_Role label and timestamp.
3. WHEN a Sub_Agent completes its investigation, THE frontend SHALL update the agent's status indicator from "investigating" to "complete".
4. WHEN the Resolution_Summary is produced, THE frontend SHALL display the summary in a structured, readable format.
5. WHILE a Swarm_Session is active, THE frontend SHALL allow the user to send a message to the Shared_Channel that all agents can read.

### Requirement 6: Error Handling and Fault Tolerance

**User Story:** As a developer, I want the swarm to handle agent failures gracefully, so that a single agent crash does not derail the entire debugging session.

#### Acceptance Criteria

1. IF a Sub_Agent crashes or becomes unresponsive, THEN THE Manager_Agent SHALL mark the Sub_Agent as failed and continue the session with the remaining Sub_Agents.
2. IF a Sub_Agent fails, THEN THE Manager_Agent SHALL log the failure reason and include the failure in the Resolution_Summary.
3. IF all Sub_Agents fail, THEN THE Manager_Agent SHALL produce a partial Resolution_Summary with whatever findings were collected and notify the user.
4. IF the Manager_Agent itself encounters an error, THEN THE system SHALL preserve all Swarm_Events collected so far and present them to the user.
5. IF a Sub_Agent exceeds its iteration or budget limit, THEN THE Manager_Agent SHALL terminate that Sub_Agent and redistribute its remaining investigation scope to other active Sub_Agents.

### Requirement 7: Integration with Existing Agent Infrastructure

**User Story:** As a developer, I want the swarm debugging feature to integrate with the existing OpenHands agent controller and event system, so that it builds on proven infrastructure rather than duplicating it.

#### Acceptance Criteria

1. THE Swarm SHALL use the existing AgentDelegateAction mechanism to spawn Sub_Agents.
2. THE Swarm SHALL use the existing EventStream to transport Swarm_Events between agents.
3. WHEN a Swarm_Session is created, THE system SHALL reuse the existing State and AgentController infrastructure for each Sub_Agent.
4. THE Shared_Channel SHALL be implemented as a namespaced partition within the existing EventStream rather than a separate communication system.
5. WHEN the Swarm feature is disabled or not triggered, THE system SHALL operate identically to the current single-agent behavior.
