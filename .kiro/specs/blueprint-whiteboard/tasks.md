# Implementation Plan: Blueprint Whiteboard-to-Code

## Overview

Implementation proceeds backend-first (analyzer and API), then frontend (Excalidraw integration, tab, controls), then wiring the generation flow. Property tests are placed close to the code they validate. The Excalidraw library is added as an npm dependency. Hypothesis is used for backend property tests, fast-check for frontend.

## Tasks

- [ ] 1. Implement Blueprint Analyzer core logic
  - [ ] 1.1 Create `openhands/server/services/blueprint_analyzer.py` with data classes (`NodeDescription`, `EdgeDescription`, `BlueprintAnalysis`) and the `analyze_blueprint` function
    - Parse Excalidraw element arrays: extract shape elements as nodes, arrow elements with bindings as edges
    - Resolve text labels by finding text elements bound to shapes via `containerId`/`boundElements`
    - Assign default names (`Shape_N`) to unlabeled elements
    - Generate warnings for disconnected arrows (null `startBinding` or `endBinding`)
    - _Requirements: 3.1, 3.3, 3.4, 3.5_

  - [ ] 1.2 Implement annotation parser in `blueprint_analyzer.py`
    - Parse strings like `"Auth Service (FastAPI)"` into name and technology
    - Handle strings without parentheses (technology = None)
    - _Requirements: 3.2_

  - [ ] 1.3 Implement `serialize_analysis` and `deserialize_analysis` functions
    - JSON serialization/deserialization for `BlueprintAnalysis`
    - _Requirements: 3.6_

  - [ ] 1.4 Implement `build_scaffold_prompt` function
    - Construct the prompt string with serialized JSON, image URL placeholder, system instructions, and warnings
    - _Requirements: 4.2, 4.4_

  - [ ] 1.5 Write property tests for Blueprint Analyzer (`tests/unit/test_blueprint_analyzer.py`)
    - **Property 3: Analyzer extracts nodes and edges from elements**
    - **Validates: Requirements 3.1, 3.3, 3.4**
    - **Property 4: Annotation parsing extracts name and technology**
    - **Validates: Requirements 3.2**
    - **Property 5: Disconnected arrows produce warnings**
    - **Validates: Requirements 3.5**
    - **Property 6: Analysis serialization round trip**
    - **Validates: Requirements 3.6**
    - **Property 7: Scaffold prompt contains all required parts**
    - **Validates: Requirements 4.2, 4.4**

  - [ ] 1.6 Write unit tests for Blueprint Analyzer edge cases
    - Test specific diagram: "Auth Service (FastAPI)" rectangle → "Users DB (Postgres)" cylinder
    - Test annotation edge cases: empty string, multiple parentheses, no parentheses
    - Test empty element array produces empty analysis
    - _Requirements: 3.1, 3.2, 3.4, 3.5_

- [ ] 2. Checkpoint - Ensure analyzer tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 3. Implement Blueprint API endpoints
  - [ ] 3.1 Create `openhands/server/routes/blueprint.py` with FastAPI router
    - PUT `/api/conversations/{conversation_id}/blueprint` — validate and store element JSON + PNG via FileStore
    - GET `/api/conversations/{conversation_id}/blueprint` — retrieve stored blueprint data
    - DELETE `/api/conversations/{conversation_id}/blueprint` — remove stored blueprint
    - POST `/api/conversations/{conversation_id}/blueprint/generate` — run analyzer, build scaffold prompt, dispatch MessageAction
    - _Requirements: 5.1, 5.2, 5.3, 4.1_

  - [ ] 3.2 Register blueprint router in `openhands/server/app.py`
    - Import and include the blueprint router following existing patterns
    - _Requirements: 5.1_

  - [ ] 3.3 Add request/response models (`BlueprintSaveRequest`, `BlueprintResponse`) with Pydantic validation
    - Validate element JSON structure, reject malformed input with 422
    - Return 404 for invalid conversation IDs
    - _Requirements: 5.4, 5.5_

  - [ ] 3.4 Write property test for blueprint persistence round trip (`tests/unit/test_blueprint_routes.py`)
    - **Property 2: Blueprint persistence round trip**
    - **Validates: Requirements 2.2, 2.4**
    - Use in-memory FileStore for testing

  - [ ] 3.5 Write unit tests for Blueprint API endpoints
    - Test CRUD cycle (PUT, GET, DELETE)
    - Test 404 for nonexistent conversation
    - Test 422 for malformed element JSON
    - Test GET returns empty response for conversation with no blueprint
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 2.5_

