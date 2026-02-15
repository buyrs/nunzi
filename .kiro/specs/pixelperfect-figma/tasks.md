# Implementation Plan: PixelPerfect (Figma-to-Code)

## Overview

Implement the PixelPerfect Figma-to-Code agent as a new `openhands/pixelperfect/` module following the same patterns as `openhands/resolver/`. Tasks are ordered to build core data models first, then Figma connectivity, then design parsing and token extraction, then component mapping, then code generation, then change detection, then PR creation and orchestration.

## Tasks

- [ ] 1. Set up module structure and data models
  - [ ] 1.1 Create `openhands/pixelperfect/__init__.py` and `openhands/pixelperfect/models.py`
    - Define all enums: `NodeType`, `TokenType`, `ChangeType`
    - Define all Pydantic models: `AutoLayoutProps`, `StyleProps`, `ConstraintProps`, `DesignNode`, `DesignTree`, `DesignToken`, `TokenSet`, `PropSpec`, `CSSProperty`, `ComponentSpec`, `ComponentSpecSet`, `GeneratedFile`, `GeneratedFileSet`, `NodeChange`, `ChangeSet`, `PixelPerfectOutput`
    - Implement JSON serialization/deserialization using Pydantic's `model_dump_json()` / `model_validate_json()`
    - _Requirements: 2.5, 2.6, 3.5, 3.6, 4.7, 4.8, 5.9, 5.10, 7.3, 7.4, 10.2, 10.4_

  - [ ] 1.2 Write property tests for round-trip serialization of all data models
    - **Property 1: Design_Tree round-trip serialization**
    - **Property 2: Token_Set round-trip serialization**
    - **Property 4: Component_Spec_Set round-trip serialization**
    - **Property 5: Generated_File_Set round-trip serialization**
    - **Property 6: Change_Set round-trip serialization**
    - **Property 7: PixelPerfect_Output round-trip serialization**
    - **Validates: Requirements 2.5, 2.6, 3.5, 3.6, 4.7, 4.8, 5.9, 5.10, 7.3, 7.4, 10.4**

- [ ] 2. Implement configuration loading
  - [ ] 2.1 Create `openhands/pixelperfect/config.py`
    - Implement `FigmaFileMapping` and `PixelPerfectConfig` dataclasses with all fields: `figma_token`, `files`, `framework`, `styling`, `typescript`, `token_format`, `component_prefix`, `ignore_nodes`, `snapshots_enabled`
    - Implement `PixelPerfectConfig.load(repo_dir)` that reads `.pixelperfect.yml`, validates fields, falls back to defaults on missing/invalid file
    - Implement `to_dict()` and `from_dict()` methods
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10_

  - [ ] 2.2 Write property test for config parsing with defaults
    - **Property 25: Config parsing with defaults**
    - **Validates: Requirements 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9, 8.10**

  - [ ] 2.3 Write unit tests for config loading edge cases
    - Test missing file (defaults), invalid YAML (fallback), partial config
    - _Requirements: 8.9, 8.10_

- [ ] 3. Implement Figma connector
  - [ ] 3.1 Create `openhands/pixelperfect/figma_connector.py`
    - Implement `FigmaUrlParts` dataclass and `FigmaConnector` class
    - Implement `parse_url()` supporting `/file/`, `/design/`, and `?node-id=` URL formats
    - Implement `get_file()` that calls the Figma REST API with authentication headers
    - Implement `_request_with_retry()` with exponential backoff (3 retries, base 1s delay) on 5xx errors
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6_

  - [ ] 3.2 Write property test for Figma URL parsing
    - **Property 8: Figma URL parsing extracts correct parts**
    - **Validates: Requirements 1.1**

  - [ ] 3.3 Write unit tests for Figma connector
    - Test authentication header setting, retry logic with mocked failures, auth error handling, node-id subtree retrieval
    - _Requirements: 1.2, 1.3, 1.4, 1.5, 1.6_

