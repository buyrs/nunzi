# Implementation Plan: Swarm Debugging (Multi-Agent Collaboration)

## Overview

This plan implements the swarm debugging feature incrementally: core data models and enums first, then the shared channel, swarm controller, manager agent, API endpoints, and finally the frontend. Tests are woven in close to each implementation step.

## Tasks

- [ ] 1. Define core data models, enums, and event types
  - [ ] 1.1 Create `openhands/controller/swarm_types.py` with `SwarmSessionStatus`, `SwarmEventType`, `SwarmConfig`, `SubAgentState`, `Correlation`, and `SwarmEvent` dataclasses
    - Define all enums and dataclasses as specified in the design Data Models section
    - _Requirements: 3.3, 4.1_
  - [ ] 1.2 Create `ResolutionSummary` dataclass with `serialize()` and `deserialize()` methods in `openhands/controller/swarm_types.py`
    - Implement JSON-compatible serialization and deserialization
    - _Requirements: 4.4, 4.5_
  - [ ] 1.3 Write property test for ResolutionSummary serialization round-trip
    - **Property 9: Resolution summary serialization round-trip**
    - **Validates: Requirements 4.4, 4.5**
  - [ ] 1.4 Create `openhands/events/action/swarm.py` with `SwarmSpawnAction` and `SwarmMessageAction`
    - Add new `ActionType` entries: `SWARM_SPAWN`, `SWARM_MESSAGE`
    - Register in `openhands/events/action/__init__.py`
    - _Requirements: 7.1_
  - [ ] 1.5 Create `openhands/events/observation/swarm.py` with `SwarmEventObservation` and `SwarmResolutionObservation`
    - Add new `ObservationType` entries: `SWARM_EVENT`, `SWARM_RESOLUTION`
    - Register in `openhands/events/observation/__init__.py`
    - _Requirements: 3.2_
  - [ ] 1.6 Update event serialization/deserialization in `openhands/events/serialization/` to handle new swarm action and observation types
    - _Requirements: 7.2_

- [ ] 2. Implement AgentRole registry
  - [ ] 2.1 Create `openhands/controller/swarm_roles.py` with `AgentRole` dataclass and `AgentRoleRegistry` class
    - Implement `register()`, `get()`, and `list_roles()` class methods
    - _Requirements: 2.5_
  - [ ] 2.2 Register default roles (frontend, backend, database) with their tool configurations
    - Frontend role: BrowsingAgent with browse_url, browse_interactive, network_capture
    - Backend role: CodeActAgent with cmd_run, file_read, ipython
    - Database role: CodeActAgent with cmd_run, ipython
    - _Requirements: 2.1, 2.2, 2.3, 2.4_
  - [ ] 2.3 Write property test for role registration preserving existing roles
    - **Property 4: Role registration preserves existing roles**
    - **Validates: Requirements 2.5**
  - [ ] 2.4 Write unit tests for default role configurations
    - Verify three default roles exist with correct tools
    - _Requirements: 2.1, 2.2, 2.3, 2.4_

- [ ] 3. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement SharedChannel
  - [ ] 4.1 Create `openhands/controller/swarm_channel.py` with `SharedChannel` class
    - Implement `post()`, `get_events()`, and `get_events_by_role()` methods
    - Use EventStream with swarm_session_id namespace filtering
    - _Requirements: 3.1, 3.2, 3.3_
  - [ ] 4.2 Write property test for event broadcast visibility
    - **Property 6: Event broadcast visibility**
    - **Validates: Requirements 3.2**
  - [ ] 4.3 Write property test for event structure completeness
    - **Property 7: Event structure completeness**
    - **Validates: Requirements 3.3**
  - [ ] 4.4 Write property test for channel creation per session
    - **Property 5: Channel creation per session**
    - **Validates: Requirements 3.1**

- [ ] 5. Implement SwarmController
  - [ ] 5.1 Create `openhands/controller/swarm_controller.py` with `SwarmController` class
    - Implement `spawn_sub_agent()` using `AgentDelegateAction` and existing `AgentController`
    - Implement `terminate_sub_agent()`, `check_progress()`, `close()`
    - _Requirements: 1.2, 1.5, 7.1, 7.3_
  - [ ] 5.2 Implement `produce_resolution()` method that aggregates findings into a `ResolutionSummary`
    - Collect findings by role, build timeline, include sub-agent states
    - _Requirements: 4.1, 4.2, 4.3_
  - [ ] 5.3 Implement sub-agent failure handling in `check_progress()`
    - Detect crashed/unresponsive agents, mark as failed, continue session
    - Handle budget/iteration limit exceeded via `ControlFlag.reached_limit()`
    - _Requirements: 6.1, 6.2, 6.5_
  - [ ] 5.4 Write property test for spawn count matching roles
    - **Property 2: Spawn count matches determined roles**
    - **Validates: Requirements 1.2**
  - [ ] 5.5 Write property test for session cleanup
    - **Property 3: Session cleanup leaves no residual state**
    - **Validates: Requirements 1.5**
  - [ ] 5.6 Write property test for sub-agent failure not halting session
    - **Property 12: Sub-agent failure does not halt session**
    - **Validates: Requirements 6.1**
  - [ ] 5.7 Write property test for failure details in resolution
    - **Property 13: Failure details included in resolution**
    - **Validates: Requirements 6.2**
  - [ ] 5.8 Write property test for budget-exceeded termination
    - **Property 14: Budget-exceeded agent is terminated and scope redistributed**
    - **Validates: Requirements 6.5**
  - [ ] 5.9 Write property test for resolution summary completeness
    - **Property 8: Resolution summary completeness**
    - **Validates: Requirements 4.1, 4.2, 4.3**
  - [ ] 5.10 Write unit tests for edge cases
    - All sub-agents failing produces partial resolution (Req 6.3)
    - Manager failure preserves events (Req 6.4)
    - _Requirements: 6.3, 6.4_

