# Implementation Plan: Nexus Smart Routing

## Overview

Implement the Nexus smart routing layer as a new RouterLLM subclass within the existing OpenHands LLM router infrastructure. The implementation follows an incremental approach: data models first, then classifier logic, then router integration, then feedback tracking.

## Tasks

- [ ] 1. Create data models and enums
  - [ ] 1.1 Create `openhands/llm/router/nexus/models.py` with ComplexityTier enum, ClassificationResult, ProjectContext, TierConfigMapping, RoutingDecision, and FeedbackRecord Pydantic models
    - ComplexityTier should be a str Enum with values: quick_fix, standard, complex, epic
    - ComplexityTier should support ordering for tier bump logic (define __lt__ based on tier order)
    - ClassificationResult: tier (ComplexityTier), confidence (float, 0.0-1.0), explanation (str)
    - ProjectContext: language (str|None), framework (str|None), estimated_lines (int|None), is_large_codebase property (>100k lines)
    - TierConfigMapping: quick_fix, standard, complex, epic fields (str LLM key names), get_llm_key(tier) method
    - RoutingDecision: task_id (str), tier, confidence, selected_model (str), explanation (str)
    - FeedbackRecord: task_id (str), routing_decision (RoutingDecision), actual_cost (float), actual_duration_seconds (float), success (bool), timestamp (datetime)
    - _Requirements: 1.1, 1.2, 2.1, 2.6, 4.1, 4.2, 4.3, 5.1, 5.4_

  - [ ] 1.2 Write property tests for data model serialization round-trips in `tests/unit/test_nexus_models.py`
    - **Property 6: RoutingDecision round-trip serialization**
    - **Validates: Requirements 4.2, 4.3**
    - **Property 7: TierConfigMapping round-trip serialization**
    - **Validates: Requirements 2.6**
    - **Property 8: FeedbackRecord round-trip serialization**
    - **Validates: Requirements 5.4**

  - [ ] 1.3 Write property tests for data model completeness invariants in `tests/unit/test_nexus_models.py`
    - **Property 5: RoutingDecision completeness**
    - **Validates: Requirements 4.1**
    - **Property 9: FeedbackRecord completeness**
    - **Validates: Requirements 5.1**

- [ ] 2. Implement TaskClassifier
  - [ ] 2.1 Create `openhands/llm/router/nexus/classifier.py` with TaskClassifier class
    - Define QUICK_FIX_KEYWORDS, EPIC_KEYWORDS, MULTI_FILE_PATTERNS as class-level lists
    - Implement classify(task_text, project_context=None) -> ClassificationResult
    - Implement _keyword_score(task_text) -> dict[ComplexityTier, float] using keyword matching
    - Implement _apply_context_adjustment(result, context) -> ClassificationResult for large codebase tier bumping
    - Default to Standard tier when confidence < 0.5
    - Wrap classification in try/except, fall back to Standard on error
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 3.2, 3.3, 7.1_

  - [ ] 2.2 Write property tests for classifier in `tests/unit/test_nexus_classifier.py`
    - **Property 1: Classifier output invariant**
    - **Validates: Requirements 1.1, 1.2**
    - **Property 2: Keyword classification correctness**
    - **Validates: Requirements 1.3, 1.4, 1.5**

  - [ ] 2.3 Write property test for tier bump logic in `tests/unit/test_nexus_classifier.py`
    - **Property 4: Large codebase tier bump**
    - **Validates: Requirements 3.2**

  - [ ] 2.4 Write unit tests for classifier edge cases in `tests/unit/test_nexus_classifier.py`
    - Test empty string input defaults to Standard
    - Test very long string input completes without error
    - Test unicode input is handled gracefully
    - Test classifier error fallback to Standard tier
    - _Requirements: 1.6, 7.1_

- [ ] 3. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 4. Implement NexusRouter
  - [ ] 4.1 Create `openhands/llm/router/nexus/__init__.py` that exports NexusRouter
    - _Requirements: 6.1_

  - [ ] 4.2 Create `openhands/llm/router/nexus/router.py` with NexusRouter class extending RouterLLM
    - Initialize TaskClassifier and TierConfigMapping in __init__
    - Implement _select_llm(messages) that extracts task text from first user message, classifies it, maps tier to LLM key
    - Implement _extract_task_text(messages) to get user task from message list
    - Fall back to "primary" if selected LLM key not in available_llms
    - Store current RoutingDecision for logging/feedback
    - Log routing decisions with task_id, tier, confidence, model name
    - Register NexusRouter in ROUTER_LLM_REGISTRY with name "nexus_router"
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 4.4, 7.2_

  - [ ] 4.3 Write property test for _select_llm in `tests/unit/test_nexus_router.py`
    - **Property 10: _select_llm returns correct LLM key**
    - **Validates: Requirements 6.2**

  - [ ] 4.4 Write property test for tier-to-config mapping in `tests/unit/test_nexus_router.py`
    - **Property 3: Tier-to-config mapping correctness**
    - **Validates: Requirements 2.2, 2.3, 2.4, 2.5**

  - [ ] 4.5 Write unit tests for NexusRouter in `tests/unit/test_nexus_router.py`
    - Test NexusRouter is registered in ROUTER_LLM_REGISTRY
    - Test fallback to primary when LLM key not available
    - Test noop behavior when no Nexus config provided
    - Test _extract_task_text with various message formats
    - _Requirements: 6.1, 6.4, 7.2_

- [ ] 5. Implement FeedbackTracker
  - [ ] 5.1 Create `openhands/llm/router/nexus/feedback.py` with FeedbackTracker class
    - Implement record(feedback: FeedbackRecord) with try/except for storage failures
    - Implement get_history(limit=100) to retrieve recent records
    - Use JSON file-based storage (one record per line, append-only)
    - Log and continue on storage errors
    - _Requirements: 5.1, 5.2, 7.3_

  - [ ] 5.2 Write unit tests for FeedbackTracker in `tests/unit/test_nexus_feedback.py`
    - Test record persists to storage
    - Test get_history retrieves records
    - Test storage failure is handled gracefully (continues without error)
    - _Requirements: 5.1, 5.2, 7.3_

- [ ] 6. Wire components together
  - [ ] 6.1 Update `openhands/llm/router/__init__.py` to import NexusRouter so it registers on module load
    - _Requirements: 6.1_

  - [ ] 6.2 Integrate FeedbackTracker into NexusRouter
    - Add feedback_tracker to NexusRouter.__init__
    - Wire NexusRouter to create RoutingDecision on each _select_llm call
    - _Requirements: 5.1, 5.3_

  - [ ] 6.3 Write integration tests in `tests/unit/test_nexus_router.py`
    - Test end-to-end: submit task text → classify → select LLM → produce RoutingDecision
    - Test with mocked LLMRegistry and available_llms
    - _Requirements: 6.2, 4.1_

- [ ] 7. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests use Hypothesis library with minimum 100 iterations
- Unit tests use pytest
- All new code goes under `openhands/llm/router/nexus/` to keep it self-contained
- NexusRouter extends the existing RouterLLM base class for seamless integration
