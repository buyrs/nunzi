# Implementation Plan: Studio Mode (RunPod AI Lab)

## Overview

Implement the Studio Mode feature incrementally: start with core data models and serialization, then build the orchestration layer, integrate with RunPod runtime, add frontend components, and wire everything together. Each task builds on the previous, with property tests validating correctness at each step.

## Tasks

- [ ] 1. Create Studio module structure and core data models
  - [ ] 1.1 Create `openhands/studio/` package with `__init__.py`, `manifest.py`, `gpu_resolver.py`, `cost_tracker.py`, `orchestrator.py`
    - Define `TaskType`, `TrainingFramework` enums in `manifest.py`
    - Define `GPUSpec` and `JobManifest` dataclasses with `to_json()` and `from_json()` methods
    - Define `GPU_HOURLY_RATES` dict and `GPU_FALLBACK_ORDER` list in `gpu_resolver.py`
    - Define `resolve_gpu_spec()` and `get_fallback_gpu()` functions
    - Define `CostTracker` dataclass with `start()`, `stop()`, `elapsed_seconds`, and `estimated_cost`
    - _Requirements: 1.1, 2.3, 7.1, 8.1, 8.2, 8.3_

  - [ ] 1.2 Write property test: Job Manifest round-trip serialization
    - **Property 10: Job Manifest Serialization Round-Trip**
    - **Validates: Requirements 8.1, 8.2, 8.3**

  - [ ] 1.3 Write property test: Manifest validation completeness
    - **Property 1: Manifest Validation Completeness**
    - **Validates: Requirements 1.1, 1.2**

  - [ ] 1.4 Write property test: GPU fallback ordering
    - **Property 3: GPU Fallback Ordering**
    - **Validates: Requirements 2.3**

  - [ ] 1.5 Write property test: Cost computation accuracy
    - **Property 8: Cost Computation Accuracy**
    - **Validates: Requirements 7.1**

  - [ ] 1.6 Write property test: Idle session detection
    - **Property 9: Idle Session Detection**
    - **Validates: Requirements 7.3**

- [ ] 2. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 3. Implement Studio Status Observation and Pipeline State
  - [ ] 3.1 Create `StudioStatusObservation` in `openhands/events/observation/studio.py`
    - Define dataclass extending `Observation` with `stage_name`, `stage_status`, `details`, `metrics`, `error`, `cost_estimate` fields
    - Add `ObservationType.STUDIO_STATUS` to the observation type enum
    - Register the new observation type in the event serialization system
    - _Requirements: 6.1, 6.2_

  - [ ] 3.2 Create `PipelineStage` and `PipelineState` models in `openhands/studio/orchestrator.py`
    - Define `StageStatus` enum, `PipelineStage` dataclass, `PipelineState` dataclass
    - Implement `apply_event(state, event)` function that updates pipeline state from a StudioStatusObservation
    - Implement `get_pipeline_summary()` method on PipelineState
    - _Requirements: 1.3, 6.1, 6.2_

  - [ ] 3.3 Write property test: Pipeline state consistency from events
    - **Property 7: Pipeline State Consistency From Events**
    - **Validates: Requirements 6.1, 6.2**

  - [ ] 3.4 Write property test: Pipeline summary contains all stages
    - **Property 2: Pipeline Summary Contains All Stages**
    - **Validates: Requirements 1.3**

- [ ] 4. Implement Pipeline Orchestrator core logic
  - [ ] 4.1 Implement `PipelineOrchestrator` class in `openhands/studio/orchestrator.py`
    - Constructor takes `JobManifest`, runtime reference, and `EventStream`
    - Implement `run()` method that sequences through pipeline stages
    - Implement `execute_stage()` that delegates to stage-specific handlers
    - Implement `emit_status()` that creates and emits `StudioStatusObservation` events
    - _Requirements: 1.3, 2.1, 2.2, 3.1, 3.2, 3.3, 4.1, 4.2, 4.3, 4.4, 5.1_

  - [ ] 4.2 Implement data preparation stage handler
    - Implement `_execute_data_prep()` that generates data scraping/processing commands
    - Implement dataset validation logic (file existence, non-zero size, format check)
    - _Requirements: 3.1, 3.2, 3.3, 3.4_

  - [ ] 4.3 Write property test: Dataset validation correctness
    - **Property 4: Dataset Validation Correctness**
    - **Validates: Requirements 3.3**

  - [ ] 4.4 Implement training stage handler
    - Implement `_execute_training()` that generates training config and executes training command
    - Implement training output metrics parser (extract loss, epoch, step from output lines)
    - Implement training polling loop that emits periodic status events with parsed metrics
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5_

  - [ ] 4.5 Write property test: Training metrics parsing
    - **Property 5: Training Metrics Parsing**
    - **Validates: Requirements 4.3**

  - [ ] 4.6 Implement deployment stage handler
    - Implement `_execute_deploy()` that deploys model as RunPod endpoint
    - Implement curl command generation from endpoint URL
    - Implement endpoint health check via test inference request
    - _Requirements: 5.1, 5.2, 5.3, 5.4_

  - [ ] 4.7 Write property test: Curl command generation
    - **Property 6: Curl Command Generation**
    - **Validates: Requirements 5.2**