- [ ] 6. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Implement SwarmManagerAgent
  - [ ] 7.1 Create `openhands/agenthub/swarm_manager_agent/` directory with `__init__.py` and `swarm_manager_agent.py`
    - Extend `Agent` base class
    - Implement `step()` orchestration loop: analyze → spawn → monitor → resolve
    - _Requirements: 1.1, 1.3, 1.4_
  - [ ] 7.2 Implement `analyze_bug_report()` method using LLM to determine required roles
    - _Requirements: 1.1_
  - [ ] 7.3 Implement `correlate_findings()` and `synthesize_resolution()` methods
    - _Requirements: 3.5, 4.1, 4.2, 4.3_
  - [ ] 7.4 Register SwarmManagerAgent in `openhands/agenthub/__init__.py`
    - _Requirements: 7.1_
  - [ ] 7.5 Write property test for role determination validity
    - **Property 1: Role determination returns valid roles**
    - **Validates: Requirements 1.1**
  - [ ] 7.6 Write unit test for non-swarm behavior unchanged
    - Verify standard single-agent flow is unaffected when swarm is not triggered
    - _Requirements: 7.5_

- [ ] 8. Implement backend API endpoints
  - [ ] 8.1 Create `openhands/server/routes/swarm.py` with REST endpoints
    - GET `/api/swarm/{session_id}/status` — returns swarm session status
    - GET `/api/swarm/{session_id}/events` — returns swarm events with `?since_id=` support
    - POST `/api/swarm/{session_id}/message` — posts user message to shared channel
    - GET `/api/swarm/{session_id}/resolution` — returns resolution summary
    - _Requirements: 5.2, 5.5_
  - [ ] 8.2 Register swarm routes in the application server
    - _Requirements: 7.2_
  - [ ] 8.3 Write unit tests for API endpoints
    - Test status, events, message posting, and resolution retrieval
    - _Requirements: 5.2, 5.5_

- [ ] 9. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Implement frontend swarm components
  - [ ] 10.1 Add swarm TypeScript types in `frontend/src/types/core/swarm.ts`
    - Define `SwarmEvent`, `SubAgentState`, `ResolutionSummary`, `SwarmSessionStatus` interfaces
    - _Requirements: 3.3, 4.1_
  - [ ] 10.2 Create TanStack Query hooks for swarm data
    - `frontend/src/hooks/query/use-swarm-session.ts` — fetch session status
    - `frontend/src/hooks/query/use-swarm-events.ts` — poll swarm events
    - `frontend/src/hooks/mutation/use-send-swarm-message.ts` — post user message
    - _Requirements: 5.2, 5.5_
  - [ ] 10.3 Create `frontend/src/components/features/swarm/swarm-panel.tsx`
    - Render active sub-agents with role labels and status indicators
    - Render shared event timeline with role labels and timestamps
    - Render resolution summary when available
    - Include user message input
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_
  - [ ] 10.4 Integrate SwarmPanel into the main chat/conversation view
    - Show SwarmPanel when a swarm session is active, hide otherwise
    - _Requirements: 5.1_
  - [ ] 10.5 Write property test for sub-agent panel rendering
    - **Property 10: Sub-agent panel renders all agents**
    - **Validates: Requirements 5.1**
  - [ ] 10.6 Write property test for event timeline rendering
    - **Property 11: Event timeline renders role and timestamp**
    - **Validates: Requirements 5.2**
  - [ ] 10.7 Write unit tests for frontend components
    - Status indicator transitions (Req 5.3)
    - Resolution summary display (Req 5.4)
    - User message input availability (Req 5.5)
    - _Requirements: 5.3, 5.4, 5.5_

- [ ] 11. Wire everything together
  - [ ] 11.1 Integrate SwarmController into the main session flow
    - When SwarmManagerAgent is selected or triggered, create SwarmController
    - Connect SwarmController to the existing conversation session
    - _Requirements: 7.3, 7.5_
  - [ ] 11.2 Add swarm event types to frontend event handling
    - Update `frontend/src/types/core/base.ts` with swarm action/observation types
    - Update event rendering to handle swarm events in the chat view
    - _Requirements: 5.2_
  - [ ] 11.3 Write integration tests for end-to-end swarm flow
    - Test: user message → manager analysis → sub-agent spawn → findings → resolution
    - _Requirements: 1.1, 1.2, 1.4, 3.2, 4.1_

- [ ] 12. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties using `hypothesis` (Python) and `fast-check` (TypeScript)
- Unit tests validate specific examples and edge cases
- The implementation builds on existing `AgentController`, `AgentDelegateAction`, and `EventStream` infrastructure