- [ ] 4. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. Implement design tree parser
  - [ ] 5.1 Create `openhands/pixelperfect/design_tree.py`
    - Implement `DesignTreeParser` class with `parse()`, `_parse_node()`, and `_resolve_instances()` methods
    - Parse raw Figma API JSON into `DesignTree` with `DesignNode` objects preserving auto-layout, style, and constraint properties
    - Resolve INSTANCE node references to main component definitions
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.7_

  - [ ] 5.2 Write property test for design tree parsing completeness
    - **Property 9: Design_Tree parsing preserves all node properties**
    - **Validates: Requirements 2.1, 2.2, 2.3, 2.4**

  - [ ] 5.3 Write property test for component instance resolution
    - **Property 10: Component instance references are resolved**
    - **Validates: Requirements 2.7**

- [ ] 6. Implement token extractor
  - [ ] 6.1 Create `openhands/pixelperfect/token_extractor.py`
    - Implement `TokenExtractor` class with `extract()`, `_extract_colors()`, `_extract_typography()`, `_extract_spacing()`, `_extract_effects()` methods
    - Extract color tokens from fill/stroke styles, typography tokens from text styles, spacing tokens from auto-layout values, shadow/blur tokens from effect styles
    - _Requirements: 3.1, 3.2, 3.3, 3.4_

  - [ ] 6.2 Write property test for token extraction coverage
    - **Property 11: Token extraction covers all style types**
    - **Validates: Requirements 3.1, 3.2, 3.3, 3.4**

- [ ] 7. Implement token file generator
  - [ ] 7.1 Create `openhands/pixelperfect/token_generator.py`
    - Implement `TokenFileGenerator` class with `generate()`, `generate_css()`, `generate_tailwind()`, `generate_typescript()` methods
    - Implement `parse_css()`, `parse_tailwind()`, `parse_typescript()` methods for round-trip support
    - Generate CSS custom properties, Tailwind config extensions, or TypeScript constants from TokenSet
    - _Requirements: 3.7, 3.8, 3.9_

  - [ ] 7.2 Write property test for token file round-trip
    - **Property 3: Token file round-trip**
    - **Validates: Requirements 3.7, 3.8, 3.9**

  - [ ] 7.3 Write unit tests for token file generation
    - Test each format (CSS, Tailwind, TypeScript) with specific token examples
    - _Requirements: 3.7_

- [ ] 8. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. Implement component mapper
  - [ ] 9.1 Create `openhands/pixelperfect/component_mapper.py`
    - Implement `ComponentMapper` class with `map()`, `_map_node()`, `_map_auto_layout_to_css()`, `_map_style_to_css()`, `_map_constraints_to_css()`, `_map_variants_to_props()`, `_infer_html_tag()` methods
    - Map Figma components to ComponentSpec with correct hierarchy, props from variants, and CSS from auto-layout/style/constraints
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

  - [ ] 9.2 Write property test for component mapping hierarchy and props
    - **Property 12: Component mapping produces correct hierarchy and props**
    - **Validates: Requirements 4.1, 4.2, 4.3**

  - [ ] 9.3 Write property test for auto-layout to CSS mapping
    - **Property 13: Auto-layout maps to correct CSS flexbox properties**
    - **Validates: Requirements 4.4**

  - [ ] 9.4 Write property test for text node CSS mapping
    - **Property 14: Text nodes map to CSS typography properties**
    - **Validates: Requirements 4.5**

  - [ ] 9.5 Write property test for constraint CSS mapping
    - **Property 15: Constraint nodes map to CSS sizing properties**
    - **Validates: Requirements 4.6**

- [ ] 10. Implement code generator
  - [ ] 10.1 Create `openhands/pixelperfect/code_generator.py`
    - Implement `CodeGenerator` class with `generate()`, `generate_barrel_export()`, `generate_token_file()` methods
    - Use OpenHands AgentController to generate React/TypeScript components from ComponentSpec
    - Support Tailwind and CSS Modules styling approaches
    - Generate barrel export file re-exporting all components
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8_

  - [ ] 10.2 Write property test for generated code structure
    - **Property 16: Generated code contains required structure**
    - **Validates: Requirements 5.1**

  - [ ] 10.3 Write property test for Tailwind class generation
    - **Property 17: Tailwind styling generates utility classes**
    - **Validates: Requirements 5.2**

  - [ ] 10.4 Write property test for CSS Modules file generation
    - **Property 18: CSS Modules generates companion file**
    - **Validates: Requirements 5.3**

  - [ ] 10.5 Write property test for barrel export completeness
    - **Property 19: Barrel export contains all components**
    - **Validates: Requirements 5.6**

  - [ ] 10.6 Write unit tests for code generator
    - Test specific component generation examples (button, card, form)
    - Test interactive state generation (hover, focus)
    - _Requirements: 5.4, 5.8_