- [ ] 4. Checkpoint - Ensure backend tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. Set up frontend Excalidraw integration
  - [ ] 5.1 Install `@excalidraw/excalidraw` npm package and add whiteboard icon SVG
    - Add dependency to `frontend/package.json`
    - Create `frontend/src/icons/whiteboard.svg`
    - _Requirements: 1.1_

  - [ ] 5.2 Create `BlueprintService` API client (`frontend/src/api/blueprint-service.ts`)
    - Implement `saveBlueprint`, `getBlueprint`, `deleteBlueprint`, `generateCode` static methods
    - Follow existing API service patterns (use `openHands.axios`)
    - _Requirements: 5.1, 5.2, 5.3_

  - [ ] 5.3 Create TanStack Query hooks
    - `frontend/src/hooks/query/use-blueprint.ts` — query hook for loading blueprint
    - `frontend/src/hooks/mutation/use-save-blueprint.ts` — mutation for saving
    - `frontend/src/hooks/mutation/use-delete-blueprint.ts` — mutation for deleting
    - `frontend/src/hooks/mutation/use-generate-code.ts` — mutation for code generation
    - _Requirements: 2.1, 2.2, 6.3_

- [ ] 6. Implement WhiteboardTab component
  - [ ] 6.1 Create `frontend/src/components/features/whiteboard/whiteboard-tab.tsx`
    - Embed Excalidraw component with onChange handler
    - Load saved blueprint on mount via `useBlueprint` hook
    - Implement 2-second debounced auto-save using `useSaveBlueprint`
    - Show placeholder text when canvas is empty
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.1_

  - [ ] 6.2 Create `frontend/src/components/features/whiteboard/blueprint-controls.tsx`
    - "Generate Code" button: enabled when elements > 0, shows loading during generation
    - "Clear Canvas" button: clears elements and calls delete mutation
    - _Requirements: 6.1, 6.2, 6.3, 6.5_

  - [ ] 6.3 Implement generate code flow in WhiteboardTab
    - Export canvas as PNG via Excalidraw API
    - Call `useGenerateCode` mutation
    - On success, switch to chat tab via `useSelectConversationTab`
    - _Requirements: 4.1, 6.4_

  - [ ] 6.4 Write property tests for frontend whiteboard logic (`frontend/src/components/features/whiteboard/__tests__/whiteboard.test.ts`)
    - **Property 1: Debounce coalesces rapid saves**
    - **Validates: Requirements 2.1**
    - **Property 8: Generate button enabled iff canvas has elements**
    - **Validates: Requirements 6.1, 6.2**
    - **Property 9: Tab switching preserves whiteboard state**
    - **Validates: Requirements 7.2, 7.3**

  - [ ] 6.5 Write unit tests for WhiteboardTab and BlueprintControls
    - Test placeholder shown when empty
    - Test loading state during generation
    - Test clear canvas resets elements
    - Test error toast on save failure
    - _Requirements: 1.3, 2.3, 6.3, 6.5_

- [ ] 7. Wire whiteboard tab into conversation UI
  - [ ] 7.1 Add "whiteboard" tab to `ConversationTabs` (`frontend/src/components/features/conversation/conversation-tabs/conversation-tabs.tsx`)
    - Add tab entry with whiteboard icon
    - _Requirements: 7.1_

  - [ ] 7.2 Render `WhiteboardTab` in `ConversationTabContent` when "whiteboard" tab is selected
    - Add case for "whiteboard" in the tab content renderer
    - _Requirements: 7.1, 7.2, 7.3_

  - [ ] 7.3 Add i18n translation keys for whiteboard UI strings
    - Add `COMMON$WHITEBOARD`, `BLUEPRINT$GENERATE_CODE`, `BLUEPRINT$CLEAR_CANVAS`, `BLUEPRINT$PLACEHOLDER`, `BLUEPRINT$SAVING`, `BLUEPRINT$GENERATION_IN_PROGRESS` to translation files
    - _Requirements: 1.3, 6.1, 6.3_

- [ ] 8. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.
  - Run `pre-commit run --config ./dev_config/python/.pre-commit-config.yaml` for backend lint
  - Run `cd frontend && npm run lint:fix && npm run build` for frontend lint and build

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties using Hypothesis (Python) and fast-check (TypeScript)
- Unit tests validate specific examples and edge cases
- The Excalidraw library handles all drawing primitives — no custom canvas code needed