- [ ] 5. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Integrate with RunPod Runtime and session management
  - [ ] 6.1 Wire Pipeline Orchestrator to RunPod Runtime
    - Implement provisioning stage that calls RunPod Runtime `create_pod()` with GPU_Spec parameters
    - Implement GPU fallback retry logic on provisioning failure
    - Implement GPU verification after pod reaches RUNNING status
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ] 6.2 Implement session lifecycle management
    - Implement idle timeout detection and pod stop logic
    - Wire CostTracker start/stop to session lifecycle
    - Implement session completion handler that emits final status event with duration and cost
    - Implement pod cleanup on session cancel/complete
    - Implement Job Manifest persistence to conversation storage on session completion
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 8.4_

  - [ ] 6.3 Write unit tests for RunPod integration
    - Test provisioning with mocked RunPod API
    - Test GPU fallback retry sequence
    - Test GPU verification success and failure paths
    - Test idle timeout detection
    - Test pod cleanup on session end
    - _Requirements: 2.1, 2.3, 2.4, 2.5, 7.2, 7.3_

- [ ] 7. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 8. Implement frontend Studio Mode components
  - [ ] 8.1 Create TypeScript types and API integration
    - Create `frontend/src/components/features/studio/studio-types.ts` with `StageStatus`, `PipelineStageInfo`, `StudioStatusEvent` types
    - Create `frontend/src/hooks/query/use-studio-session.ts` query hook
    - Create `frontend/src/hooks/mutation/use-confirm-studio-plan.ts` mutation hook
    - Wire WebSocket event handler to filter and dispatch `studio_status` observation events
    - _Requirements: 6.1, 6.2_

  - [ ] 8.2 Create Studio Pipeline Panel components
    - Create `StudioPipelinePanel.tsx` — main panel rendering list of stage cards
    - Create `StudioStageCard.tsx` — individual stage with status indicator, details, error display
    - Create `StudioTrainingMetrics.tsx` — live metrics display (loss, epoch, step)
    - Create `StudioSummaryPanel.tsx` — final summary with endpoint URL, curl command, cost estimate
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [ ] 8.3 Write frontend component tests
    - Test StudioPipelinePanel renders correct number of stages
    - Test StudioStageCard displays correct status and error states
    - Test StudioTrainingMetrics displays metric values
    - Test StudioSummaryPanel displays endpoint URL, curl command, and cost
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

- [ ] 9. Wire Studio Mode into agent conversation flow
  - [ ] 9.1 Create Studio Mode microagent
    - Create a microagent prompt in `microagents/` with triggers for AI workload keywords (fine-tune, train, RAG, deploy model)
    - The microagent instructs the agent to invoke the Pipeline Orchestrator when Studio Mode tasks are detected
    - _Requirements: 1.1_

  - [ ] 9.2 Integrate Pipeline Orchestrator into agent action handling
    - Add Studio Mode entry point in the agent's action handling flow
    - Wire user confirmation flow for Job Manifest review (present summary, handle accept/reject/modify)
    - Wire deployment confirmation flow
    - _Requirements: 1.3, 1.4, 5.1_

- [ ] 10. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties (Hypothesis library, 100+ iterations each)
- Unit tests validate specific examples and edge cases
- The RunPod Runtime itself is assumed to exist per `docs/runpod-integration-design.md`; this plan builds the Studio orchestration layer on top of it