- [ ] 11. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 12. Implement change detector
  - [ ] 12.1 Create `openhands/pixelperfect/change_detector.py`
    - Implement `ChangeDetector` class with `compare()`, `_diff_nodes()`, `_diff_properties()` methods
    - Build flat node maps from both trees, identify ADDED/REMOVED/MODIFIED nodes
    - For MODIFIED nodes, list the specific properties that changed
    - _Requirements: 7.1, 7.2_

  - [ ] 12.2 Write property test for change detection correctness
    - **Property 20: Change detection correctly categorizes differences**
    - **Validates: Requirements 7.1, 7.2**

  - [ ] 12.3 Write property test for identical tree comparison
    - **Property 21: Identical trees produce empty Change_Set**
    - **Validates: Requirements 7.7**

- [ ] 13. Implement snapshot generator
  - [ ] 13.1 Create `openhands/pixelperfect/snapshot_generator.py`
    - Implement `SnapshotGenerator` class with `generate_stories()`, `_generate_story()`, `_generate_variant_stories()` methods
    - Generate Storybook story files for each component with variant stories
    - Respect `snapshots_enabled` config flag
    - _Requirements: 9.1, 9.2, 9.3, 9.4_

  - [ ] 13.2 Write property test for story file coverage
    - **Property 24: Story files cover all components and variants**
    - **Validates: Requirements 9.1, 9.2**

- [ ] 14. Implement PR creator
  - [ ] 14.1 Create `openhands/pixelperfect/pr_creator.py`
    - Implement `PRCreator` class with `create_pr()`, `create_sync_pr()`, `generate_branch_name()`, `generate_pr_title()`, `generate_pr_body()` methods
    - Use existing OpenHands GitHub infrastructure (`send_pull_request.py` patterns)
    - Generate branch names as `pixelperfect/{file_key}/{timestamp}`
    - Generate PR titles with file name and component count
    - Generate PR bodies with component list, token count, Figma link, and change summary for sync PRs
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 7.5, 7.6_

  - [ ] 14.2 Write property test for PR content
    - **Property 22: PR content contains required information**
    - **Validates: Requirements 6.3, 6.4**

  - [ ] 14.3 Write property test for sync PR body
    - **Property 23: Sync PR body describes changes**
    - **Validates: Requirements 7.6**

  - [ ] 14.4 Write unit tests for PR creator
    - Test branch naming, PR creation failure handling, empty change set behavior
    - _Requirements: 6.1, 6.6, 7.7_

- [ ] 15. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 16. Wire components together in the PixelPerfect Agent entry point
  - [ ] 16.1 Create `openhands/pixelperfect/run.py`
    - Implement `PixelPerfectAgent` class that orchestrates the full pipeline: config loading → Figma connection → design tree parsing → token extraction → component mapping → code generation → snapshot generation → PR creation
    - Implement `sync()` method for change detection flow: Figma connection → design tree parsing → change detection → selective regeneration → follow-up PR
    - Implement CLI argument parsing analogous to `openhands/resolver/resolve_issue.py`
    - Write `PixelPerfectOutput` to the output directory as `pixelperfect_output.json`
    - Handle all error paths: continue on component failures, log errors, produce output on partial failure
    - _Requirements: 1.1, 1.2, 1.5, 7.5, 7.7, 10.1, 10.2, 10.3_

  - [ ] 16.2 Write unit tests for PixelPerfectAgent orchestration
    - Test generate flow with mocked components
    - Test sync flow with mocked components (changes found, no changes)
    - Test error handling: component failure continues, API failure logged
    - _Requirements: 10.1, 10.2, 10.3_

- [ ] 17. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties using `hypothesis`
- Unit tests validate specific examples and edge cases using `pytest`
- The implementation reuses existing OpenHands infrastructure (`Runtime`, `AgentController`, GitHub PR utils) rather than reimplementing
- Test files go in `tests/unit/test_pixelperfect/` following the existing test organization pattern
